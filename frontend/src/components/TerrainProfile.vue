<script setup lang="ts">
import { ref, onMounted, watch, nextTick, computed } from 'vue';
import { useDroneStore } from '../store/drone';

const store = useDroneStore();
const canvas = ref<HTMLCanvasElement>();

const alertSummary = computed(() => {
  const dangers = store.altitudeIssues.filter((i) => i.level === 'danger').length;
  const warnings = store.altitudeIssues.filter((i) => i.level === 'warning').length;
  return { dangers, warnings };
});

function draw() {
  const ctx = canvas.value?.getContext('2d');
  if (!ctx) return;

  const W = 800;
  const H = 280;
  const padding = { top: 40, right: 20, bottom: 40, left: 55 };
  const plotW = W - padding.left - padding.right;
  const plotH = H - padding.top - padding.bottom;

  ctx.clearRect(0, 0, W, H);

  ctx.fillStyle = '#1e293b';
  ctx.fillRect(0, 0, W, H);

  const profile = store.terrainProfile;
  if (profile.length < 2) {
    ctx.fillStyle = '#94a3b8';
    ctx.font = '14px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('暂无航线数据 — 规划航线后显示地形剖面', W / 2, H / 2);
    return;
  }

  const distances: number[] = [0];
  for (let i = 1; i < profile.length; i++) {
    const dx = (profile[i].lng - profile[i - 1].lng) * 111000;
    const dy = (profile[i].lat - profile[i - 1].lat) * 111000;
    distances.push(distances[i - 1] + Math.sqrt(dx * dx + dy * dy));
  }
  const maxDist = distances[distances.length - 1] || 1;

  let minAlt = Infinity;
  let maxAlt = -Infinity;
  for (const p of profile) {
    minAlt = Math.min(minAlt, p.terrainElevation);
    maxAlt = Math.max(maxAlt, Math.max(p.altitude, p.safeLine));
  }
  minAlt = Math.max(0, minAlt - 20);
  maxAlt = maxAlt + 20;
  const altRange = maxAlt - minAlt || 1;

  const toX = (d: number) => padding.left + (d / maxDist) * plotW;
  const toY = (a: number) => padding.top + plotH - ((a - minAlt) / altRange) * plotH;

  ctx.save();
  ctx.beginPath();
  ctx.rect(padding.left, padding.top, plotW, plotH);
  ctx.clip();

  const hazardSegs = store.hazardSegments;
  for (const seg of hazardSegs) {
    const startX = toX(distances[seg.startIndex]);
    const endX = toX(distances[seg.endIndex]);
    const segW = Math.max(endX - startX, 2);
    const color = seg.maxLevel === 'danger'
      ? 'rgba(239, 68, 68, 0.18)'
      : 'rgba(251, 191, 36, 0.18)';
    ctx.fillStyle = color;
    ctx.fillRect(startX, padding.top, segW, plotH);

    const topColor = seg.maxLevel === 'danger'
      ? 'rgba(239, 68, 68, 0.35)'
      : 'rgba(251, 191, 36, 0.35)';
    ctx.fillStyle = topColor;
    ctx.fillRect(startX, padding.top, segW, 3);
  }

  ctx.beginPath();
  ctx.moveTo(toX(distances[0]), toY(profile[0].terrainElevation));
  for (let i = 1; i < profile.length; i++) {
    ctx.lineTo(toX(distances[i]), toY(profile[i].terrainElevation));
  }
  ctx.lineTo(toX(distances[distances.length - 1]), padding.top + plotH);
  ctx.lineTo(toX(0), padding.top + plotH);
  ctx.closePath();
  ctx.fillStyle = 'rgba(139,92,46,0.6)';
  ctx.fill();
  ctx.strokeStyle = '#92613a';
  ctx.lineWidth = 1;
  ctx.stroke();

  ctx.beginPath();
  ctx.moveTo(toX(distances[0]), toY(profile[0].safeLine));
  for (let i = 1; i < profile.length; i++) {
    ctx.lineTo(toX(distances[i]), toY(profile[i].safeLine));
  }
  ctx.strokeStyle = '#22c55e';
  ctx.lineWidth = 1.5;
  ctx.setLineDash([6, 4]);
  ctx.stroke();
  ctx.setLineDash([]);

  for (let i = 0; i < profile.length; i++) {
    const p = profile[i];
    if (p.level === 'safe') continue;
    const x = toX(distances[i]);
    const terrainY = toY(p.terrainElevation);
    const safeY = toY(p.safeLine);
    const altitudeY = toY(p.altitude);

    const topY = Math.min(altitudeY, safeY);
    const bottomY = terrainY;
    const fillH = Math.abs(bottomY - topY);
    if (fillH > 0) {
      ctx.fillStyle = p.level === 'danger'
        ? 'rgba(239, 68, 68, 0.45)'
        : 'rgba(251, 191, 36, 0.45)';
      ctx.fillRect(x - 1.5, topY, 3, fillH);
    }
  }

  ctx.beginPath();
  ctx.moveTo(toX(distances[0]), toY(profile[0].terrainElevation));
  for (let i = 1; i < profile.length; i++) {
    ctx.lineTo(toX(distances[i]), toY(profile[i].terrainElevation));
  }
  for (let i = profile.length - 1; i >= 0; i--) {
    ctx.lineTo(toX(distances[i]), toY(profile[i].altitude));
  }
  ctx.closePath();
  ctx.fillStyle = 'rgba(34,197,94,0.08)';
  ctx.fill();

  for (let i = 0; i < profile.length - 1; i++) {
    const p1 = profile[i];
    const p2 = profile[i + 1];
    const segmentLevel = p1.level === 'danger' || p2.level === 'danger'
      ? 'danger'
      : p1.level === 'warning' || p2.level === 'warning'
      ? 'warning'
      : 'safe';

    let strokeColor = '#3b82f6';
    let lineWidth = 2;
    let dash: number[] | undefined = undefined;

    if (segmentLevel === 'danger') {
      strokeColor = '#ef4444';
      lineWidth = 3;
      dash = [8, 4];
    } else if (segmentLevel === 'warning') {
      strokeColor = '#f59e0b';
      lineWidth = 2.5;
    }

    ctx.beginPath();
    ctx.moveTo(toX(distances[i]), toY(p1.altitude));
    ctx.lineTo(toX(distances[i + 1]), toY(p2.altitude));
    ctx.strokeStyle = strokeColor;
    ctx.lineWidth = lineWidth;
    if (dash) ctx.setLineDash(dash);
    ctx.stroke();
    ctx.setLineDash([]);
  }

  for (let i = 0; i < profile.length; i++) {
    const p = profile[i];
    const x = toX(distances[i]);
    const y = toY(p.altitude);
    let fillColor = '#60a5fa';
    let strokeColor = '#1d4ed8';
    let radius = 4;

    if (p.level === 'danger') {
      fillColor = '#ef4444';
      strokeColor = '#991b1b';
      radius = 6;
    } else if (p.level === 'warning') {
      fillColor = '#f59e0b';
      strokeColor = '#92400e';
      radius = 5;
    }

    ctx.beginPath();
    ctx.arc(x, y, radius, 0, Math.PI * 2);
    ctx.fillStyle = fillColor;
    ctx.fill();
    ctx.strokeStyle = strokeColor;
    ctx.lineWidth = 1.5;
    ctx.stroke();

    if (p.level !== 'safe') {
      ctx.beginPath();
      ctx.moveTo(x, y - radius - 3);
      ctx.lineTo(x - 4, y - radius - 9);
      ctx.lineTo(x + 4, y - radius - 9);
      ctx.closePath();
      ctx.fillStyle = fillColor;
      ctx.fill();
      ctx.strokeStyle = strokeColor;
      ctx.lineWidth = 1;
      ctx.stroke();

      ctx.fillStyle = '#fff';
      ctx.font = 'bold 9px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('!', x, y - radius - 2);
    }
  }

  ctx.restore();

  ctx.strokeStyle = '#475569';
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(padding.left, padding.top);
  ctx.lineTo(padding.left, padding.top + plotH);
  ctx.lineTo(padding.left + plotW, padding.top + plotH);
  ctx.stroke();

  ctx.fillStyle = '#94a3b8';
  ctx.font = '11px sans-serif';
  ctx.textAlign = 'center';
  const xTicks = 5;
  for (let i = 0; i <= xTicks; i++) {
    const d = (maxDist / xTicks) * i;
    const x = toX(d);
    ctx.fillText(`${(d / 1000).toFixed(1)} km`, x, padding.top + plotH + 16);
    ctx.beginPath();
    ctx.moveTo(x, padding.top + plotH);
    ctx.lineTo(x, padding.top + plotH + 4);
    ctx.strokeStyle = '#475569';
    ctx.stroke();
  }

  ctx.textAlign = 'right';
  const yTicks = 5;
  for (let i = 0; i <= yTicks; i++) {
    const a = minAlt + (altRange / yTicks) * i;
    const y = toY(a);
    ctx.fillText(`${Math.round(a)}m`, padding.left - 6, y + 4);
    ctx.beginPath();
    ctx.moveTo(padding.left - 3, y);
    ctx.lineTo(padding.left, y);
    ctx.strokeStyle = '#475569';
    ctx.stroke();
  }

  ctx.textAlign = 'left';
  let legendX = padding.left + 8;
  const legendY = 8;

  ctx.fillStyle = '#3b82f6';
  ctx.fillRect(legendX, legendY, 14, 3);
  ctx.fillStyle = '#94a3b8';
  ctx.font = '11px sans-serif';
  ctx.fillText('飞行高度', legendX + 18, legendY + 6);
  legendX += 82;

  ctx.fillStyle = 'rgba(139,92,46,0.8)';
  ctx.fillRect(legendX, legendY - 2, 14, 9);
  ctx.fillStyle = '#94a3b8';
  ctx.fillText('地形', legendX + 18, legendY + 6);
  legendX += 50;

  ctx.strokeStyle = '#22c55e';
  ctx.lineWidth = 1.5;
  ctx.setLineDash([5, 3]);
  ctx.beginPath();
  ctx.moveTo(legendX, legendY + 1.5);
  ctx.lineTo(legendX + 14, legendY + 1.5);
  ctx.stroke();
  ctx.setLineDash([]);
  ctx.fillStyle = '#94a3b8';
  ctx.fillText('安全线', legendX + 18, legendY + 6);
  legendX += 56;

  if (alertSummary.value.warnings > 0) {
    ctx.fillStyle = '#f59e0b';
    ctx.fillRect(legendX, legendY - 1, 12, 6);
    ctx.fillStyle = '#94a3b8';
    ctx.fillText(`警告 ${alertSummary.value.warnings}`, legendX + 16, legendY + 6);
    legendX += 74;
  }

  if (alertSummary.value.dangers > 0) {
    ctx.fillStyle = '#ef4444';
    ctx.fillRect(legendX, legendY - 1, 12, 6);
    ctx.fillStyle = '#94a3b8';
    ctx.fillText(`危险 ${alertSummary.value.dangers}`, legendX + 16, legendY + 6);
  }
}

onMounted(() => nextTick(draw));
watch(
  () => [store.terrainProfile, store.hazardSegments, store.altitudeIssues],
  () => nextTick(draw),
  { deep: true }
);
</script>

<template>
  <div class="flex flex-col gap-2">
    <canvas
      ref="canvas"
      width="800"
      height="280"
      class="w-full rounded-lg border border-slate-700"
    />
    <div v-if="store.altitudeIssues.length > 0" class="space-y-1.5">
      <div
        v-for="issue in store.altitudeIssues.slice(0, 5)"
        :key="`issue-${issue.index}`"
        :class="[
          'px-3 py-2 rounded text-xs flex items-start gap-2 border',
          issue.level === 'danger'
            ? 'bg-red-900/30 border-red-700/60 text-red-200'
            : 'bg-amber-900/30 border-amber-700/60 text-amber-200'
        ]"
      >
        <span
          :class="[
            'inline-block w-4 h-4 rounded-full text-center leading-4 flex-shrink-0 text-[10px] font-bold',
            issue.level === 'danger' ? 'bg-red-600 text-white' : 'bg-amber-600 text-white'
          ]"
        >!</span>
        <div class="flex-1 min-w-0">
          <div class="font-medium">
            航点 {{ issue.index + 1 }} ·
            净高 <span class="font-mono">{{ issue.clearance.toFixed(1) }}m</span>
            <span class="text-slate-400"> (安全 {{ issue.safeDistance }}m)</span>
          </div>
          <div class="opacity-80 mt-0.5">{{ issue.message }}</div>
        </div>
      </div>
      <div
        v-if="store.altitudeIssues.length > 5"
        class="text-xs text-slate-400 pl-1"
      >
        ...另有 {{ store.altitudeIssues.length - 5 }} 处高度差问题
      </div>
    </div>
    <div
      v-else-if="store.terrainProfile.length >= 2"
      class="px-3 py-2 rounded text-xs bg-green-900/20 border border-green-700/50 text-green-300 flex items-center gap-2"
    >
      <span class="w-4 h-4 rounded-full bg-green-600 text-white text-center leading-4 text-[10px] font-bold">✓</span>
      所有航点满足安全距离要求（{{ store.droneConfig.safeDistance }}m）
    </div>
  </div>
</template>
