<script lang="ts" setup>
import { useRoute } from 'vue-router';

import DesktopLinkItem from './DesktopLinkItem.vue';

/**
 * COMPOSABLES
 */
const route = useRoute();

/**
 * METHODS
 */
function isActive(page: string): boolean {
  if (route.name === page) return true;
  return false;
}
</script>

<template>
  <div class="desktop-links">
    <DesktopLinkItem to="/" :active="isActive('home')">
      {{ $t('invest') }}
    </DesktopLinkItem>
    <DesktopLinkItem to="/trade" :active="isActive('trade')">
      {{ $t('trade') }}
    </DesktopLinkItem>
    <DesktopLinkItem to="/vehldr" :active="isActive('vehldr')">
      veHLDR
    </DesktopLinkItem>
    <DesktopLinkItem to="/bribe" :active="isActive('bribe')">
      Bribe
    </DesktopLinkItem> 
    <DesktopLinkItem to="/claim" :active="isActive('claim')">
      <div class="flex items-center">
        {{ $t('claim') }}
        <StarsIcon class="ml-1 h-5 w-4" />
      </div>
    </DesktopLinkItem>
  </div>
</template>

<style scoped>
.desktop-links {
  @apply grid gap-6 grid-flow-col grid-rows-1 h-full content-center;
}
</style>
