<template>
    <LMap ref="map" :center="[55, -3]" :zoom="6" :minZoom="5" @ready="onMapReady">
        <LTileLayer
            url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
            attribution='&amp;copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors'
            layer-type="base"
            name="OpenStreetMap"
        />
    </LMap>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { LMap, LTileLayer } from "@maxel01/vue-leaflet";

const map = ref(null);
const onMapReady = () => {
    map.value.leafletObject.invalidateSize();
};
</script>
