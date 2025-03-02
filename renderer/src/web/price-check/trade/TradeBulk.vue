<template>
  <div v-if="!error" class="layout-column min-h-0" style="height: auto">
    <!-- @TODO: fix "Matched" text jumping (min-height: 22px) -->
    <div
      class="mb-2 flex pl-2 justify-between items-baseline"
      style="min-height: 1.375rem"
    >
      <div class="flex items-center text-gray-500">
        <span class="mr-1">{{ t(":matched") }}</span>
        <span v-if="!result" class="text-gray-600">...</span>
        <div v-else class="flex items-center">
          <button
            class="btn flex items-center mr-1"
            :style="{
              background:
                selectedCurr !== 'xchgExalted' ? 'transparent' : undefined,
            }"
            @click="selectedCurr = 'xchgExalted'"
          >
            <img src="/images/exa.png" class="trade-bulk-currency-icon" />
            <span>{{ result.xchgExalted.listed.value?.total ?? "?" }}</span>
          </button>
          <button
            class="btn flex items-center mr-1"
            :style="{
              background:
                selectedCurr !== 'xchgStable' ? 'transparent' : undefined,
            }"
            @click="selectedCurr = 'xchgStable'"
          >
            <img src="/images/divine.png" class="trade-bulk-currency-icon" />
            <span>{{ result.xchgStable.listed.value?.total ?? "?" }}</span>
          </button>
          <span class="ml-1"><online-filter :filters="filters" /></span>
        </div>
      </div>
      <trade-links v-if="result" :get-link="makeTradeLink" />
    </div>
    <div class="layout-column overflow-y-auto overflow-x-hidden">
      <table class="table-stripped w-full">
        <thead>
          <tr class="text-left">
            <th class="trade-table-heading">
              <div class="px-2">{{ t(":price") }}</div>
            </th>
            <th class="trade-table-heading">
              <div
                class="pl-1 pr-2 flex text-xs"
                style="line-height: 1.3125rem"
              >
                <span class="w-8 inline-block text-right -ml-px mr-px">{{
                  selectedCurr === "xchgExalted" ? "exalt" : "div"
                }}</span
                ><span>{{ "\u2009" }}/{{ "\u2009" }}</span
                ><span class="w-8 inline-block">{{ t(":bulk") }}</span>
              </div>
            </th>
            <th class="trade-table-heading">
              <div class="px-1">{{ t(":stock") }}</div>
            </th>
            <th class="trade-table-heading">
              <div class="px-1">{{ t(":fulfill") }}</div>
            </th>
            <th class="trade-table-heading" :class="{ 'w-full': !showSeller }">
              <div class="pr-2 pl-4">
                <span class="ml-1" style="padding-left: 0.375rem">{{
                  t(":listed")
                }}</span>
              </div>
            </th>
            <th v-if="showSeller" class="trade-table-heading w-full">
              <div class="px-2">{{ t(":seller") }}</div>
            </th>
          </tr>
        </thead>
        <tbody style="overflow: scroll">
          <template v-for="(result, idx) in selectedResults">
            <tr v-if="!result" :key="idx">
              <td colspan="100" class="text-transparent">***</td>
            </tr>
            <tr v-else :key="result.id">
              <td class="px-2">
                {{
                  Number((result.exchangeAmount / result.itemAmount).toFixed(4))
                }}
              </td>
              <td class="pl-1 whitespace-nowrap">
                <span class="w-8 inline-block text-right">{{
                  result.exchangeAmount
                }}</span
                ><span>{{ "\u2009" }}/{{ "\u2009" }}</span
                ><span class="w-8 inline-block">{{ result.itemAmount }}</span>
              </td>
              <td class="px-1 text-right">{{ result.stock }}</td>
              <td class="px-1 text-right">
                <i
                  v-if="result.stock < result.itemAmount"
                  class="fas fa-exclamation-triangle mr-1 text-gray-500"
                ></i
                >{{ Math.floor(result.stock / result.itemAmount) }}
              </td>
              <td class="pr-2 pl-4 whitespace-nowrap">
                <div class="inline-flex items-center">
                  <div
                    class="account-status"
                    :class="result.accountStatus"
                  ></div>
                  <div class="ml-1 font-sans text-xs">
                    {{ result.relativeDate }}
                  </div>
                </div>
                <span
                  v-if="!showSeller && result.isMine"
                  class="rounded px-1 text-gray-800 bg-gray-400 ml-1"
                  >{{ t("You") }}</span
                >
              </td>
              <td v-if="showSeller" class="px-2 whitespace-nowrap">
                <span
                  v-if="result.isMine"
                  class="rounded px-1 text-gray-800 bg-gray-400"
                  >{{ t("You") }}</span
                >
                <span v-else class="font-sans text-xs">{{
                  showSeller === "ign" ? result.ign : result.accountName
                }}</span>
              </td>
            </tr>
          </template>
        </tbody>
      </table>
    </div>
  </div>
  <ui-error-box class="mb-4" v-else>
    <template #name>{{ t(":error") }}</template>
    <p>Error: {{ error }}</p>
    <template #actions>
      <button class="btn" @click="execSearch">{{ t("Retry") }}</button>
      <button class="btn" @click="openTradeLink">{{ t("Browser") }}</button>
    </template>
  </ui-error-box>
</template>

<script lang="ts">
import {
  defineComponent,
  PropType,
  computed,
  watch,
  ComputedRef,
  Ref,
  shallowRef,
  shallowReactive,
  inject,
} from "vue";
import { useI18nNs } from "@/web/i18n";
import UiErrorBox from "@/web/ui/UiErrorBox.vue";
import {
  BulkSearch,
  execBulkSearch,
  createTradeRequest,
  PricingResult,
} from "./pathofexile-bulk";
import { getTradeEndpoint } from "./common";
import { AppConfig } from "@/web/Config";
import { ItemFilters } from "../filters/interfaces";
import { ParsedItem } from "@/parser";
import { PriceCheckWidget } from "@/web/overlay/interfaces";
import { artificialSlowdown } from "./artificial-slowdown";
import OnlineFilter from "./OnlineFilter.vue";
import TradeLinks from "./TradeLinks.vue";
import { Host } from "@/web/background/IPC";

const slowdown = artificialSlowdown(900);

function useBulkApi() {
  type BulkSearchExtended = Record<
    "xchgExalted" | "xchgStable",
    {
      listed: Ref<BulkSearch | null>;
      listedLazy: ComputedRef<PricingResult[]>;
    }
  >;

  let searchId = 0;
  const error = shallowRef<string | null>(null);
  const result = shallowRef<BulkSearchExtended | null>(null);

  async function search(item: ParsedItem, filters: ItemFilters) {
    try {
      searchId += 1;
      error.value = null;
      result.value = null;

      const _searchId = searchId;

      // override, because at league start many players set wrong price, and this breaks optimistic search
      const have =
        item.info.refName === "Exalted Orb"
          ? ["divine"]
          : item.info.refName === "Divine Orb"
            ? ["exalted"]
            : ["divine", "exalted"];

      const optimisticSearch = await execBulkSearch(item, filters, have, {
        accountName: AppConfig().accountName,
      });
      if (_searchId === searchId) {
        result.value = {
          xchgStable: getResultsByHave(
            item,
            filters,
            optimisticSearch,
            "divine",
          ),
          xchgExalted: getResultsByHave(
            item,
            filters,
            optimisticSearch,
            "exalted",
          ),
        };
      }
    } catch (err) {
      error.value = (err as Error).message;
    }
  }

  function getResultsByHave(
    item: ParsedItem,
    filters: ItemFilters,
    preloaded: Array<BulkSearch | null>,
    have: "divine" | "exalted",
  ) {
    const _result = shallowRef(
      preloaded.some((res) => res?.haveTag === have)
        ? shallowReactive(preloaded.find((res) => res?.haveTag === have)!)
        : null,
    );
    const items = shallowRef<PricingResult[]>(_result.value?.listed ?? []);
    let requested: boolean = _result.value != null;

    const listedLazy = computed(() => {
      if (!requested) {
        (async function () {
          try {
            requested = true;
            _result.value = shallowReactive(
              (
                await execBulkSearch(item, filters, [have], {
                  accountName: AppConfig().accountName,
                })
              )[0]!,
            );
            items.value = _result.value.listed;
            const otherHave =
              have === "divine"
                ? result.value?.xchgExalted?.listed.value!
                : result.value?.xchgStable?.listed.value!;
            // fix best guess we did while making optimistic search
            otherHave.total -= _result.value.total;
          } catch (err) {
            error.value = (err as Error).message;
          }
        })();
      }

      return items.value;
    });

    return { listed: _result, listedLazy };
  }

  return { error, result, search };
}

// function tempOverrideApi() {
//   async function search(item: ParsedItem, filters: ItemFilters) {
//     console.warn("Bulk Exchange is not available in this version.");
//   }

//   const error = shallowRef<string | null>(null);
//   error.value = "Bulk Exchange is not available in this version.";

//   const result: ShallowRef<Record<
//     "xchgExalted" | "xchgStable",
//     {
//       listed: Ref<BulkSearch | null>;
//       listedLazy: ComputedRef<PricingResult[]>;
//     }
//   > | null> = shallowRef(null);

//   return { error, result, search };
// }

export default defineComponent({
  components: { OnlineFilter, TradeLinks, UiErrorBox },
  props: {
    filters: {
      type: Object as PropType<ItemFilters>,
      required: true,
    },
    item: {
      type: Object as PropType<ParsedItem>,
      required: true,
    },
  },
  setup(props) {
    const widget = computed(() => AppConfig<PriceCheckWidget>("price-check")!);
    const { error, result, search } = useBulkApi();
    // const { error, result, search } = tempOverrideApi();

    const showBrowser = inject<(url: string) => void>("builtin-browser")!;

    const selectedCurr = shallowRef<"xchgExalted" | "xchgStable">(
      "xchgExalted",
    );

    watch(
      () => props.item,
      (item) => {
        slowdown.reset(item);
      },
      { immediate: true },
    );

    const selectedResults = computed(() => {
      const arr = Array(20);
      if (!slowdown.isReady.value || !result.value) return arr;

      const listed = result.value[selectedCurr.value].listedLazy.value;
      arr.splice(0, listed.length, ...listed);
      return arr;
    });

    watch(result, () => {
      const stableTotal = result.value?.xchgStable.listed.value?.total;
      const exaltedTotal = result.value?.xchgExalted.listed.value?.total;
      if (stableTotal == null) {
        selectedCurr.value = "xchgExalted";
      } else if (exaltedTotal == null) {
        selectedCurr.value = "xchgStable";
      } else {
        selectedCurr.value =
          stableTotal > exaltedTotal ? "xchgStable" : "xchgExalted";
      }
    });

    function makeTradeLink(_have?: string[]) {
      const have =
        _have ??
        (selectedCurr.value === "xchgStable" ? ["divine"] : ["exalted"]);
      const httpPostBody = createTradeRequest(props.filters, props.item, have);
      const httpGetQuery = { exchange: httpPostBody.query };
      return `https://${getTradeEndpoint()}/trade2/exchange/poe2/${props.filters.trade.league}?q=${JSON.stringify(httpGetQuery)}`;
    }

    const { t } = useI18nNs("trade_result");

    return {
      t,
      error,
      result,
      selectedResults,
      selectedCurr,
      execSearch: () => {
        search(props.item, props.filters);
      },
      showSeller: computed(() => widget.value.showSeller),
      makeTradeLink,
      openTradeLink() {
        if (widget.value.builtinBrowser && Host.isElectron) {
          showBrowser(makeTradeLink(["mirror"]));
        } else {
          window.open(makeTradeLink(["mirror"]));
        }
      },
    };
  },
});
</script>

<style lang="postcss">
.trade-bulk-currency-icon {
  width: 1.75rem;
  height: 1.75rem;
  margin: -0.4375rem;
  margin-right: 0.125rem;
  filter: grayscale(1);
}

.trade-table-heading {
  @apply whitespace-nowrap;
}
</style>
