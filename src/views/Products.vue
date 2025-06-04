<script setup>
import axios from "axios";
import { FilterMatchMode } from "@primevue/core/api";
import { useToast } from "primevue/usetoast";
import { onMounted, ref } from "vue";

const API_BASE = import.meta.env.VITE_API_BASE_PRODUCTS;

const toast = useToast();
const dt = ref();
const products = ref([]);
const productDialog = ref(false);
const deleteProductDialog = ref(false);
const deleteProductsDialog = ref(false);
const selectedProduct = ref({});
const selectedProducts = ref([]);
const filters = ref({
    global: { value: null, matchMode: FilterMatchMode.CONTAINS }
});
const submitted = ref(false);
const statuses = ref([
    { label: "INSTOCK", value: "instock" },
    { label: "LOWSTOCK", value: "lowstock" },
    { label: "OUTOFSTOCK", value: "outofstock" }
]);

onMounted(fetchProducts);

async function fetchProducts() {
    try {
        const { data } = await axios.get(`${API_BASE}/api/products`);
        products.value = data;
    } catch (e) {
        toast.add({ severity: "error", summary: "Error", detail: "Failed to fetch products", life: 3000 });
    }
}

function formatCurrency(value) {
    if (value) return value.toLocaleString("en-US", { style: "currency", currency: "USD" });
    return;
}

function openNew() {
    selectedProduct.value = {};
    submitted.value = false;
    productDialog.value = true;
}

function hideDialog() {
    productDialog.value = false;
    submitted.value = false;
}

async function saveProduct() {
    submitted.value = true;
    console.log(JSON.stringify(selectedProduct.value));
    if (selectedProduct?.value.name?.trim()) {
        try {
            if (selectedProduct.value.id) {
                // Update existing
                await axios.put(`${API_BASE}/${selectedProduct.value.id}`, selectedProduct.value);
                const idx = findIndexById(selectedProduct.value.id);
                products.value[idx] = { ...selectedProduct.value };
                toast.add({ severity: "success", summary: "Successful", detail: "Product Updated", life: 3000 });
            } else {
                // Create new
                const { data } = await axios.post(API_BASE, selectedProduct.value);
                products.value.push(data);
                toast.add({ severity: "success", summary: "Successful", detail: "Product Created", life: 3000 });
            }
            productDialog.value = false;
            selectedProduct.value = {};
        } catch (e) {
            toast.add({ severity: "error", summary: "Error", detail: "Failed to save product", life: 3000 });
        }
    }
}

function editProduct(prod) {
    selectedProduct.value = { ...prod };
    productDialog.value = true;
}

function confirmDeleteProduct(prod) {
    selectedProduct.value = prod;
    deleteProductDialog.value = true;
}

async function deleteProduct() {
    try {
        await axios.delete(`${API_BASE}/${selectedProduct.value.id}`);
        products.value = products.value.filter((val) => val.id !== selectedProduct.value.id);
        deleteProductDialog.value = false;
        selectedProduct.value = {};
        toast.add({ severity: "success", summary: "Successful", detail: "Product Deleted", life: 3000 });
    } catch (e) {
        toast.add({ severity: "error", summary: "Error", detail: "Failed to delete product", life: 3000 });
    }
}

function findIndexById(id) {
    return products.value.findIndex((p) => p.id === id);
}

function exportCSV() {
    dt.value.exportCSV();
}

function confirmDeleteSelected() {
    deleteProductsDialog.value = true;
}

async function deleteSelectedProducts() {
    try {
        // Optionally, batch delete on server (requires backend support)
        for (const prod of selectedProducts.value) {
            await axios.delete(`${API_BASE}/${prod.id}`);
        }
        products.value = products.value.filter((val) => !selectedProducts.value.includes(val));
        deleteProductsDialog.value = false;
        selectedProducts.value = [];
        toast.add({ severity: "success", summary: "Successful", detail: "Products Deleted", life: 3000 });
    } catch (e) {
        toast.add({ severity: "error", summary: "Error", detail: "Failed to delete selected products", life: 3000 });
    }
}

function getStatusLabel(status) {
    switch (status) {
        case "INSTOCK":
            return "success";
        case "LOWSTOCK":
            return "warn";
        case "OUTOFSTOCK":
            return "danger";
        default:
            return null;
    }
}
</script>

<template>
    <div>
        <div class="card">
            <Toolbar class="mb-6">
                <template #start>
                    <Button label="New" icon="pi pi-plus" severity="secondary" class="mr-2" @click="openNew" />
                    <Button label="Delete" icon="pi pi-trash" severity="secondary" @click="confirmDeleteSelected"
                            :disabled="!selectedProducts || !selectedProducts.length" />
                </template>

                <template #end>
                    <Button label="Export" icon="pi pi-upload" severity="secondary" @click="exportCSV($event)" />
                </template>
            </Toolbar>

            <DataTable
                ref="dt"
                v-model:selection="selectedProducts"
                :value="products"
                dataKey="id"
                :paginator="true"
                :rows="10"
                :filters="filters"
                paginatorTemplate="FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink CurrentPageReport RowsPerPageDropdown"
                :rowsPerPageOptions="[5, 10, 25]"
                currentPageReportTemplate="Showing {first} to {last} of {totalRecords} products"
            >
                <template #header>
                    <div class="flex flex-wrap gap-2 items-center justify-between">
                        <h4 class="m-0">Manage Products</h4>
                        <IconField>
                            <InputIcon>
                                <i class="pi pi-search" />
                            </InputIcon>
                            <InputText v-model="filters['global'].value" placeholder="Search..." />
                        </IconField>
                    </div>
                </template>

                <Column selectionMode="multiple" style="width: 3rem" :exportable="false"></Column>
                <Column field="code" header="Code" sortable style="min-width: 12rem"></Column>
                <Column field="name" header="Name" sortable style="min-width: 16rem"></Column>
                <Column header="Image">
                    <template #body="slotProps">
                        <img :src="`https://primefaces.org/cdn/primevue/images/product/${slotProps.data.image}`"
                             :alt="slotProps.data.image" class="rounded" style="width: 64px" />
                    </template>
                </Column>
                <Column field="price" header="Price" sortable style="min-width: 8rem">
                    <template #body="slotProps">
                        {{ formatCurrency(slotProps.data.price) }}
                    </template>
                </Column>
                <Column field="category" header="Category" sortable style="min-width: 10rem"></Column>
                <Column field="rating" header="Reviews" sortable style="min-width: 12rem">
                    <template #body="slotProps">
                        <Rating :modelValue="slotProps.data.rating" :readonly="true" />
                    </template>
                </Column>
                <Column field="inventoryStatus" header="Status" sortable style="min-width: 12rem">
                    <template #body="slotProps">
                        <Tag :value="slotProps.data.inventoryStatus"
                             :severity="getStatusLabel(slotProps.data.inventoryStatus)" />
                    </template>
                </Column>
                <Column :exportable="false" style="min-width: 12rem">
                    <template #body="slotProps">
                        <Button icon="pi pi-pencil" outlined rounded class="mr-2"
                                @click="editProduct(slotProps.data)" />
                        <Button icon="pi pi-trash" outlined rounded severity="danger"
                                @click="confirmDeleteProduct(slotProps.data)" />
                    </template>
                </Column>
            </DataTable>
        </div>

        <ProductDialog
            v-model:visible="productDialog"
            v-model:product="selectedProduct"
            :statuses="statuses"
            @save="saveProduct"
            @close="productDialog = false"
        />

        <Dialog v-model:visible="deleteProductDialog" :style="{ width: '450px' }" header="Confirm" :modal="true">
            <div class="flex items-center gap-4">
                <i class="pi pi-exclamation-triangle !text-3xl" />
                <span v-if="selectedProduct"
                >Are you sure you want to delete <b>{{ selectedProduct.name }}</b
                >?</span
                >
            </div>
            <template #footer>
                <Button label="No" icon="pi pi-times" text @click="deleteProductDialog = false" />
                <Button label="Yes" icon="pi pi-check" @click="deleteProduct" />
            </template>
        </Dialog>

        <Dialog v-model:visible="deleteProductsDialog" :style="{ width: '450px' }" header="Confirm" :modal="true">
            <div class="flex items-center gap-4">
                <i class="pi pi-exclamation-triangle !text-3xl" />
                <span v-if="selectedProduct">Are you sure you want to delete the selected products?</span>
            </div>
            <template #footer>
                <Button label="No" icon="pi pi-times" text @click="deleteProductsDialog = false" />
                <Button label="Yes" icon="pi pi-check" text @click="deleteSelectedProducts" />
            </template>
        </Dialog>
    </div>
</template>
