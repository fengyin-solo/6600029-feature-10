<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch, nextTick, computed } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import { useDroneStore } from '../store/drone';
import type { AltitudeAlertLevel } from '../types';

const store = useDroneStore();
const mapContainer = ref<HTMLElement>();
let map: L.Map | null = null;
let waypointLayer: L.LayerGroup | null = null;
let routeSegmentsLayer: L.LayerGroup | null = null;
let zoneLayer: L.LayerGroup | null = null;
let hazardMarkerLayer: L.LayerGroup | null = null;
let droneMarker: L.CircleMarker | null = null;

const addMode = ref(false);

const overallRouteStatus = computed<AltitudeAlertLevel>(() => {
  if (store.hasAltitudeDanger) return 'danger';
  if (store.hasAltitudeWarning) return 'warning';
  return 'safe';
});

function initMap() {
  if (!mapContainer.value || map) return;
  map = L.map(mapContainer.value).setView(store.mapCenter, 12);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap',
    maxZoom: 18,
  }).addTo(map);

  waypointLayer = L.layerGroup().addTo(map);
  zoneLayer = L.layerGroup().addTo(map);
  routeSegmentsLayer = L.layerGroup().addTo(map);
  hazardMarkerLayer = L.layerGroup().addTo(map);

  map.on('click', (e: L.LeafletMouseEvent) => {
    if (addMode.value) {
      store.addWaypoint(e.latlng.lat, e.latlng.lng);
    }
  });
}

function drawNoFlyZones() {
  if (!zoneLayer) return;
  zoneLayer.clearLayers();
  for (const zone of store.noFlyZones) {
    const color =
      zone.type === 'airport' ? '#ef4444' :
      zone.type === 'military' ? '#f97316' : '#a855f7';
    L.circle([zone.center[0], zone.center[1]], {
      radius: zone.radius,
      color,
      fillColor: color,
      fillOpacity: 0.15,
      weight: 2,
    })
      .bindPopup(`<b>${zone.name}</b><br>Type: ${zone.type}<br>Radius: ${zone.radius}m`)
      .addTo(zoneLayer);
  }
}

function drawWaypoints() {
  if (!waypointLayer) return;
  waypointLayer.clearLayers();
  const profile = store.terrainProfile;
  store.waypoints.forEach((wp, idx) => {
    const point = profile[idx];
    const level: AltitudeAlertLevel = point?.level ?? 'safe';

    let color = '#3b82f6';
    let fillColor = '#60a5fa';
    let radius = 8;

    if (level === 'danger') {
      color = '#991b1b';
      fillColor = '#ef4444';
      radius = 10;
    } else if (level === 'warning') {
      color = '#92400e';
      fillColor = '#f59e0b';
      radius = 9;
    }

    const marker = L.circleMarker([wp.lat, wp.lng], {
      radius,
      color,
      fillColor,
      fillOpacity: 0.92,
      weight: 2,
    });

    const tooltipClass = level !== 'safe'
      ? 'wp-tooltip wp-tooltip-alert'
      : 'wp-tooltip';
    marker.bindTooltip(`WP${idx + 1}`, {
      permanent: true,
      direction: 'top',
      className: tooltipClass,
      offset: [0, -radius - 2],
    });

    let popupHtml = `<div style="min-width:180px">
      <b>Waypoint ${idx + 1}</b><br>
      Altitude: ${wp.altitude}m<br>
      Speed: ${wp.speed} m/s<br>
      Action: ${wp.action}<br>`;

    if (point) {
      const badgeClass = level === 'danger'
        ? 'background:#ef4444;color:#fff;padding:1px 6px;border-radius:3px;font-size:11px'
        : level === 'warning'
        ? 'background:#f59e0b;color:#fff;padding:1px 6px;border-radius:3px;font-size:11px'
        : 'background:#22c55e;color:#fff;padding:1px 6px;border-radius:3px;font-size:11px';
      const badgeText = level === 'danger'
        ? '危险'
        : level === 'warning'
        ? '警告'
        : '安全';
      popupHtml += `<hr style="border:none;border-top:1px solid #334155;margin:6px 0">
        地形高度: ${point.terrainElevation.toFixed(1)}m<br>
        净高: <b>${point.clearance.toFixed(1)}m</b><br>
        <span style="${badgeClass}">${badgeText}</span>`;
    }

    popupHtml += `<hr style="border:none;border-top:1px solid #334155;margin:6px 0">
      <button onclick="this.closest('.leaflet-popup').remove()" style="margin-top:4px;color:#ef4444;background:none;border:none;cursor:pointer;padding:0">Remove</button>
    </div>`;
    marker.bindPopup(popupHtml);

    marker.on('dragend', (e: any) => {
      const ll = e.target.getLatLng();
      store.updateWaypoint(wp.id, { lat: ll.lat, lng: ll.lng });
    });
    marker.addTo(waypointLayer!);
  });
}

function getSegmentLevel(i: number): AltitudeAlertLevel {
  const profile = store.terrainProfile;
  if (!profile[i] || !profile[i + 1]) {
    for (const wp of [store.waypoints[i], store.waypoints[i + 1]]) {
      if (!wp) continue;
      for (const zone of store.noFlyZones) {
        const d = Math.sqrt(
          (wp.lat - zone.center[0]) ** 2 + (wp.lng - zone.center[1]) ** 2
        ) * 111000;
        if (d < zone.radius * 1.5) return 'danger';
      }
    }
    return 'safe';
  }
  const p1 = profile[i];
  const p2 = profile[i + 1];
  if (p1.level === 'danger' || p2.level === 'danger') return 'danger';
  if (p1.level === 'warning' || p2.level === 'warning') return 'warning';
  for (const wp of [store.waypoints[i], store.waypoints[i + 1]]) {
    for (const zone of store.noFlyZones) {
      const d = Math.sqrt(
        (wp.lat - zone.center[0]) ** 2 + (wp.lng - zone.center[1]) ** 2
      ) * 111000;
      if (d < zone.radius * 1.5) return 'danger';
    }
  }
  return 'safe';
}

function drawRouteSegments() {
  if (!routeSegmentsLayer) return;
  routeSegmentsLayer.clearLayers();
  if (store.waypoints.length < 2 || !map) return;

  for (let i = 0; i < store.waypoints.length - 1; i++) {
    const wp1 = store.waypoints[i];
    const wp2 = store.waypoints[i + 1];
    const latlngs: [number, number][] = [
      [wp1.lat, wp1.lng],
      [wp2.lat, wp2.lng],
    ];
    const level = getSegmentLevel(i);

    let color = '#22c55e';
    let weight = 3;
    let opacity = 0.85;
    let dashArray: string | undefined = undefined;

    if (level === 'danger') {
      color = '#ef4444';
      weight = 5;
      opacity = 0.9;
      dashArray = '10, 6';
    } else if (level === 'warning') {
      color = '#f59e0b';
      weight = 4;
      opacity = 0.88;
    }

    const polyline = L.polyline(latlngs, {
      color,
      weight,
      opacity,
      dashArray,
      lineJoin: 'round',
    });

    if (level !== 'safe') {
      const segs = store.hazardSegments.filter(
        (s) => s.startIndex <= i && s.endIndex >= i
      );
      const seg = segs[0];
      if (seg) {
        const label = level === 'danger'
          ? `⚠ 危险段 · 最低净高 ${seg.minClearance.toFixed(1)}m`
          : `⚠ 警告段 · 最低净高 ${seg.minClearance.toFixed(1)}m`;
        polyline.bindTooltip(label, {
          sticky: true,
          className: level === 'danger' ? 'seg-tooltip-danger' : 'seg-tooltip-warning',
          direction: 'top',
        });
      }
    }
    polyline.addTo(routeSegmentsLayer);
  }

  if (store.hazardSegments.length > 0 && map) {
    for (const seg of store.hazardSegments) {
      const color = seg.maxLevel === 'danger' ? '#ef4444' : '#f59e0b';
      const bgColor = seg.maxLevel === 'danger'
        ? 'rgba(239, 68, 68, 0.12)'
        : 'rgba(245, 158, 11, 0.12)';
      const midLat = (seg.startLat + seg.endLat) / 2;
      const midLng = (seg.startLng + seg.endLng) / 2;
      const segLen = Math.sqrt(
        ((seg.endLat - seg.startLat) * 111000) ** 2 +
        ((seg.endLng - seg.startLng) * 111000) ** 2
      );
      const radius = Math.max(80, segLen * 0.55);
      L.circle([midLat, midLng], {
        radius,
        color,
        fillColor: color,
        fillOpacity: 0.08,
        weight: 0,
      }).addTo(routeSegmentsLayer);

      L.rectangle(
        [
          [seg.startLat, seg.startLng],
          [seg.endLat, seg.endLng],
        ],
        {
          color,
          fillColor: bgColor,
          fillOpacity: 0,
          weight: 0,
        }
      ).addTo(routeSegmentsLayer);
    }
  }
}

function drawHazardMarkers() {
  if (!hazardMarkerLayer) return;
  hazardMarkerLayer.clearLayers();

  for (const issue of store.altitudeIssues) {
    const color = issue.level === 'danger' ? '#ef4444' : '#f59e0b';
    const icon = L.divIcon({
      className: 'hazard-marker',
      html: `<div style="
        width: 22px; height: 22px;
        background: ${color};
        border: 2px solid #fff;
        border-radius: 50%;
        color: #fff;
        font-weight: 900;
        font-size: 13px;
        line-height: 18px;
        text-align: center;
        box-shadow: 0 2px 6px rgba(0,0,0,0.5);
      ">!</div>`,
      iconSize: [22, 22],
      iconAnchor: [11, 11],
    });
    const marker = L.marker([issue.lat, issue.lng], { icon });
    marker.bindPopup(`
      <div style="min-width:200px">
        <b style="color:${color};font-size:13px">
          ${issue.level === 'danger' ? '⚠ 超高差危险' : '⚠ 超高差警告'}
        </b><br>
        <hr style="border:none;border-top:1px solid #334155;margin:6px 0">
        航点: WP${issue.index + 1}<br>
        飞行高度: ${issue.altitude.toFixed(1)}m<br>
        地形高度: ${issue.terrainElevation.toFixed(1)}m<br>
        <b>净高: ${issue.clearance.toFixed(1)}m</b><br>
        安全距离: ${issue.safeDistance}m<br>
        <hr style="border:none;border-top:1px solid #334155;margin:6px 0">
        <span style="font-size:12px;opacity:0.9">${issue.message}</span>
      </div>
    `);
    marker.addTo(hazardMarkerLayer);
  }
}

function drawSimDrone() {
  if (!map || store.waypoints.length < 2) return;
  const progress = store.simProgress / 100;
  const totalWp = store.waypoints.length;
  const segIdx = Math.min(Math.floor(progress * (totalWp - 1)), totalWp - 2);
  const segProgress = (progress * (totalWp - 1)) - segIdx;
  const wp1 = store.waypoints[segIdx];
  const wp2 = store.waypoints[segIdx + 1];
  const lat = wp1.lat + (wp2.lat - wp1.lat) * segProgress;
  const lng = wp1.lng + (wp2.lng - wp1.lng) * segProgress;

  if (droneMarker) {
    droneMarker.setLatLng([lat, lng]);
  } else {
    droneMarker = L.circleMarker([lat, lng], {
      radius: 10,
      color: '#fbbf24',
      fillColor: '#f59e0b',
      fillOpacity: 1,
      weight: 3,
    }).addTo(map);
  }
}

watch(
  () => store.waypoints,
  () => {
    nextTick(() => {
      drawWaypoints();
      drawRouteSegments();
      drawHazardMarkers();
    });
  },
  { deep: true }
);

watch(
  () => [store.hazardSegments, store.altitudeIssues, store.terrainProfile],
  () => {
    nextTick(() => {
      drawWaypoints();
      drawRouteSegments();
      drawHazardMarkers();
    });
  },
  { deep: true }
);

watch(() => store.noFlyZones.length, drawNoFlyZones);
watch(() => store.simProgress, drawSimDrone);

onMounted(() => {
  nextTick(initMap);
});

onUnmounted(() => {
  if (map) {
    map.remove();
    map = null;
  }
});

function toggleAddMode() {
  addMode.value = !addMode.value;
}

function handlePlanRoute() {
  if (store.waypoints.length < 2) return;
  const first = store.waypoints[0];
  const last = store.waypoints[store.waypoints.length - 1];
  store.planRoute([first.lat, first.lng], [last.lat, last.lng]);
}
</script>

<template>
  <div class="relative w-full h-full">
    <div ref="mapContainer" class="w-full h-full rounded-lg" />

    <div class="absolute top-2 right-2 z-[1000] flex flex-col gap-1.5">
      <button
        @click="toggleAddMode"
        :class="addMode ? 'bg-blue-600 text-white' : 'bg-gray-800 text-gray-300'"
        class="px-3 py-1 rounded text-xs font-medium shadow hover:opacity-90 transition"
      >
        {{ addMode ? '✦ 添加模式' : '○ 点击添加' }}
      </button>
      <button
        @click="handlePlanRoute"
        class="px-3 py-1 rounded text-xs font-medium bg-green-700 text-white shadow hover:opacity-90 transition"
      >
        规划航线
      </button>
      <button
        @click="store.clearRoute()"
        class="px-3 py-1 rounded text-xs font-medium bg-red-700 text-white shadow hover:opacity-90 transition"
      >
        清除
      </button>
    </div>

    <div
      v-if="overallRouteStatus !== 'safe' && store.waypoints.length >= 2"
      :class="[
        'absolute top-2 left-2 z-[1000] px-3 py-1.5 rounded text-xs font-medium shadow flex items-center gap-1.5 max-w-[240px]',
        overallRouteStatus === 'danger'
          ? 'bg-red-900/90 border border-red-600 text-red-100'
          : 'bg-amber-900/90 border border-amber-600 text-amber-100'
      ]"
    >
      <span class="text-sm">⚠</span>
      <span class="flex-1 leading-tight">
        航线存在
        {{ overallRouteStatus === 'danger' ? `${store.hazardSegments.filter(s => s.maxLevel === 'danger').length} 个危险段` : `${store.hazardSegments.filter(s => s.maxLevel === 'warning').length} 个警告段` }}
      </span>
    </div>

    <div
      class="absolute bottom-2 left-2 z-[1000] bg-slate-900/90 border border-slate-700 rounded px-2.5 py-1.5 text-[10px] text-slate-300 space-y-1"
    >
      <div class="flex items-center gap-1.5">
        <span class="inline-block w-4 h-1 rounded bg-green-500"></span>
        <span>安全段</span>
      </div>
      <div class="flex items-center gap-1.5">
        <span class="inline-block w-4 h-1 rounded bg-amber-500"></span>
        <span>警告段</span>
      </div>
      <div class="flex items-center gap-1.5">
        <span
          class="inline-block w-4 h-1 rounded bg-red-500"
          style="border-top:1.5px dashed #ef4444;background:repeating-linear-gradient(90deg,#ef4444 0 4px,transparent 4px 7px);height:3px"
        ></span>
        <span>危险段</span>
      </div>
      <div class="flex items-center gap-1.5 pt-0.5 border-t border-slate-700">
        <span class="inline-block w-3 h-3 rounded-full bg-red-500 text-white text-center leading-3 text-[9px] font-bold border border-white">!</span>
        <span>超高差点</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
:deep(.wp-tooltip) {
  background: rgba(30, 41, 59, 0.9);
  color: #e2e8f0;
  border: 1px solid #475569;
  font-size: 10px;
  padding: 1px 4px;
  border-radius: 4px;
}
:deep(.wp-tooltip-alert) {
  background: rgba(239, 68, 68, 0.92);
  color: #fff;
  border: 1px solid #fca5a5;
  font-weight: 700;
  box-shadow: 0 0 0 1px rgba(0, 0, 0, 0.3);
}
:deep(.seg-tooltip-danger) {
  background: rgba(127, 29, 29, 0.95);
  color: #fecaca;
  border: 1px solid #ef4444;
  border-radius: 4px;
  font-size: 11px;
  padding: 4px 8px;
  font-weight: 600;
}
:deep(.seg-tooltip-warning) {
  background: rgba(120, 53, 15, 0.95);
  color: #fde68a;
  border: 1px solid #f59e0b;
  border-radius: 4px;
  font-size: 11px;
  padding: 4px 8px;
  font-weight: 600;
}
</style>
