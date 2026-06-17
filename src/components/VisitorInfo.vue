<script setup lang="ts">
import { Icon } from '@iconify/vue'
import { useWindowSize } from '@vueuse/core'
import { computed, onMounted, ref } from 'vue'

interface VisitorData {
  ip: string
  country: string
  org: string
}

const show = ref(false)
const dismissed = ref(false)
const visitor = ref<VisitorData | null>(null)
const { width } = useWindowSize()
const showOrg = computed(() => width.value >= 640)

onMounted(async () => {
  try {
    const res = await fetch('https://ipapi.co/json/')
    const data = await res.json()
    visitor.value = {
      ip: data.ip,
      country: data.country_name,
      org: data.org,
    }
  }
  catch {
    visitor.value = null
  }
  finally {
    setTimeout(() => {
      show.value = true
    }, 600)
  }
})
</script>

<template>
  <Transition name="slide-up">
    <div
      v-if="show && visitor"
      class="fixed bottom-4 left-1/2 -translate-x-1/2 z-50
             flex items-center gap-2 px-4 py-1.5 rounded-full
             bg-white/55 dark:bg-black/50
             backdrop-blur-md
             border border-white/40 dark:border-white/10
             shadow-lg text-[13px] select-none whitespace-nowrap"
    >
      <Icon icon="icon-park-outline:earth" :width="14" :height="14" class="text-blue-500 shrink-0" />
      <span class="text-muted-foreground">Your IP:</span>
      <span class="font-semibold text-foreground">{{ visitor.ip }}</span>
      <span class="text-muted-foreground/40">|</span>
      <span class="text-muted-foreground">{{ visitor.country }}</span>
      <template v-if="showOrg">
        <span class="text-muted-foreground/40">|</span>
        <span class="text-muted-foreground truncate max-w-[220px]">{{ visitor.org }}</span>
      </template>
    </div>
  </Transition>
</template>

<style scoped>
.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.4s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.slide-up-enter-from,
.slide-up-leave-to {
  opacity: 0;
  transform: translateX(-50%) translateY(20px);
}
</style>
