<script lang="ts" setup>
import { computed } from 'vue';
import { useRouter } from 'vue-router';

import usePools from '@/composables/pools/usePools';
import useStaking from '@/composables/staking/useStaking';
import useDarkMode from '@/composables/useDarkMode';
import useFathom from '@/composables/useFathom';
import { useLock } from '@/composables/useLock';
import { isL2 } from '@/composables/useNetwork';
import useNumbers, { FNumFormats } from '@/composables/useNumbers';
import { EXTERNAL_LINKS } from '@/constants/links';
import { bnum } from '@/lib/utils';
import useWeb3 from '@/services/web3/useWeb3';

/**
 * COMPOSABLES
 */
const router = useRouter();
const { fNum2 } = useNumbers();
const {
  isWalletReady,
  toggleWalletSelectModal,
  isWalletConnecting
} = useWeb3();
const { trackGoal, Goals } = useFathom();
const { totalInvestedAmount, isLoadingUserPools } = usePools();
const { darkMode } = useDarkMode();
const { lockFiatValue, isLoadingLock } = useLock();
const {
  userData: {
    totalStakedFiatValue,
    isLoadingUserStakingData,
    isLoadingStakedPools,
    isUserStakeDataIdle
  }
} = useStaking();

/**
 * COMPUTED
 */
const classes = computed(() => ({
  ['h-72']: !isWalletReady.value && !isWalletConnecting.value,
  ['h-56']: isWalletReady.value || isWalletConnecting.value
}));

const isStakingLoading = computed(() => {
  return (
    isLoadingStakedPools.value ||
    isLoadingUserStakingData.value ||
    isUserStakeDataIdle.value
  );
});

const totalInvestedLabel = computed((): string => {
  const value = bnum(totalInvestedAmount.value || '0')
    .plus(lockFiatValue.value)
    .plus(totalStakedFiatValue.value)
    .toString();
  return fNum2(value, FNumFormats.fiat);
});

const totalVeBalLabel = computed((): string =>
  fNum2(lockFiatValue.value, FNumFormats.fiat)
);

const isLoadingLockAndStaking = computed(
  (): boolean => (!isL2.value && isLoadingLock.value) || isStakingLoading.value
);

const isLoadingTotalValue = computed(
  (): boolean => isLoadingUserPools.value || isLoadingLockAndStaking.value
);

/**
 * METHODS
 */
function onClickConnect() {
  toggleWalletSelectModal(true);
  trackGoal(Goals.ClickHeroConnectWallet);
}
</script>

<template>
  <div :class="['app-hero', classes]">
    <div class="w-full max-w-2xl mx-auto">
      <template v-if="isWalletReady || isWalletConnecting">
        <h4 class="pb-3">
          <BalLink
            href="https://www.notion.so/solacefi/Holdr-Friendly-Fork-Proposal-d6d6805fe17d447280bc20742eb19b26"
            external
            noStyle
          >
            Click here to see proposal for Holdr
          </BalLink>
        </h4>
        <h1
          v-text="$t('myBalancerInvestments')"
          class="text-base font-medium text-white opacity-90 font-body mb-2"
        />
        <BalLoadingBlock
          v-if="isLoadingTotalValue"
          class="h-10 w-40 mx-auto"
          white
        />
        <div v-else class="text-3xl font-bold text-white mb-1">
          {{ totalInvestedLabel }}
        </div>
        <div v-if="!isL2" class="relative mt-2 inline-block">
          <BalLoadingBlock
            v-if="isLoadingTotalValue"
            class="h-8 w-40 mx-auto"
            white
          />
          <div
            v-else
            class="
              vebal-banner
              h-8
              flex
              items-center
              px-3
              text-blue-500 text-sm
              font-medium
              cursor-pointer
              border border-blue-500
              group
              hover:text-white
              focus:text-white
              transition-colors
              rounded-bl rounded-tr
            "
            @click="router.push({ name: 'vehldr' })"
          >
            <span v-if="lockFiatValue === '0'"
              >{{ lockFiatValue }} {{ $t('veBAL.hero.tokens.veBAL') }}</span
            >
            <span v-else>{{ $t('inclXInVeBal', [totalVeBalLabel]) }}</span>
          </div>
        </div>
      </template>
      <template v-else>
        <h4 class="pb-3">
          <BalLink
            href="https://www.notion.so/solacefi/Holdr-Friendly-Fork-Proposal-d6d6805fe17d447280bc20742eb19b26"
            external
            noStyle
          >
            Click here to see proposal for Holdr
          </BalLink>
        </h4>
        <h1 v-text="$t('ammPlatform')" class="headline" />
        <div class="flex justify-center mt-4">
          <BalBtn
            :color="darkMode ? 'gray' : 'white'"
            class="mr-3"
            @click="onClickConnect"
          >
            {{ $t('connectWallet') }}
          </BalBtn>
          <BalBtn
            tag="a"
            :href="EXTERNAL_LINKS.Balancer.Home"
            target="_blank"
            rel="noreferrer"
            color="white"
            outline
            @click="trackGoal(Goals.ClickHeroLearnMore)"
          >
            {{ $t('learnMore') }}
            <BalIcon name="arrow-up-right" size="sm" class="ml-1" />
          </BalBtn>
        </div>
      </template>
    </div>
  </div>
</template>

<style>
.app-hero {
  @apply bg-cover bg-center flex items-center justify-center text-center px-4;
  transition: all 0.3s ease-in-out;
  background-image: url('/images/backgrounds/bg-header.svg');
}

.vebal-banner::before {
  @apply border border-blue-500;
  content: '';
  width: 16px;
  height: 6px;
  left: 0;
  top: -5px;
  position: absolute;
  border-top-left-radius: 8px;
}
.vebal-banner::after {
  @apply border border-blue-500;
  content: '';
  width: 16px;
  height: 6px;
  bottom: -5px;
  right: 0;
  position: absolute;
  border-bottom-right-radius: 8px;
}
.headline {
  @apply text-white text-center text-4xl md:text-5xl pb-2 font-display font-black;
  font-weight: 600;
  font-variation-settings: 'wght' 700;
  font-family: system-ui;
}
</style>
