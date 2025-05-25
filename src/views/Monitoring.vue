<script setup>
import { onBeforeUnmount, ref } from "vue";
import Card from "primevue/card";
import Button from "primevue/button";
import axios from "axios";

const BASE_URL = import.meta.env.VITE_API_BASE ?? "";
const STREAM_URL = `${BASE_URL}/video_feed`;
// const STREAM_URL = 'https://placecats.com/neo_banana/300/200'

const imgRef = ref(null);
const isStreaming = ref(false);
const loading = ref(false);
let retryInterval = null;
const fps = ref(0);
let frameCount = 0;

// This will be called every time a new image loads
function onFrameDisplayed() {
    frameCount++;
}

setInterval(() => {
    fps.value = frameCount;
    frameCount = 0;
}, 1000);

async function startStream() {
    if (isStreaming.value || loading.value) return;
    loading.value = true;
    try {
        // Adjust the endpoint as needed for your backend
        await axios.post(`${BASE_URL}/start`);
        setTimeout(() => {
            if (imgRef.value) {
                imgRef.value.src = `${STREAM_URL}?t=${Date.now()}`;
                isStreaming.value = true;
                loading.value = false;
                startRetry();
            }
        }, 200);
    } catch (e) {
        // Handle error (show message, etc.)
        console.error("Failed to start stream:", e);
        loading.value = false;
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
        loading.value = false;
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

onBeforeUnmount(() => {
    clearInterval(retryInterval);
});
</script>

<template>
    <div class="p-card w-full">
        <Card>
            <template #header>
                <div class="flex align-items-center justify-content-between w-full">
                    <h2 class="m-0 text-xl font-semibold">Live Stream</h2>
                    <Button
                        :label="isStreaming ? 'Stop' : 'Start'"
                        :icon="isStreaming ? 'pi pi-stop' : 'pi pi-play'"
                        :loading="loading"
                        :disabled="loading"
                        @click="toggleStream"
                        :severity="isStreaming ? 'danger' : 'primary'"
                        outlined
                    />
                </div>
            </template>

            <template #content>
                <div class="w-full">
                    <img
                        v-show="isStreaming"
                        ref="imgRef"
                        class="w-full border-round-lg"
                        alt="Live video stream"
                        @load="onFrameDisplayed"
                    />
                    <div v-show="isStreaming" class="text-right text-sm text-color-secondary pr-2 mt-2">
                        FPS: {{ fps }}
                    </div>
                    <p v-if="!isStreaming" class="text-center py-4 text-color-secondary">
                        Click “Start” to begin streaming.
                    </p>
                </div>
            </template>
        </Card>
    </div>
</template>

<style scoped lang="scss">
</style>
