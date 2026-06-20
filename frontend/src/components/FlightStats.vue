<script setup lang="ts">
import { useDroneStore } from '../store/drone';

const store = useDroneStore();

function formatTime(seconds: number): string {
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60);
  return `${m}:${s.toString().padStart(2, '0')}`;
}

function batteryColor(pct: number): string {
  if (pct < 50) return '#22c55e';
  if (pct < 80) return '#eab308';
  return '#ef4444';
}

function handleExport() {
  const kml = store.exportPlan();
  if (!kml) return;
  const blob = new Blob([kml], { type: 'application/vnd.google-earth.kml+xml' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'flight-plan.kml';
  a.click();
  URL.revokeObjectURL(url);
}
</script>

<template>
  <div class="bg-slate-800 rounded-lg p-4 space-y-3">
    <h3 class="text-sm font-bold text-slate-200 border-b border-slate-700 pb-2">
      飞行统计
    </h3>

    <div class="grid grid-cols-2 gap-2 text-xs">
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">总距离</div>
        <div class="text-lg font-bold text-sky-400">
          {{ (store.totalDistance / 1000).toFixed(2) }}
          <span class="text-xs text-slate-500">km</span>
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">预计时间</div>
        <div class="text-lg font-bold text-sky-400">
          {{ formatTime(store.estimatedTime) }}
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">电量消耗</div>
        <div class="text-lg font-bold" :style="{ color: batteryColor(store.batteryPercent) }">
          {{ store.batteryPercent.toFixed(1) }}%
        </div>
        <div class="w-full bg-slate-700 rounded-full h-1.5 mt-1">
          <div
            class="h-1.5 rounded-full transition-all"
            :style="{ width: store.batteryPercent + '%', backgroundColor: batteryColor(store.batteryPercent) }"
          />
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">航点数量</div>
        <div class="text-lg font-bold text-sky-400">{{ store.waypoints.length }}</div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">最大高度</div>
        <div class="text-lg font-bold text-sky-400">
          {{ store.waypoints.length > 0 ? Math.max(...store.waypoints.map((w) => w.altitude)) : 0 }}
          <span class="text-xs text-slate-500">m</span>
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">算法</div>
        <div class="text-lg font-bold text-purple-400 uppercase">
          {{ store.selectedAlgorithm }}
        </div>
      </div>
    </div>

    <div
      v-if="store.terrainProfile.length >= 2"
      class="bg-slate-900 rounded p-2 space-y-1.5"
    >
      <div class="text-slate-400 text-[11px] mb-1">地形安全检查 (安全距离 {{ store.droneConfig.safeDistance }}m)</div>

      <div class="flex items-center justify-between text-[11px]">
        <span class="text-slate-400">检查航点</span>
        <span class="text-slate-200 font-mono">{{ store.terrainProfile.length }}</span>
      </div>

      <div class="flex items-center justify-between text-[11px]">
        <span class="text-slate-400">✓ 安全</span>
        <span class="text-green-400 font-mono">
          {{ store.terrainProfile.length - store.altitudeIssues.length }}
        </span>
      </div>

      <div
        v-if="store.hasAltitudeWarning"
        class="flex items-center justify-between text-[11px] py-0.5 px-1 rounded bg-amber-900/30 -mx-1"
      >
        <span class="text-amber-300">⚠ 警告</span>
        <span class="text-amber-400 font-mono font-bold">
          {{ store.altitudeIssues.filter(i => i.level === 'warning').length }}
        </span>
      </div>

      <div
        v-if="store.hasAltitudeDanger"
        class="flex items-center justify-between text-[11px] py-0.5 px-1 rounded bg-red-900/40 -mx-1"
      >
        <span class="text-red-300">✗ 危险</span>
        <span class="text-red-400 font-mono font-bold">
          {{ store.altitudeIssues.filter(i => i.level === 'danger').length }}
        </span>
      </div>

      <div v-if="store.hazardSegments.length > 0" class="pt-1 border-t border-slate-700 mt-1">
        <div class="text-[10px] text-slate-400 mb-1">
          连续问题段: {{ store.hazardSegments.length }}
        </div>
        <div class="space-y-0.5">
          <div
            v-for="seg in store.hazardSegments.slice(0, 3)"
            :key="`seg-${seg.startIndex}-${seg.endIndex}`"
            :class="[
              'text-[10px] px-1.5 py-0.5 rounded flex justify-between',
              seg.maxLevel === 'danger'
                ? 'bg-red-900/40 text-red-300'
                : 'bg-amber-900/40 text-amber-300'
            ]"
          >
            <span>WP{{ seg.startIndex + 1 }}-{{ seg.endIndex + 1 }}</span>
            <span class="font-mono">
              净高 {{ seg.minClearance.toFixed(1) }}m
            </span>
          </div>
          <div
            v-if="store.hazardSegments.length > 3"
            class="text-[10px] text-slate-500 px-1"
          >
            ...另有 {{ store.hazardSegments.length - 3 }} 段
          </div>
        </div>
      </div>
    </div>

    <div class="border-t border-slate-700 pt-2">
      <h4 class="text-xs text-slate-400 mb-1">无人机配置</h4>
      <div class="grid grid-cols-3 gap-1 text-[10px] text-slate-500">
        <div>最大高度: {{ store.droneConfig.maxAltitude }}m</div>
        <div>最大速度: {{ store.droneConfig.maxSpeed }}m/s</div>
        <div>电池: {{ store.droneConfig.batteryCapacity }}mAh</div>
      </div>
    </div>

    <button
      @click="handleExport"
      :disabled="!store.currentPlan"
      class="w-full py-2 rounded text-xs font-medium bg-indigo-700 text-white hover:bg-indigo-600 disabled:opacity-40 disabled:cursor-not-allowed transition"
    >
      导出 KML
    </button>
  </div>
</template>
