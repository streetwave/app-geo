<template>
    <div class="flex h-screen bg-base-300">
        <div class="w-1/2 h-full flex flex-col gap-10">
            <!-- Input Field -->
            <div class="h-1/4 p-2 flex flex-col gap-1">
                <label class="btn btn-outline">
                    <input type="file" class="hidden" @change="handleFileUpload" />
                    Open from File
                </label>
                <textarea
                    :class="[
                        'textarea w-full h-full rounded-md bg-base-100 resize-none',
                        isValid === true ? 'textarea-success' : isValid === false ? 'textarea-error' : '',
                    ]"
                    v-model="inputField"
                    placeholder="Enter GeoJSON or WKT..."
                ></textarea>
            </div>

            <!-- Parsed GeoJSON and WKT -->
            <div class="flex h-2/3">
                <div class="flex-1 h-5/6 p-2 flex flex-col gap-2">
                    <h2 class="text-lg font-bold">GeoJSON</h2>
                    <div class="relative h-full">
                        <button
                            v-if="parsedGeoJson"
                            @click="copyToClipboard(stringifiedGeoJson)"
                            class="btn btn-neutral btn-outline absolute top-3 right-5 z-10"
                        >
                            Copy
                        </button>
                        <vue-json-pretty
                            class="w-full h-full border border-gray-300 bg-base-100 rounded-md p-2 overflow-y-auto"
                            :data="parsedGeoJson"
                        />
                    </div>
                    <button @click="downloadGeojson" :disabled="!isValid" class="btn btn-primary btn-outline w-full">Download</button>
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
                    <button @click="downloadWkt" :disabled="!isValid" class="btn btn-primary btn-outline w-full">Download</button>
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
import VueJsonPretty from "vue-json-pretty";
import "vue-json-pretty/lib/styles.css";

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
        return parsedInput;
    } catch (error) {
        parsedGeoJson.value = null;
        return null;
    }
}

function tryParseWkt(input: string): Feature | Geometry | null {
    try {
        const geoJsonConversion = wktToGeojson(input);
        return geoJsonConversion;
    } catch (error) {
        parsedGeoJson.value = null;
        return null;
    }
}

watch(inputField, () => {
    if (inputField.value === "") {
        parsedGeoJson.value = null;
        return;
    }

    const resultGeojson = tryParseGeojson(inputField.value);
    const resultWkt = tryParseWkt(inputField.value);
    if (resultGeojson || resultWkt) {
        parsedGeoJson.value = resultGeojson || resultWkt;
    } else {
        parsedGeoJson.value = null;
    }
});

const isValid = computed(() => {
    if (inputField.value === "") {
        return null;
    }
    return parsedGeoJson.value !== null;
});

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

function handleFileUpload(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
            inputField.value = e.target?.result as string;
        };
        reader.readAsText(file);
    }
}
</script>
