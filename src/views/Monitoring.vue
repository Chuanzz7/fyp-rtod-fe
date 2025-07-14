<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from "vue";
import Card from "primevue/card";
import Button from "primevue/button";
import DataTable from "primevue/datatable";
import Column from "primevue/column";
import InputText from "primevue/inputtext";
import InputNumber from "primevue/inputnumber";
import Dropdown from "primevue/dropdown";
import Dialog from "primevue/dialog";
import Message from "primevue/message";
import Toast from "primevue/toast";
import { useToast } from "primevue/usetoast";
import axios from "axios";

const toast = useToast();
const BASE_URL = import.meta.env.VITE_API_BASE_PROCESSOR ?? "";
const STREAM_URL = `${BASE_URL}/video_feed`;

// Region Management Data
const regions = ref([]);
const loading = ref(false);
const selectedRegion = ref(null);
const showDialog = ref(false);
const dialogMode = ref("create"); // 'create' or 'edit'
const error = ref("");

// Live Stream Data
const imgRef = ref(null);
const isStreaming = ref(false);
const streamLoading = ref(false);
let retryInterval = null;

// Form data
const regionForm = ref({
    label: "",
    bbox: {
        x1: 0,
        y1: 0,
        x2: 0,
        y2: 0
    },
});

// Computed
const dialogTitle = computed(() => {
    return dialogMode.value === "create" ? "Create New Region" : "Edit Region";
});

const isFormValid = computed(() => {
    return regionForm.value.label.trim() !== "" &&
        regionForm.value.bbox.x1 < regionForm.value.bbox.x2 &&
        regionForm.value.bbox.y1 < regionForm.value.bbox.y2;
});

// Region Management Methods
async function fetchRegions() {
    loading.value = true;
    error.value = "";
    try {
        const response = await axios.get(`${BASE_URL}/api/regions`);
        regions.value = response.data.regions.map((region, index) => ({
            ...region,
            index,
            bbox: {
                x1: region.bbox[0],
                y1: region.bbox[1],
                x2: region.bbox[2],
                y2: region.bbox[3]
            }
        }));
    } catch (e) {
        error.value = "Failed to fetch regions";
        console.error("Error fetching regions:", e);
        toast.add({ severity: "error", summary: "Error", detail: "Failed to fetch regions", life: 3000 });
    } finally {
        loading.value = false;
    }
}

function openCreateDialog() {
    dialogMode.value = "create";
    regionForm.value = {
        label: "",
        bbox: {
            x1: 0,
            y1: 0,
            x2: 100,
            y2: 100
        },
    };
    showDialog.value = true;
}

function openEditDialog(region) {
    dialogMode.value = "edit";
    selectedRegion.value = region;
    regionForm.value = {
        label: region.label,
        bbox: { ...region.bbox },
    };
    showDialog.value = true;
}

function closeDialog() {
    showDialog.value = false;
    selectedRegion.value = null;
    regionForm.value = {
        label: "",
        bbox: {
            x1: 0,
            y1: 0,
            x2: 0,
            y2: 0
        },
    };
}

async function saveRegion() {
    if (!isFormValid.value) {
        toast.add({
            severity: "warn",
            summary: "Validation Error",
            detail: "Please fill all required fields correctly",
            life: 3000
        });
        return;
    }

    loading.value = true;
    try {
        if (dialogMode.value === "create") {
            // Create new region by updating all regions
            const newRegions = [...regions.value, {
                label: regionForm.value.label,
                bbox: regionForm.value.bbox,
            }];

            const updateData = {
                regions: newRegions.map(region => ({
                    label: region.label,
                    bbox: {
                        x1: region.bbox.x1,
                        y1: region.bbox.y1,
                        x2: region.bbox.x2,
                        y2: region.bbox.y2
                    },
                }))
            };

            await axios.post(`${BASE_URL}/api/regions`, updateData);
            toast.add({ severity: "success", summary: "Success", detail: "Region created successfully", life: 3000 });
        } else {
            // Update existing region
            const updateData = {
                label: regionForm.value.label,
                bbox: {
                    x1: regionForm.value.bbox.x1,
                    y1: regionForm.value.bbox.y1,
                    x2: regionForm.value.bbox.x2,
                    y2: regionForm.value.bbox.y2
                },
            };

            await axios.post(`${BASE_URL}/api/regions/single?region_index=${selectedRegion.value.index}`, updateData);
            toast.add({ severity: "success", summary: "Success", detail: "Region updated successfully", life: 3000 });
        }

        closeDialog();
        await fetchRegions();
    } catch (e) {
        console.error("Error saving region:", e);
        toast.add({ severity: "error", summary: "Error", detail: "Failed to save region", life: 3000 });
    } finally {
        loading.value = false;
    }
}

async function deleteRegion(region) {
    if (!confirm(`Are you sure you want to delete region "${region.label}"?`)) {
        return;
    }

    loading.value = true;
    try {
        await axios.delete(`${BASE_URL}/api/regions/${region.index}`);
        toast.add({ severity: "success", summary: "Success", detail: "Region deleted successfully", life: 3000 });
        await fetchRegions();
    } catch (e) {
        console.error("Error deleting region:", e);
        toast.add({ severity: "error", summary: "Error", detail: "Failed to delete region", life: 3000 });
    } finally {
        loading.value = false;
    }
}

function formatBbox(bbox) {
    return `(${bbox.x1}, ${bbox.y1}) - (${bbox.x2}, ${bbox.y2})`;
}

// Live Stream Methods
async function startStream() {
    if (isStreaming.value || streamLoading.value) return;
    streamLoading.value = true;
    try {
        await axios.post(`${BASE_URL}/start`);
        setTimeout(() => {
            if (imgRef.value) {
                imgRef.value.src = `${STREAM_URL}?t=${Date.now()}`;
                isStreaming.value = true;
                streamLoading.value = false;
                startRetry();
            }
        }, 200);
    } catch (e) {
        console.error("Failed to start stream:", e);
        streamLoading.value = false;
    }
}

async function stopStream() {
    if (!isStreaming.value) return;
    clearInterval(retryInterval);
    isStreaming.value = false;
    try {
        await axios.post(`${BASE_URL}/stop`);
    } catch (e) {
        console.error("Failed to stop stream:", e);
    } finally {
        isStreaming.value = false;
        if (imgRef.value) imgRef.value.src = "";
        streamLoading.value = false;
    }
}

function toggleStream() {
    if (isStreaming.value) {
        stopStream();
    } else {
        startStream();
    }
}

function startRetry() {
    clearInterval(retryInterval);
    retryInterval = setInterval(() => {
        if (imgRef.value && isStreaming.value) {
            imgRef.value.src = `${STREAM_URL}?t=${Date.now()}`;
        }
    }, 60000);
}

// Lifecycle
onMounted(() => {
    fetchRegions();
});

onBeforeUnmount(() => {
    clearInterval(retryInterval);
});
</script>

<template>
    <div class="grid">
        <!-- Region Management Section -->
        <div class="col-12">
            <Card>
                <template #header>
                    <div class="flex align-items-center justify-between w-full p-5">
                        <h2 class="flex m-0 text-xl font-semibold">Region Management</h2>
                        <div class="flex gap-2">
                            <Button
                                label="Refresh"
                                icon="pi pi-refresh"
                                :loading="loading"
                                @click="fetchRegions"
                                outlined
                            />
                            <Button
                                label="Add Region"
                                icon="pi pi-plus"
                                @click="openCreateDialog"
                                :disabled="loading"
                            />
                        </div>
                    </div>
                </template>

                <template #content>
                    <div class="w-full px-5">
                        <Message v-if="error" severity="error" :closable="false" class="mb-4">
                            {{ error }}
                        </Message>

                        <DataTable
                            :value="regions"
                            :loading="loading"
                            dataKey="index"
                            showGridlines
                            stripedRows
                            responsiveLayout="scroll"
                        >
                            <Column field="label" header="Label" sortable>
                                <template #body="slotProps">
                                    <span class="font-semibold">{{ slotProps.data.label }}</span>
                                </template>
                            </Column>

                            <Column field="bbox" header="Bounding Box">
                                <template #body="slotProps">
                                    <code class="text-sm bg-gray-100 px-2 py-1 border-round">
                                        {{ formatBbox(slotProps.data.bbox) }}
                                    </code>
                                </template>
                            </Column>

                            <Column header="Actions" :exportable="false">
                                <template #body="slotProps">
                                    <div class="flex gap-2">
                                        <Button
                                            icon="pi pi-pencil"
                                            size="small"
                                            outlined
                                            @click="openEditDialog(slotProps.data)"
                                            :disabled="loading"
                                        />
                                        <Button
                                            icon="pi pi-trash"
                                            size="small"
                                            severity="danger"
                                            outlined
                                            @click="deleteRegion(slotProps.data)"
                                            :disabled="loading"
                                        />
                                    </div>
                                </template>
                            </Column>
                        </DataTable>

                        <div v-if="regions.length === 0 && !loading" class="text-center py-8">
                            <i class="pi pi-inbox text-4xl text-color-secondary mb-3"></i>
                            <p class="text-color-secondary">No regions configured</p>
                            <Button
                                label="Add First Region"
                                icon="pi pi-plus"
                                @click="openCreateDialog"
                                class="mt-3"
                            />
                        </div>
                    </div>


                    <div class="flex align-items-center justify-between w-full p-5">
                        <h2 class="flex m-0 text-xl font-semibold">Live Stream</h2>
                        <Button
                            class="flex"
                            :label="isStreaming ? 'Stop' : 'Start'"
                            :icon="isStreaming ? 'pi pi-stop' : 'pi pi-play'"
                            :loading="streamLoading"
                            :disabled="streamLoading"
                            @click="toggleStream"
                            :severity="isStreaming ? 'danger' : 'primary'"
                            outlined
                        />
                    </div>


                    <div class="w-full px-5">
                        <img
                            v-show="isStreaming"
                            ref="imgRef"
                            class="w-full border-round-lg"
                            alt="Live video stream"
                        />
                        <p v-if="!isStreaming" class="text-center py-4 text-color-secondary">
                            Click "Start" to begin streaming.
                        </p>
                    </div>
                </template>
            </Card>
        </div>
    </div>

    <!-- Create/Edit Dialog -->
    <Dialog
        v-model:visible="showDialog"
        :header="dialogTitle"
        :modal="true"
        :closable="true"
        :draggable="false"
        :resizable="false"
        style="width: 500px"
        @hide="closeDialog"
    >
        <div class="grid formgrid p-fluid">
            <div class="field col-12">
                <label for="label">Label *</label>
                <InputText
                    id="label"
                    v-model="regionForm.label"
                    placeholder="Enter region label"
                    :class="{ 'p-invalid': regionForm.label.trim() === '' }"
                />
            </div>
            <div class="field col-12">
                <label>Bounding Box Coordinates *</label>
                <div class="grid formgrid p-fluid">
                    <div class="field col-6">
                        <label for="x1">X1 (Left)</label>
                        <InputNumber
                            id="x1"
                            v-model="regionForm.bbox.x1"
                            placeholder="X1"
                            :min="0"
                        />
                    </div>
                    <div class="field col-6">
                        <label for="y1">Y1 (Top)</label>
                        <InputNumber
                            id="y1"
                            v-model="regionForm.bbox.y1"
                            placeholder="Y1"
                            :min="0"
                        />
                    </div>
                    <div class="field col-6">
                        <label for="x2">X2 (Right)</label>
                        <InputNumber
                            id="x2"
                            v-model="regionForm.bbox.x2"
                            placeholder="X2"
                            :min="0"
                        />
                    </div>
                    <div class="field col-6">
                        <label for="y2">Y2 (Bottom)</label>
                        <InputNumber
                            id="y2"
                            v-model="regionForm.bbox.y2"
                            placeholder="Y2"
                            :min="0"
                        />
                    </div>
                </div>
                <small class="text-color-secondary">
                    Note: X2 must be greater than X1, and Y2 must be greater than Y1
                </small>
            </div>
        </div>

        <template #footer>
            <div class="flex justify-end gap-2">
                <Button
                    label="Cancel"
                    icon="pi pi-times"
                    @click="closeDialog"
                    text
                />
                <Button
                    :label="dialogMode === 'create' ? 'Create' : 'Update'"
                    :icon="dialogMode === 'create' ? 'pi pi-plus' : 'pi pi-check'"
                    @click="saveRegion"
                    :disabled="!isFormValid"
                    :loading="loading"
                />
            </div>
        </template>
    </Dialog>

    <Toast />
</template>

<style scoped lang="scss">
.p-datatable {
    .p-datatable-tbody > tr > td {
        vertical-align: middle;
    }
}

code {
    font-family: 'Courier New', monospace;
    font-size: 0.85em;
}
</style>
