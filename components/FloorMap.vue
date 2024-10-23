<template>
  <div class="floor-map-container">
    <div class="button-container">
      <button
        v-for="marker in markers"
        :key="marker.label"
        class="button"
        :class="{ 'button--is-active': marker.label === selectedMarkerLabel }"
        @click="handleMarkerClick(marker.label)"
      >
        {{ marker.label }}
      </button>
    </div>
    <div ref="mapContainer">
      <div v-if="!isMapInitialized">Loading...</div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type {
  Map,
  LayerGroup,
  ImageOverlay,
  MapOptions,
  DivIcon,
  LatLng,
} from "leaflet";
import { ref, watch, onMounted, onBeforeUnmount } from "vue";

type Marker = {
  id: number;
  x: number;
  y: number;
  label: string;
};

type FloorMapProps = {
  floorPlanUrl: string;
  markers: Marker[];
};

type MapBounds = {
  bottomLeft: LatLng;
  topRight: LatLng;
};

const props = withDefaults(defineProps<FloorMapProps>(), {
  markers: () => [],
  floorPlanUrl: "",
});

const mapContainer = ref<HTMLElement | null>(null);
const map = ref<Map | null>(null);
const markerLayer = ref<LayerGroup | null>(null);
const imageOverlay = ref<ImageOverlay | null>(null);
const isMapInitialized = ref<boolean>(false);
const L = ref<any>(null);
const imageHeight = ref<number>(0);
const selectedMarkerLabel = ref<string>("");
const markersRef = ref<{ [key: string]: L.Marker }>({});
const defaultBounds = ref<L.LatLngBounds | null>(null);

const router = useRouter();
const route = useRoute();

const setQueryParam = (name: string): void => {
  router.push({
    path: route.path,
    query: name ? { name } : undefined,
  });
};

const setStateFromQueryParam = (): void => {
  const { name } = route.query;
  if (!name) return

  selectedMarkerLabel.value = name as string;

  if (map.value) {
    const selectedMarker = markersRef.value[name as string];
    map.value.setView(selectedMarker.getLatLng(), 1, {
      animate: true,
      duration: 0.5,
    });
  }
};

const createMarkerIcon = (label: string, isSelected: boolean): DivIcon => {
  if (!L.value) return {} as DivIcon;

  const selectedClass = isSelected ? "selected" : "";

  return L.value.divIcon({
    className: `custom-marker-icon ${selectedClass}`,
    html: `
      <div class="custom-marker-container ${selectedClass}">
        <div class="custom-marker-dot ${selectedClass}"></div>
        <div class="custom-marker-tooltip ${selectedClass}">${label}</div>
      </div>
    `,
  });
};

const resetMapView = (): void => {
  if (map.value && defaultBounds.value) {
    map.value.fitBounds(defaultBounds.value, {
      animate: true,
      duration: 0.5,
    });
  }
};

const handleMarkerClick = (label: string) => {
  if (selectedMarkerLabel.value === label) {
    selectedMarkerLabel.value = "";
    resetMapView();
  } else {
    selectedMarkerLabel.value = label;
    const selectedMarker = markersRef.value[label];
    if (selectedMarker && map.value) {
      map.value.setView(selectedMarker.getLatLng(), 1, {
        animate: true,
        duration: 0.5,
      });
    }
  }

  setQueryParam(selectedMarkerLabel.value);

  updateMarkers();
};

const calculateBounds = (width: number, height: number, map: Map): MapBounds => {
  const bottomLeft = map.unproject([0, height], 0);
  const topRight = map.unproject([width, 0], 0);
  return { bottomLeft, topRight };
};

const transformCoordinates = (x: number, y: number): [number, number] => {
  return [x, imageHeight.value - y];
};

const updateMarkers = (): void => {
  if (!map.value || !markerLayer.value || !L.value) return;

  markerLayer.value.clearLayers();
  markersRef.value = {};

  props.markers.forEach((marker: Marker) => {
    if (!map.value) return;

    const [transformedX, transformedY] = transformCoordinates(
      marker.x,
      marker.y
    );
    const point = map.value.unproject([transformedX, transformedY], 0);
    const isSelected = marker.label === selectedMarkerLabel.value;
    const markerIcon = createMarkerIcon(marker.label, isSelected);

    const mapMarker = L.value
      .marker(point, { icon: markerIcon })
      .addTo(markerLayer.value)
      .on("click", () => handleMarkerClick(marker.label));

    markersRef.value[marker.label] = mapMarker;
  });
};

const initializeMap = async (): Promise<void> => {
  if (!mapContainer.value) return;

  try {
    const leaflet = await import("leaflet");
    L.value = leaflet.default;
    await import("leaflet/dist/leaflet.css");

    const mapOptions: MapOptions = {
      crs: L.value.CRS.Simple,
      minZoom: -0.6,
      maxZoom: 1,
      zoomAnimation: true,
      zoomAnimationThreshold: 0,
      scrollWheelZoom: true,
      wheelDebounceTime: 0,
      wheelPxPerZoomLevel: 50,
      inertia: true,
      inertiaDeceleration: 6000,
      zoomSnap: 0,
      zoomControl: true,
      maxBoundsViscosity: 1.0,
      bounceAtZoomLimits: false,
    };

    map.value = L.value.map(mapContainer.value, mapOptions);

    if (map.value === null) {
      throw new Error("Map not initialized");
    }

    const img = new Image();
    img.src = props.floorPlanUrl;

    await new Promise<void>((resolve, reject) => {
      img.onload = () => resolve();
      img.onerror = () => reject(new Error("Failed to load floor plan"));
    });

    imageHeight.value = img.height;

    const { bottomLeft, topRight } = calculateBounds(img.width, img.height, map.value as Map);
    const bounds = new L.value.LatLngBounds(bottomLeft, topRight);

    defaultBounds.value = bounds;

    const padding = 10;
    const maxBounds = bounds.pad(padding / Math.max(img.width, img.height));
    map.value.setMaxBounds(maxBounds);

    imageOverlay.value = L.value
      .imageOverlay(props.floorPlanUrl, bounds)
      .addTo(map.value);
    markerLayer.value = L.value.layerGroup().addTo(map.value);
    map.value.fitBounds(bounds);

    updateMarkers();
    isMapInitialized.value = true;
  } catch (error) {
    console.error("Error initializing map:", error);
    throw error;
  }
};

watch(() => selectedMarkerLabel.value, updateMarkers, {
  deep: true,
});

onMounted(() => {
  initializeMap()
    .then(() => setStateFromQueryParam())
    .catch((error) => console.error("Failed to initialize map:", error));
});

</script>

<style>
.floor-map-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 100%;
  height: 100%;
}

.leaflet-container {
  height: 100%;
  width: 100%;
}

.leaflet-control-attribution {
  display: none;
}

.custom-marker-icon {
  background: transparent;
  border: none;
}

.custom-marker-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.custom-marker-dot {
  position: absolute;
  width: 16px;
  height: 16px;
  background-color: #3b82f6;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  left: 50%;
  top: 50%;
  transition: all 0.2s ease;
}

.custom-marker-dot.selected {
  background-color: #ef4444;
  box-shadow: 0 0 0 4px rgba(239, 68, 68, 0.2);
  width: 20px;
  height: 20px;
}

.custom-marker-tooltip {
  position: absolute;
  background-color: white;
  padding: 4px 8px;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  font-size: 0.875rem;
  white-space: nowrap;
  transform: translate(-50%, -100%);
  left: 50%;
  top: -8px;
  z-index: 1000;
  transition: all 0.2s ease;
}

.custom-marker-tooltip.selected {
  background-color: #ef4444;
  color: white;
  box-shadow: 0 2px 4px rgba(239, 68, 68, 0.2);
}

.custom-marker-container:hover .custom-marker-dot:not(.selected) {
  background-color: #2563eb;
  transform: translate(-50%, -50%) scale(1.1);
}

.custom-marker-container:hover .custom-marker-tooltip:not(.selected) {
  transform: translate(-50%, -100%) scale(1.05);
}

.button {
  background-color: #3b82f6;
  border: 1px solid #3b82f6;
  color: white;
  padding: 8px 16px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 14px;
  margin: 4px 2px;
  cursor: pointer;
  border-radius: 8px;
  transition-duration: 0.4s;
}

.button:hover {
  background-color: white;
  color: black;
  border: 1px solid #3b82f6;
}

.button--is-active {
  background-color: #ef4444;
  border: 1px solid #ef4444;
  color: white;
}
</style>
