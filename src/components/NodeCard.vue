<script setup lang="ts">
import type { NodeData } from '@/stores/nodes'
import { Icon } from '@iconify/vue'
import { computed } from 'vue'
import { Badge } from '@/components/ui/badge'
import { CardX } from '@/components/ui/card-x'
import { DataTooltip } from '@/components/ui/data-tooltip'
import { ProgressThin } from '@/components/ui/progress-thin'
import { useNodePingDisplay } from '@/composables/useNodePingDisplay'
import { useAppStore } from '@/stores/app'
import { formatBytesPerSecondWithConfig, formatBytesWithConfig, formatDateTime, getStatus } from '@/utils/helper'
import { getOSImage, getOSName } from '@/utils/osImageHelper'
import { getRegionCode, getRegionDisplayName } from '@/utils/regionHelper'
import { formatCurrencyValue, getBillingCycleText, getDaysUntilExpired, getExpireStatus, getRemainingValue, parseTags } from '@/utils/tagHelper'

const props = defineProps<{ node: NodeData }>()
const emit = defineEmits<{ click: [] }>()
const appStore = useAppStore()

const formatBytes = (bytes: number) => formatBytesWithConfig(bytes, appStore.byteDecimals)
const formatBytesPerSecond = (bytes: number) => formatBytesPerSecondWithConfig(bytes, appStore.byteDecimals)
const offlineTime = computed(() => formatDateTime(props.node.time))

const cpuStatus = computed(() => getStatus(props.node.cpu ?? 0))
const memPercentage = computed(() => (props.node.ram ?? 0) / (props.node.mem_total || 1) * 100)
const memStatus = computed(() => getStatus(memPercentage.value))
const diskPercentage = computed(() => (props.node.disk ?? 0) / (props.node.disk_total || 1) * 100)
const diskStatus = computed(() => getStatus(diskPercentage.value))
const swapPercentage = computed(() => (props.node.swap ?? 0) / (props.node.swap_total || 1) * 100)
const swapStatus = computed(() => getStatus(swapPercentage.value))

const {
  latencyRenderBars,
  lossRenderBars,
  latencyDisplay,
  lossDisplay,
  latencyLabel,
  lossLabel,
  latencyPanelTooltip,
  lossPanelTooltip,
} = useNodePingDisplay(() => props.node.uuid)

function showTrafficProgress(node: NodeData): boolean {
  return node.traffic_limit > 0
}

const trafficUsedPercentage = computed(() => {
  if (props.node.traffic_limit <= 0)
    return 0
  const { net_total_up = 0, net_total_down = 0, traffic_limit_type } = props.node
  let used = 0
  switch (traffic_limit_type) {
    case 'up':
      used = net_total_up
      break
    case 'down':
      used = net_total_down
      break
    case 'min':
      used = Math.min(net_total_up, net_total_down)
      break
    case 'max':
      used = Math.max(net_total_up, net_total_down)
      break
    case 'sum':
    default:
      used = net_total_up + net_total_down
      break
  }
  return Math.min((used / props.node.traffic_limit) * 100, 100)
})

const trafficUsed = computed(() => {
  const { net_total_up = 0, net_total_down = 0, traffic_limit_type } = props.node
  switch (traffic_limit_type) {
    case 'up': return net_total_up
    case 'down': return net_total_down
    case 'min': return Math.min(net_total_up, net_total_down)
    case 'max': return Math.max(net_total_up, net_total_down)
    case 'sum':
    default: return net_total_up + net_total_down
  }
})

function getTrafficFillColor(percentage: number): string {
  if (percentage < 50)
    return 'rgb(59 130 246)'
  if (percentage <= 80)
    return 'rgb(249 115 22)'
  return 'rgb(239 68 68)'
}

const trafficFillStyle = computed(() => ({
  '--traffic-fill-level': showTrafficProgress(props.node) ? `${trafficUsedPercentage.value}%` : '0%',
  '--traffic-fill-color': getTrafficFillColor(trafficUsedPercentage.value),
}))
const trafficQuotaLabel = computed(() => showTrafficProgress(props.node)
  ? `${trafficUsedPercentage.value.toFixed(1)}%`
  : '∞')
const trafficLimitLabel = computed(() => showTrafficProgress(props.node)
  ? formatBytes(props.node.traffic_limit)
  : '∞')
const trafficPanelTooltip = computed(() => showTrafficProgress(props.node)
  ? `套餐流量 ${formatBytes(trafficUsed.value)} / ${formatBytes(props.node.traffic_limit)}`
  : `套餐流量 ${formatBytes(trafficUsed.value)} / ∞`)

// 是否显示金额：未登录且开启「未登录隐藏价格」时不显示价格 / 剩余价值，
// 但在线天数、剩余天数等非金额信息仍然展示
const showPrice = computed(() => appStore.isLoggedIn || !appStore.hidePriceWhenLoggedOut)

// 顶部标签：$112/年 · 在线7天 · 剩300天/$112
const onlineInfoTags = computed(() => {
  const node = props.node
  const lang = appStore.lang
  const days = Math.floor((node.uptime ?? 0) / 86400)
  const onlineText = lang === 'zh-CN' ? `在线${days}天` : `${days}d online`

  const expireDays = getDaysUntilExpired(node.expired_at)
  const expireStatus = getExpireStatus(node.expired_at)
  let remainDaysText: string
  if (expireStatus === 'expired')
    remainDaysText = lang === 'zh-CN' ? '已过期' : 'Expired'
  else if (expireStatus === 'long_term')
    remainDaysText = lang === 'zh-CN' ? '长期' : 'Long-term'
  else
    remainDaysText = lang === 'zh-CN' ? `剩${expireDays}天` : `${expireDays}d left`

  if (showPrice.value) {
    const priceLabel = (node.price === 0 || node.price === -1)
      ? (lang === 'zh-CN' ? '免费' : 'Free')
      : formatCurrencyValue(node.price, node.currency)
    const priceCycleText = `${priceLabel}/${getBillingCycleText(node.billing_cycle, lang)}`
    const remainValue = formatCurrencyValue(
      getRemainingValue(node.price, node.billing_cycle, node.expired_at),
      node.currency,
    )
    return [priceCycleText, onlineText, `${remainDaysText}/${remainValue}`]
  }

  return [onlineText, remainDaysText]
})

const customTags = computed(() => parseTags(props.node.tags).map(t => t.text))

function hasRegion(region: string | null | undefined): boolean {
  return Boolean(region?.trim())
}
</script>

<template>
  <CardX
    hoverable
    class="node-card w-full cursor-pointer border-none shadow-[0_0_0_3px] shadow-transparent transition-all duration-200 rounded-xl"
    :class="[!props.node.online && '!shadow-red-500/30']"
    @click="emit('click')"
  >
    <!-- 头部：在线点 + 名称 -->
    <template #header>
      <div class="flex items-center gap-2 min-w-0">
        <div class="relative size-2.5 shrink-0">
          <span
            class="size-2.5 rounded-full block"
            :class="props.node.online ? 'bg-green-500' : 'bg-red-500'"
          />
          <span
            class="animate-ping absolute inset-0 rounded-full opacity-60"
            :class="props.node.online ? 'bg-green-500' : 'bg-red-500'"
          />
        </div>
        <span class="text-sm font-bold flex-1 min-w-0 truncate">{{ props.node.name }}</span>
      </div>
    </template>

    <!-- 头部右侧：OS + 国旗 -->
    <template #header-extra>
      <div class="flex gap-1.5 items-center shrink-0">
        <img :src="getOSImage(props.node.os)" :alt="getOSName(props.node.os)" class="size-4">
        <img
          v-if="hasRegion(props.node.region)"
          :src="`/images/flags/${getRegionCode(props.node.region)}.svg`"
          :alt="getRegionDisplayName(props.node.region)"
          class="size-5 shrink-0"
        >
      </div>
    </template>

    <template #default>
      <div class="flex flex-col gap-3 relative">
        <!-- 顶部标签行 -->
        <div v-if="onlineInfoTags.length" class="flex gap-1.5 flex-wrap -mt-1">
          <span
            v-for="(tag, i) in onlineInfoTags" :key="i"
            class="text-[11px] px-2 py-0.5 rounded-full bg-slate-500/10 text-muted-foreground leading-tight"
          >
            {{ tag }}
          </span>
        </div>

        <!-- 四项进度条 -->
        <div class="grid grid-cols-2 gap-x-4 gap-y-2.5">
          <!-- CPU -->
          <div class="flex flex-col gap-1">
            <div class="flex justify-between text-xs">
              <span class="text-muted-foreground">CPU</span>
              <span class="tabular-nums font-medium">{{ (props.node.cpu ?? 0).toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="props.node.cpu ?? 0" :status="cpuStatus" :height="4" />
            <div class="text-[11px] text-muted-foreground truncate">
              {{ props.node.load.toFixed(2) }}, {{ props.node.load5.toFixed(2) }}, {{ props.node.load15.toFixed(2) }}
            </div>
          </div>

          <!-- 内存 -->
          <div class="flex flex-col gap-1">
            <div class="flex justify-between text-xs">
              <span class="text-muted-foreground">内存</span>
              <span class="tabular-nums font-medium">{{ memPercentage.toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="memPercentage" :status="memStatus" :height="4" />
            <div class="text-[11px] text-muted-foreground truncate">
              {{ formatBytes(props.node.ram ?? 0) }} / {{ formatBytes(props.node.mem_total ?? 0) }}
            </div>
          </div>

          <!-- 硬盘 -->
          <div class="flex flex-col gap-1">
            <div class="flex justify-between text-xs">
              <span class="text-muted-foreground">硬盘</span>
              <span class="tabular-nums font-medium">{{ diskPercentage.toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="diskPercentage" :status="diskStatus" :height="4" />
            <div class="text-[11px] text-muted-foreground truncate">
              {{ formatBytes(props.node.disk ?? 0) }} / {{ formatBytes(props.node.disk_total ?? 0) }}
            </div>
          </div>

          <!-- 交换 -->
          <div class="flex flex-col gap-1">
            <div class="flex justify-between text-xs">
              <span class="text-muted-foreground">交换</span>
              <span class="tabular-nums font-medium">{{ swapPercentage.toFixed(1) }}%</span>
            </div>
            <ProgressThin :percentage="swapPercentage" :status="swapStatus" :height="4" />
            <div class="text-[11px] text-muted-foreground truncate">
              {{ formatBytes(props.node.swap ?? 0) }} / {{ formatBytes(props.node.swap_total ?? 0) }}
            </div>
          </div>
        </div>

        <!-- 两列：网速 / 总流量 -->
        <div class="grid grid-cols-2 gap-1.5">
          <!-- 实时网速 -->
          <div class="flex flex-col gap-0.5 px-2 py-1.5 rounded-lg bg-slate-500/5 min-w-0">
            <div class="text-[11px] text-green-600 flex items-center gap-1">
              <Icon icon="tabler:chevron-up" width="11" height="11" />
              <span class="truncate min-w-0">{{ formatBytesPerSecond(props.node.net_out ?? 0) }}</span>
            </div>
            <div class="text-[11px] text-blue-600 flex items-center gap-1">
              <Icon icon="tabler:chevron-down" width="11" height="11" />
              <span class="truncate min-w-0">{{ formatBytesPerSecond(props.node.net_in ?? 0) }}</span>
            </div>
          </div>

          <!-- 总流量 + 套餐使用渐变 -->
          <div
            class="traffic-total-panel relative isolate flex flex-col gap-0.5 px-2 py-1.5 rounded-lg bg-slate-500/5 min-w-0 overflow-hidden"
            :style="trafficFillStyle"
            :title="trafficPanelTooltip"
          >
            <div
              class="traffic-quota-label pointer-events-none absolute inset-y-0 right-2 z-1 flex flex-col items-end justify-center gap-0.5 text-[11px] font-medium tabular-nums opacity-60"
            >
              <span>{{ trafficQuotaLabel }}</span>
              <span>{{ trafficLimitLabel }}</span>
            </div>
            <div class="relative z-2 text-[11px] text-muted-foreground flex items-center gap-1 min-w-0">
              <Icon icon="tabler:upload" width="11" height="11" class="shrink-0" />
              <span class="truncate min-w-0">{{ formatBytes(props.node.net_total_up ?? 0) }}</span>
            </div>
            <div class="relative z-2 text-[11px] text-muted-foreground flex items-center gap-1 min-w-0">
              <Icon icon="tabler:download" width="11" height="11" class="shrink-0" />
              <span class="truncate min-w-0">{{ formatBytes(props.node.net_total_down ?? 0) }}</span>
            </div>
          </div>
        </div>

        <!-- 延迟 + 丢包 -->
        <div class="grid grid-cols-2 gap-1.5">
          <div
            class="group/panel relative flex flex-col gap-1.5 p-2 h-11 rounded-lg bg-slate-500/5"
            :class="[!props.node.online ? 'blur-xs opacity-50' : '']"
            :title="latencyPanelTooltip"
          >
            <div class="flex items-center justify-between text-[11px] leading-none">
              <span class="text-muted-foreground truncate min-w-0">{{ latencyLabel }}</span>
              <span class="font-medium">{{ latencyDisplay }}</span>
            </div>
            <div
              class="grid h-full items-end gap-[1px] opacity-80 group-hover/panel:opacity-100"
              :style="{ gridTemplateColumns: `repeat(${latencyRenderBars.length}, minmax(0, 1fr))` }"
            >
              <DataTooltip
                v-for="bar in latencyRenderBars" :key="bar.key"
                placement="top" :content="bar.tooltip" class="h-full w-full"
              >
                <span
                  class="block h-full w-full rounded-[1px] transition-transform duration-150 group-hover/data-tooltip:scale-y-160 group-hover/panel:opacity-60 group-hover/data-tooltip:!opacity-100"
                  :class="bar.className"
                />
              </DataTooltip>
            </div>
          </div>

          <div
            class="group/panel relative flex flex-col gap-1.5 p-2 h-11 rounded-lg bg-slate-500/5"
            :class="[!props.node.online ? 'blur-xs opacity-50' : '']"
            :title="lossPanelTooltip"
          >
            <div class="flex items-center justify-between text-[11px] leading-none">
              <span class="text-muted-foreground truncate min-w-0">{{ lossLabel }}</span>
              <span class="font-medium">{{ lossDisplay }}</span>
            </div>
            <div
              class="grid h-full items-end gap-[1px] opacity-80 group-hover/panel:opacity-100"
              :style="{ gridTemplateColumns: `repeat(${lossRenderBars.length}, minmax(0, 1fr))` }"
            >
              <DataTooltip
                v-for="bar in lossRenderBars" :key="bar.key"
                placement="top" :content="bar.tooltip" class="h-full w-full"
              >
                <span
                  class="block h-full w-full rounded-[1px] transition-transform duration-150 group-hover/data-tooltip:scale-y-160 group-hover/panel:opacity-60 group-hover/data-tooltip:!opacity-100"
                  :class="bar.className"
                />
              </DataTooltip>
            </div>
          </div>
        </div>

        <!-- 自定义标签 -->
        <div v-if="customTags.length > 0" class="flex flex-wrap gap-1">
          <Badge
            v-for="(tag, i) in customTags" :key="i"
            variant="outline"
            class="!text-[11px] rounded-full text-muted-foreground border-muted-foreground/15 px-2 py-0"
          >
            {{ tag }}
          </Badge>
        </div>

        <!-- 离线遮罩 -->
        <div
          v-if="!props.node.online"
          class="absolute inset-0 flex flex-col items-center justify-center z-10 rounded-xl bg-white/20 dark:bg-black/20 backdrop-blur-[2px]"
        >
          <div class="text-sm font-semibold text-destructive">
            离线
          </div>
          <div class="text-[11px] text-muted-foreground mt-1">
            {{ offlineTime }}
          </div>
        </div>
      </div>
    </template>
  </CardX>
</template>

<style scoped>
.node-card {
  position: relative;
  overflow: hidden;
}

.traffic-quota-label {
  color: var(--traffic-fill-color);
  transition: color 500ms ease;
}

.traffic-total-panel {
  background-image: linear-gradient(
    to right,
    color-mix(in srgb, var(--traffic-fill-color) 3%, transparent) 0%,
    color-mix(in srgb, var(--traffic-fill-color) 22%, transparent) var(--traffic-fill-level),
    transparent var(--traffic-fill-level),
    transparent 100%
  );
}

@media (prefers-reduced-motion: reduce) {
  .traffic-quota-label {
    transition: none;
  }
}
</style>
