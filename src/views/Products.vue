<script setup>
import axios from "axios";
import { FilterMatchMode } from "@primevue/core/api";
import { useToast } from "primevue/usetoast";
import { onMounted, ref } from "vue";
import ProductDialog from '../components/product/ProductDialog.vue';

const API_BASE = import.meta.env.VITE_API_BASE_PRODUCTS;

const toast = useToast();

// For product lookup feature
const lookupDialog = ref(false);
const lookupCategory = ref('');
const lookupOcr = ref('');
const lookupResult = ref(null);
const lookupLoading = ref(false);
const dt = ref();
const products = ref([]);
const productDialogData = ref(false);
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
    productDialogData.value = true;
}

function hideDialog() {
    productDialogData.value = false;
    submitted.value = false;
}

async function saveProduct() {
    submitted.value = true;
    if (selectedProduct?.value.name?.trim()) {
        try {
            const formData = new FormData();

            // Add all form fields to FormData
            formData.append('name', selectedProduct.value.name);
            formData.append('description', selectedProduct.value.description || '');
            formData.append('category', selectedProduct.value.category || '');
            formData.append('price', selectedProduct.value.price || 0);
            formData.append('quantity', selectedProduct.value.quantity || 0);
            formData.append('ocr', selectedProduct.value.ocr || '');

            // Handle inventory status - it could be an object or a string
            let inventoryStatus = 'INSTOCK';

            if (selectedProduct.value.inventoryStatus) {
                if (typeof selectedProduct.value.inventoryStatus === 'object' && selectedProduct.value.inventoryStatus.value) {
                    // It's an object with a value property
                    inventoryStatus = selectedProduct.value.inventoryStatus.value;
                } else if (typeof selectedProduct.value.inventoryStatus === 'string') {
                    // It's already a string
                    inventoryStatus = selectedProduct.value.inventoryStatus;
                }
            }

            formData.append('inventoryStatus', inventoryStatus);

            // Add image file if present
            if (selectedProduct.value.imageFile) {
                formData.append('image', selectedProduct.value.imageFile);
            } else if (selectedProduct.value.image && typeof selectedProduct.value.image === 'string' && selectedProduct.value.image.startsWith('data:')) {
                // Convert data URL to File object if needed
                const imageFile = dataURLtoFile(selectedProduct.value.image, 'product-image.jpg');
                formData.append('image', imageFile);
            }

            let data;
            if (selectedProduct.value.id) {
                // Update existing product
                const response = await axios.put(`${API_BASE}/api/products/${selectedProduct.value.id}`, formData, {
                    headers: {
                        'Content-Type': 'multipart/form-data'
                    }
                });
                data = response.data;
                const idx = findIndexById(selectedProduct.value.id);
                products.value[idx] = data;
                toast.add({ severity: "success", summary: "Successful", detail: "Product Updated", life: 3000 });
            } else {
                // Create new product
                const response = await axios.post(`${API_BASE}/api/products`, formData, {
                    headers: {
                        'Content-Type': 'multipart/form-data'
                    }
                });
                data = response.data;
                products.value.push(data);
                toast.add({ severity: "success", summary: "Successful", detail: "Product Created", life: 3000 });
            }
            productDialogData.value = false;
            selectedProduct.value = {};
        } catch (e) {
            console.error('Error saving product:', e);
            toast.add({
                severity: "error",
                summary: "Error",
                detail: e.response?.data?.detail || "Failed to save product",
                life: 3000
            });
        }
    }
}

function editProduct(prod) {
    // Create a copy of the product
    const productCopy = { ...prod };

    // Map string inventory status to the correct object format if needed
    if (productCopy.inventoryStatus && typeof productCopy.inventoryStatus === 'string') {
        const matchingStatus = statuses.value.find(s => s.value === productCopy.inventoryStatus.toLowerCase());
        if (matchingStatus) {
            productCopy.inventoryStatus = matchingStatus;
        }
    }

    selectedProduct.value = productCopy;
    productDialogData.value = true;
}

function confirmDeleteProduct(prod) {
    selectedProduct.value = prod;
    deleteProductDialog.value = true;
}

async function deleteProduct() {
    try {
        await axios.delete(`${API_BASE}/api/products/${selectedProduct.value.id}`);
        products.value = products.value.filter((val) => val.id !== selectedProduct.value.id);
        deleteProductDialog.value = false;
        selectedProduct.value = {};
        toast.add({ severity: "success", summary: "Successful", detail: "Product Deleted", life: 3000 });
    } catch (e) {
        console.error('Error deleting product:', e);
        toast.add({
            severity: "error",
            summary: "Error",
            detail: e.response?.data?.detail || "Failed to delete product",
            life: 3000
        });
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
        // Process each product deletion sequentially
        for (const prod of selectedProducts.value) {
            await axios.delete(`${API_BASE}/api/products/${prod.id}`);
        }
        products.value = products.value.filter((val) => !selectedProducts.value.includes(val));
        deleteProductsDialog.value = false;
        selectedProducts.value = [];
        toast.add({ severity: "success", summary: "Successful", detail: "Products Deleted", life: 3000 });
    } catch (e) {
        console.error('Error deleting products:', e);
        toast.add({
            severity: "error",
            summary: "Error",
            detail: e.response?.data?.detail || "Failed to delete selected products",
            life: 3000
        });
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

function getImageUrl(imageName) {
    if (!imageName) return '';

    // Check if it's a placeholder image
    if (imageName === 'product-placeholder.svg') {
        return 'https://primefaces.org/cdn/primevue/images/product/product-placeholder.svg';
    }

    // Handle data URLs directly
    if (typeof imageName === 'string' && imageName.startsWith('data:')) {
        return imageName;
    }

    // Return the URL to the uploaded image on the server
    return `${API_BASE}/uploads/${imageName}`;
}

// Helper function to convert data URL to File object
function dataURLtoFile(dataURL, filename) {
    const arr = dataURL.split(',');
    const mime = arr[0].match(/:(.*?);/)[1];
    const bstr = atob(arr[1]);
    let n = bstr.length;
    const u8arr = new Uint8Array(n);

    while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
    }

    return new File([u8arr], filename, { type: mime });
}

async function performProductLookup() {
    if (!lookupCategory.value || !lookupOcr.value) {
        toast.add({
            severity: "warn",
            summary: "Missing Information",
            detail: "Please provide both category and OCR text",
            life: 3000
        });
        return;
    }

    lookupLoading.value = true;
    lookupResult.value = null;

    try {
        const { data } = await axios.post(`${API_BASE}/api/product_lookup`, {
            category: lookupCategory.value,
            ocr: lookupOcr.value
        });

        lookupResult.value = data;
        toast.add({
            severity: "success",
            summary: "Product Found",
            detail: `Found product: ${data.name} (${Math.round(data.confidence * 100)}% match)`,
            life: 3000
        });
    } catch (e) {
        console.error('Lookup error:', e);
        toast.add({
            severity: "error",
            summary: "Lookup Failed",
            detail: e.response?.data?.detail || "No matching product found",
            life: 3000
        });
    } finally {
        lookupLoading.value = false;
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
                            :disabled="!selectedProducts || !selectedProducts.length" class="mr-2" />
                    <Button label="Lookup" icon="pi pi-search" severity="info" @click="lookupDialog = true" />
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
                <Column field="category" header="Category" sortable style="min-width: 10rem"></Column>
                <Column field="name" header="Name" sortable style="min-width: 16rem"></Column>
                <Column field="quantity" header="Quantity" sortable style="min-width: 8rem"></Column>
                <Column header="Image">
                    <template #body="slotProps">
                        <img :src="getImageUrl(slotProps.data.image)"
                             :alt="slotProps.data.image" class="rounded" style="width: 64px" />
                    </template>
                </Column>
                <Column field="price" header="Price" sortable style="min-width: 8rem">
                    <template #body="slotProps">
                        {{ formatCurrency(slotProps.data.price) }}
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
            v-model:visible="productDialogData"
            v-model:product="selectedProduct"
            :statuses="statuses"
            @save="saveProduct"
            @close="productDialogData = false"
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

        <!-- Product Lookup Dialog -->
        <Dialog v-model:visible="lookupDialog" header="Product Lookup" :modal="true" :style="{ width: '550px' }">
            <div class="flex flex-col gap-4">
                <!-- Category field -->
                <div>
                    <label for="lookupCategory" class="block font-bold mb-2">Category</label>
                    <InputText id="lookupCategory" v-model="lookupCategory" class="w-full" />
                </div>

                <!-- OCR Text field -->
                <div>
                    <label for="lookupOcr" class="block font-bold mb-2">OCR Text</label>
                    <Textarea id="lookupOcr" v-model="lookupOcr" rows="4" class="w-full" />
                </div>

                <!-- Lookup button -->
                <div class="flex justify-end">
                    <Button label="Lookup Product" icon="pi pi-search" @click="performProductLookup" :loading="lookupLoading" />
                </div>

                <!-- Results section -->
                <div v-if="lookupResult" class="mt-4 p-4 border rounded bg-gray-50">
                    <h3 class="text-xl font-bold mb-3">Found Product</h3>
                    <div class="flex items-start gap-3">
                        <img :src="getImageUrl(lookupResult.image)" class="w-20 h-20 object-contain" :alt="lookupResult.name" />
                        <div>
                            <p class="font-bold text-lg">{{ lookupResult.name }}</p>
                            <p class="text-sm text-gray-600">Category: {{ lookupResult.category }}</p>
                            <p class="text-sm text-gray-600">Price: {{ formatCurrency(lookupResult.price) }}</p>
                            <p class="text-sm text-green-600 font-bold">Match confidence: {{ Math.round(lookupResult.confidence * 100) }}%</p>
                        </div>
                    </div>
                </div>
            </div>
        </Dialog>
    </div>
</template>
