<template>
  <BalCard class="relative" :shadow="tradeCardShadow" :no-border="!darkMode">
    <template v-slot:header>
      <div class="w-full flex items-center justify-between">
        <h4 class="font-bold">{{ title }}</h4>
        <TradeSettingsPopover :context="TradeSettingsContext.trade" />
      </div>
    </template>
    <div>
      <TradePair
        v-model:tokenInAmount="tokenInAmount"
        v-model:tokenInAddress="tokenInAddress"
        v-model:tokenOutAmount="tokenOutAmount"
        v-model:tokenOutAddress="tokenOutAddress"
        v-model:exactIn="exactIn"
        :priceImpact="priceImpact"
        @amountChange="handleAmountChange"
        class="mb-4"
      />
      <BalAlert
        v-if="error"
        class="mb-4"
        type="error"
        size="sm"
        :title="error.header"
        :description="error.body"
        :action-label="error.label"
        block
        @actionClick="handleErrorButtonClick"
      />
      <BalBtn
        v-if="poolsLoading || isLoadingApprovals"
        :loading="true"
        :loading-label="$t('loading')"
        block
      />
      <BalBtn
        v-else
        :label="'Preview trade'"
        :disabled="tradeDisabled"
        :loading-label="$t('confirming')"
        color="gradient"
        block
        @click.prevent="showTradePreviewModal"
      />
      <TradeRoute
        class="mt-5"
        :address-in="tokenInAddress"
        :amount-in="tokenInAmount"
        :address-out="tokenOutAddress"
        :amount-out="tokenOutAmount"
        :pools="pools"
        :sor-return="sorReturn"
      />
    </div>
    <SuccessOverlay
      v-if="tradeSuccess"
      :title="$t('tradeSettled')"
      :description="$t('tradeSuccess')"
      :closeLabel="$t('close')"
      :explorer-link="explorer.txLink(txHash)"
      @close="tradeSuccess = false"
    />
  </BalCard>
  <teleport to="#modal">
    <TradePreviewModal
      v-if="modalTradePreviewIsOpen"
      :address-in="tokenInAddress"
      :amount-in="tokenInAmount"
      :address-out="tokenOutAddress"
      :amount-out="tokenOutAmount"
      :trading="trading"
      @trade="trade"
      @close="modalTradePreviewIsOpen = false"
    />
  </teleport>
</template>

<script lang="ts">
import { getAddress, isAddress } from '@ethersproject/address';
import { computed, defineComponent, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRouter } from 'vue-router';
import { useStore } from 'vuex';

import SuccessOverlay from '@/components/cards/SuccessOverlay.vue';
import TradePair from '@/components/cards/TradeCard/TradePair.vue';
import TradeRoute from '@/components/cards/TradeCard/TradeRoute.vue';
import TradePreviewModal from '@/components/modals/TradePreviewModal.vue';
import TradeSettingsPopover, {
  TradeSettingsContext
} from '@/components/popovers/TradeSettingsPopover.vue';
import useSor from '@/composables/trade/useSor';
import useTokenApproval from '@/composables/trade/useTokenApproval';
import { useTradeState } from '@/composables/trade/useTradeState';
import useValidation, {
  TradeValidation
} from '@/composables/trade/useValidation';
import useBreakpoints from '@/composables/useBreakpoints';
import useDarkMode from '@/composables/useDarkMode';
import useTokens from '@/composables/useTokens';
import useUserSettings from '@/composables/useUserSettings';
import { getWrapAction, WrapType } from '@/lib/utils/balancer/wrapper';
import { isRequired } from '@/lib/utils/validations';
import useWeb3 from '@/services/web3/useWeb3';

export default defineComponent({
  components: {
    SuccessOverlay,
    TradePair,
    TradePreviewModal,
    TradeRoute,
    TradeSettingsPopover
  },

  setup() {
    const highPiAccepted = ref(false);
    const store = useStore();
    const router = useRouter();
    const { explorerLinks, isMismatchedNetwork } = useWeb3();
    const { t } = useI18n();
    const { bp } = useBreakpoints();

    const { tokens, getToken, nativeAsset } = useTokens();
    const { userNetworkConfig } = useWeb3();
    const { darkMode } = useDarkMode();
    const {
      tokenInAddress,
      tokenOutAddress,
      tokenInAmount,
      tokenOutAmount,
      setTokenInAddress,
      setTokenOutAddress
    } = useTradeState();
    const { slippage } = useUserSettings();

    const exactIn = ref(true);

    const tradeSuccess = ref(false);
    const txHash = ref('');
    const modalTradePreviewIsOpen = ref(false);

    const slippageBufferRate = computed(() => parseFloat(slippage.value));

    const tokenIn = computed(() => getToken(tokenInAddress.value));

    const tokenOut = computed(() => getToken(tokenOutAddress.value));

    const tradeCardShadow = computed(() => {
      switch (bp.value) {
        case 'xs':
          return 'none';
        case 'sm':
          return 'lg';
        default:
          return 'xl';
      }
    });

    const wrapType = computed(() =>
      getWrapAction(tokenInAddress.value, tokenOutAddress.value)
    );
    const isWrap = computed(() => wrapType.value === WrapType.Wrap);
    const isUnwrap = computed(() => wrapType.value === WrapType.Unwrap);

    const isHighPriceImpact = computed(() => {
      return priceImpact.value >= 0.05 && !highPiAccepted.value;
    });

    const tradeDisabled = computed(() => {
      if (isMismatchedNetwork.value) return true;
      if (errorMessage.value !== TradeValidation.VALID) return true;
      if (isHighPriceImpact.value) return true;
      return false;
    });

    // COMPOSABLES
    const { isLoading: isLoadingApprovals } = useTokenApproval(
      tokenInAddress,
      tokenInAmount,
      tokens
    );
    const {
      trading,
      trade,
      initSor,
      handleAmountChange,
      priceImpact,
      sorReturn,
      latestTxHash,
      pools,
      fetchPools,
      poolsLoading
    } = useSor({
      exactIn,
      tokenInAddressInput: tokenInAddress,
      tokenInAmountInput: tokenInAmount,
      tokenOutAddressInput: tokenOutAddress,
      tokenOutAmountInput: tokenOutAmount,
      wrapType,
      tokenIn,
      tokenOut,
      slippageBufferRate
    });
    const { errorMessage } = useValidation(
      tokenInAddress,
      tokenInAmount,
      tokenOutAddress,
      tokenOutAmount
    );

    const title = computed(() => {
      if (isWrap.value) return t('wrap');
      if (isUnwrap.value) return t('unwrap');
      return t('trade');
    });

    const error = computed(() => {
      if (isHighPriceImpact.value) {
        return {
          header: t('highPriceImpact'),
          body: t('highPriceImpactDetailed'),
          label: t('accept')
        };
      }
      switch (errorMessage.value) {
        case TradeValidation.NO_LIQUIDITY:
          return {
            header: t('insufficientLiquidity'),
            body: t('insufficientLiquidityDetailed')
          };
        default:
          return undefined;
      }
    });

    function handleErrorButtonClick() {
      if (isHighPriceImpact.value) {
        highPiAccepted.value = true;
      }
    }

    // SOLACE_TODO: populate initial tokens via localstorage
    async function populateInitialTokens(): Promise<void> {
      let assetIn = router.currentRoute.value.params.assetIn as string;
      if (assetIn === nativeAsset.deeplinkId) assetIn = nativeAsset.address;
      else if (isAddress(assetIn)) assetIn = getAddress(assetIn);
      let assetOut = router.currentRoute.value.params.assetOut as string;
      if (assetOut === nativeAsset.deeplinkId) assetOut = nativeAsset.address;
      else if (isAddress(assetOut)) assetOut = getAddress(assetOut);

      setTokenInAddress(assetIn || store.state.trade.inputAsset);
      setTokenOutAddress(assetOut || '');
    }

    function showTradePreviewModal() {
      modalTradePreviewIsOpen.value = true;
    }

    watch(userNetworkConfig, async () => {
      await initSor();
      await handleAmountChange();
    });

    watch(tokenInAddress, () => {
      store.commit('trade/setInputAsset', tokenInAddress.value);
      handleAmountChange();
    });

    watch(tokenOutAddress, () => {
      store.commit('trade/setOutputAsset', tokenOutAddress.value);
      handleAmountChange();
    });

    watch(latestTxHash, () => {
      // Refresh SOR pools
      fetchPools();
      txHash.value = latestTxHash.value;
      tradeSuccess.value = true;
      modalTradePreviewIsOpen.value = false;
    });

    populateInitialTokens();

    return {
      highPiAccepted,
      title,
      error,
      handleErrorButtonClick,
      tokenInAddress,
      tokenInAmount,
      tokenOutAddress,
      tokenOutAmount,
      exactIn,
      handleAmountChange,
      errorMessage,
      sorReturn,
      pools,
      trading,
      trade,
      txHash,
      modalTradePreviewIsOpen,
      tradeSuccess,
      priceImpact,
      isRequired,
      tradeDisabled,
      TradeSettingsContext,
      poolsLoading,
      showTradePreviewModal,
      isLoadingApprovals,
      bp,
      darkMode,
      tradeCardShadow,
      explorer: explorerLinks
    };
  }
});
</script>
