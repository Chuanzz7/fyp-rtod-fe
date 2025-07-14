<script setup>
import { ref, watch } from "vue";
import { useToast } from "primevue/usetoast";
import axios from "axios";

const PRODUCT_BASE = import.meta.env.VITE_API_BASE_PRODUCTS ?? "";

const toast = useToast();

const props = defineProps({
    visible: Boolean
});
const emit = defineEmits(["update:visible", "save", "cancel"]);

const visible = ref(props.visible);
watch(() => props.visible, (v) => visible.value = v);
watch(visible, (v) => emit("update:visible", v));

// For controlled input
const product = defineModel("product");
const submitted = ref(false);

// Image upload states
const preview = ref("");
const imageFile = ref(null);
const fileInput = ref(null);
const dropZone = ref(null);

// Watch for changes to product and update preview accordingly
watch(() => product.value, (newProduct) => {
    if (newProduct && newProduct.image) {
        preview.value = typeof newProduct.image === 'string' && newProduct.image.startsWith('data:')
            ? newProduct.image
            : `${import.meta.env.VITE_API_BASE_PRODUCTS}/uploads/${newProduct.image}`;
    } else {
        preview.value = "";
    }
}, { immediate: true });

// Result and loading state
const result = ref(null);
const loading = ref(false);

// Only allow one image, replace if user uploads again
function onFileChange(e) {
    const file = e.target.files[0];
    handleImageFile(file);
}

function onDrop(e) {
    const file = e.dataTransfer.files[0];
    handleImageFile(file);
}

function handleImageFile(file) {
    if (file && file.type.startsWith("image/")) {
        imageFile.value = file;
        // Store the actual file object in product.value.imageFile for form submission
        product.value.imageFile = file;

        const reader = new FileReader();
        reader.onload = e => {
            preview.value = e.target.result;
            product.value.image = e.target.result; // Preview image (data URL)
        };
        reader.readAsDataURL(file);
        result.value = null; // clear result when new image uploaded
    }
}

function removeImage() {
    imageFile.value = null;
    preview.value = "";
    product.value.image = "";
    product.value.imageFile = null; // Also clear the file object from product.value
    result.value = null;
}

// Send image to backend for detection and OCR
async function onDetect() {
    if (!imageFile.value) return;
    loading.value = true;
    result.value = null;
    const formData = new FormData();
    formData.append("image", imageFile.value);

    try {
        const res = await axios.post(`${PRODUCT_BASE}/api/detect_item`, formData, {
            headers: {
                "Content-Type": "multipart/form-data"
            }
        });
        // Axios automatically parses JSON
        result.value = res.data;
        product.value.category = result.value.detections[0].class_name;
        product.value.ocr = result.value.ocrTexts;

    } catch (e) {
        toast.add({ severity: "error", summary: "Failed", detail: e.response?.data?.detail, life: 3000 });
    }
    loading.value = false;
}

function hideDialog() {
    emit("cancel");
    visible.value = false;
}

function saveProduct() {
    submitted.value = true;

    // Check if the product has all required fields
    if (!product.value.name?.trim()) {
        toast.add({ severity: "error", summary: "Validation Error", detail: "Product name is required", life: 3000 });
        return;
    }

    // For new uploads, ensure detection has been run
    if (imageFile.value && !result.value) {
        toast.add({ severity: "error", summary: "No Image Detect", detail: "Please detect before save", life: 3000 });
        return;
    }

    // If editing with existing image and no new upload, allow save without detection
    if (!imageFile.value && preview.value && !result.value && product.value.id) {
        // Continue with save for edit operations with existing images
    }

    emit("save", { ...product });
    visible.value = false;
}

function copyToClipboard(text) {
    navigator.clipboard.writeText(text)
        .then(() => {
            toast.add({ severity: "success", summary: "Copied", detail: "Text copied to clipboard", life: 2000 });
        })
        .catch(() => {
            toast.add({ severity: "error", summary: "Failed", detail: "Could not copy text", life: 3000 });
        });
}

const statuses = ref([
    { label: "INSTOCK", value: "instock" },
    { label: "LOWSTOCK", value: "lowstock" },
    { label: "OUTOFSTOCK", value: "outofstock" }
]);
</script>


<template>
    <Dialog v-model:visible="visible" :style="{ width: '1000px' }" header="Product Details" :modal="true">
        <div class="flex flex-col gap-6">

            <!-- Image Upload + Preview + Detect Button -->
            <div>
                <label class="block font-bold mb-3">Image</label>
                <div
                    ref="dropZone"
                    class="flex items-center gap-3 border-2 border-dashed rounded p-3 text-center cursor-pointer transition-all duration-200 hover:bg-gray-50 hover:border-primary"
                    @dragover.prevent="(e) => dropZone.value?.classList.add('bg-gray-50', 'border-primary')"
                    @dragleave.prevent="(e) => dropZone.value?.classList.remove('bg-gray-50', 'border-primary')"
                    @drop.prevent="(e) => { onDrop(e); dropZone.value?.classList.remove('bg-gray-50', 'border-primary'); }"
                >
                    <input
                        ref="fileInput"
                        type="file"
                        accept="image/*"
                        class="hidden"
                        @change="onFileChange"
                    />
                    <div class="flex-1 flex flex-col items-center justify-center" @click="fileInput.click()" style="min-height: 90px;">
                        <template v-if="preview">
                            <img :src="preview" alt="Product" class="block m-auto max-h-24 mb-2" />
                        </template>
                        <template v-else>
                            <i class="pi pi-image text-gray-400 text-3xl mb-2"></i>
                            <p class="text-gray-500">Click or drag image here to upload</p>
                            <p class="text-xs text-gray-400 mt-1">Supported formats: JPG, PNG, GIF</p>
                        </template>
                    </div>
                    <div class="flex flex-col gap-2 items-center">
                        <Button icon="pi pi-upload" label="Detect & OCR" class="mb-2" @click="onDetect"
                                :disabled="!imageFile || loading" />
                        <Button icon="pi pi-times" text @click="removeImage" label="Remove" :disabled="!imageFile" />
                    </div>
                </div>
                <!-- Combined Detection + OCR Result Card (non-editable, always visible if result exists) -->
                <div v-if="result" class="bg-white rounded-2xl shadow px-4 py-2 border mt-5">
                    <div class="flex items-center gap-2 ">
                        <i class="pi pi-eye text-blue-500"></i>
                        <h3 class="font-semibold text-lg">Detection & OCR Result</h3>
                    </div>
                    <!-- Detected Object -->
                    <div class="mb-2">
                        <b>Detected Object : </b>
                        <span v-if="result.detections && result.detections.length">{{ result.detections[0].class_name }}
                       <span
                           v-if="result.detections[0].confidence">({{ (result.detections[0].confidence * 100).toFixed(1)
                           }}%)</span></span>
                        <span v-else class="text-gray-500">No object detected.</span>
                    </div>
                    <!-- OCR Text -->
                    <div>
                        <b>OCR Text:</b>
                        <div
                            class="whitespace-pre-line p-2 rounded-lg border bg-gray-50 text-gray-700 select-all mt-1"
                            :style="{ minHeight: '60px', cursor: result.ocrTexts ? 'copy' : 'default' }"
                            title="Click to copy"
                            @click="result.ocrTexts && copyToClipboard(result.ocrTexts)">
                            {{ result.ocrTexts || "No text found." }}
                        </div>
                    </div>
                </div>
            </div>

            <!-- Name -->
            <div>
                <label for="name" class="block font-bold mb-3">Name</label>
                <InputText id="name" v-model.trim="product.name" required autofocus
                           :invalid="submitted && !product.name" fluid />
                <small v-if="submitted && !product.name" class="text-red-500">Name is required.</small>
            </div>

            <!-- Description -->
            <div>
                <label for="description" class="block font-bold mb-3">Description</label>
                <Textarea id="description" v-model="product.description" rows="3" cols="20"
                          :invalid="submitted && !product.description" fluid />
                <small v-if="submitted && !product.description" class="text-red-500">description is required.</small>
            </div>

            <!-- Price and quantity in same row -->
            <div class="grid grid-cols-12 gap-4">
                <div class="col-span-6">
                    <label for="price" class="block font-bold mb-3">Price ($)</label>
                    <InputNumber id="price" v-model="product.price" mode="currency" currency="USD" locale="en-US"
                                 :invalid="submitted && !product.price" fluid />
                    <small v-if="submitted && !product.price" class="text-red-500">Price is required.</small>
                </div>
                <div class="col-span-6">
                    <label for="quantity" class="block font-bold mb-3">Quantity</label>
                    <InputNumber id="quantity" v-model="product.quantity" :invalid="submitted && !product.quantity"
                                 integeronly fluid />
                    <small v-if="submitted && !product.quantity" class="text-red-500">Quantity is required.</small>
                </div>
            </div>
        </div>

        <template #footer>
            <Button label="Cancel" icon="pi pi-times" text @click="hideDialog" />
            <Button label="Save" icon="pi pi-check" @click="saveProduct" />
        </template>
    </Dialog>
</template>

