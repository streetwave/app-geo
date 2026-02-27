<template>
    <div class="flex h-screen bg-base-300">
        <div class="w-1/2 h-full flex flex-col gap-10">
            <!-- Input Field -->
            <div class="h-1/4 p-2">
                <textarea
                    class="w-full h-full border border-gray-300 rounded-md bg-base-100 p-2 resize-none"
                    v-model="inputField"
                    placeholder="Enter GeoJSON or WKT..."
                ></textarea>
            </div>

            <!-- Parsed GeoJSON and WKT -->
            <div class="flex h-2/3">
                <div class="flex-1 h-full p-2 flex flex-col gap-2">
                    <h2 class="text-lg font-bold">GeoJSON</h2>
                    <div class="relative h-full">
                        <button
                            v-if="stringifiedGeoJson"
                            @click="copyToClipboard(stringifiedGeoJson)"
                            class="btn btn-neutral btn-outline absolute top-3 right-5 z-10"
                        >
                            Copy
                        </button>
                        <textarea
                            class="w-full h-full border border-gray-300 bg-base-100 rounded-md resize-none p-2"
                            v-model="stringifiedGeoJson"
                            disabled
                        ></textarea>
                    </div>
                    <button @click="downloadGeojson" :disabled="!isValid" class="btn btn-primary w-full">Download</button>
                </div>

                <div class="flex-1 h-full p-2 flex flex-col gap-2">
                    <h2 class="text-lg font-bold">WKT (Well Known Text)</h2>
                    <div class="relative h-full">
                        <button v-if="parsedWkt" @click="copyToClipboard(parsedWkt)" class="btn btn-neutral btn-outline absolute top-3 right-5 z-10">
                            Copy
                        </button>
                        <textarea
                            class="w-full h-full border border-gray-300 bg-base-100 rounded-md resize-none p-2"
                            v-model="parsedWkt"
                            disabled
                        ></textarea>
                    </div>
                    <button @click="downloadWkt" :disabled="!isValid" class="btn btn-primary w-full">Download</button>
                </div>
            </div>
        </div>

        <!-- Map -->
        <div class="w-1/2 h-full p-2 border-l border-gray-300">
            <div class="overflow-hidden rounded-lg h-full">
                <LMap ref="map" :center="[55, -3]" :zoom="6" :minZoom="5" @ready="onMapReady">
                    <LTileLayer
                        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
                        attribution='&amp;copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors'
                        layer-type="base"
                        name="OpenStreetMap"
                    />

                    <LGeoJson v-if="parsedGeoJson" :geojson="parsedGeoJson" :options-style="geoStyler" />
                </LMap>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from "vue";
import { geojsonToWkt, wktToGeojson } from "wkt-parser-helper";
import { LMap, LTileLayer, LGeoJson } from "@maxel01/vue-leaflet";
import type { Feature, Geometry } from "geojson";

const map = ref(null);
const onMapReady = () => {
    map.value.leafletObject.invalidateSize();
};
const geoStyler = (feature) => ({
    opacity: feature.properties.code / 100000,
});

const inputField = ref("");

const parsedGeoJson = ref<Feature | Geometry | null>(null);
const stringifiedGeoJson = computed(() => {
    if (!parsedGeoJson.value) return null;
    return JSON.stringify(parsedGeoJson.value, null, 2);
});
const parsedWkt = computed(() => {
    if (!parsedGeoJson.value) return null;
    return geojsonToWkt(parsedGeoJson.value);
});

function tryParseGeojson(input: string): Feature | Geometry | null {
    try {
        const parsedInput = JSON.parse(input);
        parsedGeoJson.value = parsedInput;
        return parsedInput;
    } catch (error) {
        isValid.value = false;
        return null;
    }
}

function tryParseWkt(input: string): Feature | Geometry | null {
    try {
        const geoJsonConversion = wktToGeojson(input);
        parsedGeoJson.value = geoJsonConversion;
        return geoJsonConversion;
    } catch (error) {
        isValid.value = false;
        return null;
    }
}

watch(inputField, () => {
    if (inputField.value === "") {
        parsedGeoJson.value = null;
        return;
    }

    tryParseGeojson(inputField.value);
    tryParseWkt(inputField.value);
});

const isValid = computed(() => parsedGeoJson.value);

function downloadFile(content: string, extension: string) {
    const blob = new Blob([content], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `geo.${extension}`;
    a.click();
    URL.revokeObjectURL(url);
}
function downloadGeojson() {
    downloadFile(stringifiedGeoJson.value, "geojson");
}
function downloadWkt() {
    downloadFile(parsedWkt.value, "wkt");
}

function copyToClipboard(content: string) {
    if (!content) return;
    navigator.clipboard.writeText(content);
}
</script>
