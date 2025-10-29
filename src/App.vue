<script>
  /* eslint-disable no-undef */
  import { setOptions, importLibrary } from "@googlemaps/js-api-loader";
  import { computed, ref, onMounted, onUnmounted, watch } from "vue";
  import { useGeolocation } from "./useGeolocation";

  const GOOGLE_MAPS_API_KEY = "AIzaSyAxVyrdtdgsG2jguXbYNLBMyx6MyT9-CSU";
  setOptions({ key: GOOGLE_MAPS_API_KEY });

  export default {
    name: "App",
    setup() {
      const { coords } = useGeolocation();
      const currPos = computed(() => ({ lat: coords.value.latitude, lng: coords.value.longitude }));
      const otherPos = ref(null);
      
      const mapDiv = ref(null);
      let map = ref(null);
      let clickListener = null;

      onMounted(async () => {
        await importLibrary("maps");

        map.value = new google.maps.Map(mapDiv.value, { center: currPos.value, zoom: 7 });
        clickListener = map.value.addListener("click", ({ latLng: { lat, lng } }) => otherPos.value = { lat: lat(), lng: lng() });
      });
      onUnmounted(async () => {
        if (clickListener) clickListener.remove();
      });

      let line = null;
      watch([map, currPos, otherPos], () => {
        if (line) line.setMap(null);
        if (map.value && otherPos.value != null) line = new google.maps.Polyline({ path: [currPos.value, otherPos.value], map: map.value });
      });

      const haversineDistance = (pos1, pos2) => {
        const R = 3958.8; // Radius of the Earth in miles
        const rlat1 = pos1.lat * (Math.PI / 180); // Convert degrees to radians
        const rlat2 = pos2.lat * (Math.PI / 180); // Convert degrees to radians
        const difflat = rlat2 - rlat1; // Radian difference (latitudes)
        const difflon = (pos2.lng - pos1.lng) * (Math.PI / 180); // Radian difference (longitudes)

        const d = 2 * R * Math.asin(
          Math.sqrt(Math.sin(difflat / 2) * Math.sin(difflat / 2) + Math.cos(rlat1) * Math.cos(rlat2) * Math.sin(difflon / 2) * Math.sin(difflon / 2))
        );

        return d;
      };

      const distance = computed(() => otherPos.value === null ? 0 : haversineDistance(currPos.value, otherPos.value));

      return { currPos, otherPos, distance, mapDiv };
    },
  };
</script>

<template>
  <div class="w-100 row text-center py-1" style="height: 25vh">
    <div class="m-auto col-sm col-md">
      <h4>Your Position</h4>
      Latitude: {{ currPos.lat.toFixed(2) }}, Longitude:
      {{ currPos.lng.toFixed(2) }}
    </div>

    <div class="m-auto col-sm col-md">
      <h4>Distance</h4>
      {{ distance.toFixed(2) }} miles
    </div>

    <div class="m-auto col-sm col-md">
      <h4>Clicked Position</h4>
      <span v-if="otherPos">
        Latitude: {{ otherPos.lat.toFixed(2) }}, Longitude:
        {{ otherPos.lng.toFixed(2) }}
      </span>
      <span v-else>Click the map to select a position</span>
    </div>
  </div>

  <div ref="mapDiv" style="width: 100%; height: 75vh" />
</template>