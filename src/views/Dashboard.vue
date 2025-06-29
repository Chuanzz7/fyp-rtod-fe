<template>
    <div class="grid">
        <div class="col-12">
            <div class="card">
                <h5>Application Performance Pipeline</h5>
                <p>
                    Live metrics from the application's processing stages, automatically refreshing every 3 seconds.
                    Last updated: {{ lastUpdateTime || 'N/A' }}
                </p>
            </div>
        </div>

        <!-- Loading Spinner -->
        <div v-if="loading" class="col-12 text-center">
            <ProgressSpinner />
            <p>Fetching initial metrics...</p>
        </div>

        <!-- Error Message -->
        <div v-if="error" class="col-12">
            <Message severity="error" :closable="false">{{ error }}</Message>
        </div>

        <!-- ======================================================= -->
        <!-- TIMELINE 1: FRAME INGESTION (LEFT ALIGNED)              -->
        <!-- ======================================================= -->
        <div class="col-12" v-if="!loading">
            <div class="card">
                <Timeline :value="ingestionSteps" align="left" class="customized-timeline">
                    <!-- The #marker slot now checks if the item is a header -->
                    <template #marker="slotProps">
                        <span v-if="slotProps.item.isHeader" class="flex w-10 h-10 align-items-center justify-content-center text-white border-circle z-1 shadow-2" :style="{ backgroundColor: slotProps.item.color }">
                            <i :class="[slotProps.item.icon]" style="font-size: xx-large"></i>
                        </span>
                        <span v-else class="flex w-2rem h-2rem align-items-center justify-content-center text-white border-circle z-1 shadow-1" :style="{ backgroundColor: slotProps.item.color }"></span>
                    </template>
                    <!-- The #content slot also checks if the item is a header -->
                    <template #content="slotProps">
                        <div v-if="slotProps.item.isHeader" class="text-2xl font-bold p-2">{{ slotProps.item.title }}</div>
                        <Card v-else>
                            <template #title><div class="text-lg font-semibold">{{ formatMetricName(slotProps.item.name) }}</div></template>
                            <template #subtitle><div class="text-base">{{ getUnit(slotProps.item.name) }}</div></template>
                            <template #content>
                                <div class="mb-3">
                                    <span class="text-500">Last: </span>
                                    <span class="text-green-500 font-medium text-2xl">{{ slotProps.item.metric.last.toFixed(3) }}</span>
                                </div>
                                <Chart type="line" :data="chartData[slotProps.item.name]" :options="chartOptions"></Chart>
                            </template>
                        </Card>
                    </template>
                    <template #opposite="slotProps">
                        <div v-if="!slotProps.item.isHeader" class="p-2">
                            <div class="text-500">Avg: <strong class="text-700">{{ slotProps.item.metric.avg.toFixed(3) }}</strong></div>
                            <div class="text-500">Max: <strong class="text-700">{{ slotProps.item.metric.max.toFixed(3) }}</strong></div>
                        </div>
                    </template>
                </Timeline>
            </div>
        </div>

        <!-- ======================================================= -->
        <!-- TIMELINE 2: AI PROCESSING (RIGHT ALIGNED)               -->
        <!-- ======================================================= -->
        <div class="col-12" v-if="!loading">
            <div class="card">
                <Timeline :value="aiProcessingSteps" align="right" class="customized-timeline">
                    <!-- The #marker slot now checks if the item is a header -->
                    <template #marker="slotProps">
                        <span v-if="slotProps.item.isHeader" class="flex w-10 h-10 align-items-center justify-content-center text-white border-circle z-1 shadow-2" :style="{ backgroundColor: slotProps.item.color }">
                            <i :class="[slotProps.item.icon]" style="font-size: xx-large"></i>
                        </span>
                        <span v-else class="flex w-2rem h-2rem align-items-center justify-content-center text-white border-circle z-1 shadow-1" :style="{ backgroundColor: slotProps.item.color }"></span>
                    </template>
                    <!-- The #content slot also checks if the item is a header -->
                    <template #content="slotProps">
                        <div v-if="slotProps.item.isHeader" class="text-2xl font-bold p-2">{{ slotProps.item.title }}</div>
                        <Card v-else>
                            <template #title><div class="text-lg font-semibold">{{ formatMetricName(slotProps.item.name) }}</div></template>
                            <template #subtitle><div class="text-base">{{ getUnit(slotProps.item.name) }}</div></template>
                            <template #content>
                                <div class="mb-3">
                                    <span class="text-500">Last: </span>
                                    <span class="text-green-500 font-medium text-2xl">{{ slotProps.item.metric.last.toFixed(3) }}</span>
                                </div>
                                <Chart type="line" :data="chartData[slotProps.item.name]" :options="chartOptions"></Chart>
                            </template>
                        </Card>
                    </template>
                    <template #opposite="slotProps">
                        <div v-if="!slotProps.item.isHeader" class="p-2">
                            <div class="text-500">Avg: <strong class="text-700">{{ slotProps.item.metric.avg.toFixed(3) }}</strong></div>
                            <div class="text-500">Max: <strong class="text-700">{{ slotProps.item.metric.max.toFixed(3) }}</strong></div>
                        </div>
                    </template>
                </Timeline>
            </div>
        </div>

        <!-- ======================================================= -->
        <!-- TIMELINE 3: OUTPUT GENERATION (LEFT ALIGNED)            -->
        <!-- ======================================================= -->
        <div class="col-12" v-if="!loading">
            <div class="card">
                <Timeline :value="outputSteps" align="left" class="customized-timeline">
                    <!-- The #marker slot now checks if the item is a header -->
                    <template #marker="slotProps">
                        <span v-if="slotProps.item.isHeader" class="flex w-10 h-10 align-items-center justify-content-center text-white border-circle z-1 shadow-2" :style="{ backgroundColor: slotProps.item.color }">
                            <i :class="[slotProps.item.icon]" style="font-size: xx-large" ></i>
                        </span>
                        <span v-else class="flex w-2rem h-2rem align-items-center justify-content-center text-white border-circle z-1 shadow-1" :style="{ backgroundColor: slotProps.item.color }"></span>
                    </template>
                    <!-- The #content slot also checks if the item is a header -->
                    <template #content="slotProps">
                        <div v-if="slotProps.item.isHeader" class="text-2xl font-bold p-2">{{ slotProps.item.title }}</div>
                        <Card v-else>
                            <template #title><div class="text-lg font-semibold">{{ formatMetricName(slotProps.item.name) }}</div></template>
                            <template #subtitle><div class="text-base">{{ getUnit(slotProps.item.name) }}</div></template>
                            <template #content>
                                <div class="mb-3">
                                    <span class="text-500">Last: </span>
                                    <span class="text-green-500 font-medium text-2xl">{{ slotProps.item.metric.last.toFixed(3) }}</span>
                                </div>
                                <Chart type="line" :data="chartData[slotProps.item.name]" :options="chartOptions"></Chart>
                            </template>
                        </Card>
                    </template>
                    <template #opposite="slotProps">
                        <div v-if="!slotProps.item.isHeader" class="p-2">
                            <div class="text-500">Avg: <strong class="text-700">{{ slotProps.item.metric.avg.toFixed(3) }}</strong></div>
                            <div class="text-500">Max: <strong class="text-700">{{ slotProps.item.metric.max.toFixed(3) }}</strong></div>
                        </div>
                    </template>
                </Timeline>
            </div>
        </div>

    </div>
</template>


<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';
import axios from 'axios';
const BASE_URL = import.meta.env.VITE_API_BASE_PROCESSOR ?? "";

// --- Reactive State (unchanged) ---
const metrics = ref({});
const loading = ref(true);
const error = ref(null);
const lastUpdateTime = ref(null);
const chartData = ref({});
let intervalId = null;

// --- NEW: Define the structure and order of our pipeline groups ---
// This is the single source of truth for the timeline layout.
const pipelineGroups = [
    {
        title: 'Frame Ingestion',
        icon: 'pi pi-camera',
        color: '#3B82F6', // Blue
        metrics: ['upload_frame_qps']
    },
    {
        title: 'AI Processing',
        icon: 'pi pi-microchip',
        color: '#8B5CF6', // Purple
        metrics: [
            'decode_time_ms',
            'dfine_inference_time_ms',
            'sort_and_cache_time_ms',
            'draw_time_ms',
            'ocr_time_ms',
            'total_processing_time_ms'
        ]
    },
    {
        title: 'Output Generation',
        icon: 'pi pi-chart-line',
        color: '#10B981', // Green
        metrics: [
            'output_panel_time_ms',
            'output_api_time_ms',
            'output_draw_time_ms',
            'output_encode_time_ms',
            'output_total_processing_time_ms',
            'output_fps'
        ]
    }
];



/// --- NEW: Helper function and 3 separate computed properties ---
// A reusable function to generate the steps for a specific group.
const createTimelineSteps = (groupTitle) => {
    const group = pipelineGroups.find(g => g.title === groupTitle);
    if (!group) return [];

    // Create the special header object for this group
    const header = {
        isHeader: true,
        title: group.title,
        icon: group.icon,
        color: group.color
    };

    // Create the array of regular metric items
    const metricItems = group.metrics.map(metricName => {
        const metricData = metrics.value[metricName];
        if (metricData) {
            return {
                isHeader: false, // Mark this as a regular item
                name: metricName,
                metric: metricData,
                color: group.color,
            };
        }
        return null;
    }).filter(Boolean);

    // Return a new array with the header at the start
    return [header, ...metricItems];
};

// Create a computed property for each timeline.
const ingestionSteps = computed(() => createTimelineSteps('Frame Ingestion'));
const aiProcessingSteps = computed(() => createTimelineSteps('AI Processing'));
const outputSteps = computed(() => createTimelineSteps('Output Generation'));

// --- Chart Options (unchanged) ---
const chartOptions = {
    animation: { duration: 0 },
    maintainAspectRatio: false,
    aspectRatio: 1.8,
    plugins: { legend: { display: false } },
    scales: {
        x: { ticks: { display: false }, grid: { display: false } },
        y: { ticks: { color: '#495057', font: { size: 10 } }, grid: { color: '#ebedef' } }
    },
    elements: { point: { radius: 0 } },
    tension: 0.4
};

// --- Data Fetching, Chart Updates, Lifecyle, Helpers (all unchanged) ---
const fetchData = async () => {
    try {
        const response = await axios.get(`${BASE_URL}/api/metrics`);
        const data = response.data;
        error.value = null;
        if (data.status === 'success' && data.metrics) {
            metrics.value = data.metrics;
            lastUpdateTime.value = new Date().toLocaleTimeString();
            updateAllCharts(data.metrics);
        } else { throw new Error(data.message || 'API response was not in the expected format.'); }
    } catch (e) {
        console.error("Failed to fetch metrics:", e);
        if (e.response) { error.value = `API Error: ${e.response.status} - ${e.response.data.message || 'Server Error'}`; }
        else if (e.request) { error.value = 'Failed to connect to the API. Is the backend server running?'; }
        else { error.value = `An unexpected error occurred: ${e.message}`; }
    } finally { loading.value = false; }
};

const MAX_CHART_POINTS = 30;
const updateAllCharts = (newMetrics) => {
    for (const name in newMetrics) {
        const lastValue = newMetrics[name].last;
        const currentData = chartData.value[name]?.datasets[0]?.data || [];
        const newData = [...currentData, lastValue];
        if (newData.length > MAX_CHART_POINTS) newData.shift();
        chartData.value[name] = {
            labels: Array(newData.length).fill(''),
            datasets: [{
                label: 'Last Value', data: newData, fill: true,
                borderColor: '#42A5F5', backgroundColor: 'rgba(66,165,245,0.2)', borderWidth: 2
            }]
        };
    }
};

onMounted(() => {
    fetchData();
    intervalId = setInterval(fetchData, 3000);
});
onUnmounted(() => {
    if (intervalId) clearInterval(intervalId);
});

const formatMetricName = (name) => name.split('_').map(word => word.charAt(0).toUpperCase() + word.slice(1)).join(' ');
const getUnit = (name) => {
    if (name.includes('_ms')) return 'ms';
    if (name.includes('qps') || name.includes('fps')) return name.split('_').pop().toUpperCase();
    return 'Value';
};
</script>

<style lang="scss" scoped>
::v-deep(.customized-timeline) {
    .p-timeline-event:nth-child(even) {
        @media screen and (max-width: 960px) {
            flex-direction: row !important;
            .p-timeline-event-content { text-align: left !important; }
        }
    }
    .p-timeline-event-opposite {
        @media screen and (max-width: 960px) {
            flex: 0;
            padding: 0;
        }
    }
    .p-card {
        margin-top: 1rem;
    }
}
</style>
