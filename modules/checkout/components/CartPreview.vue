<template>
  <div>
    <div class="highlighted">
      <SfHeading
        :level="3"
        :title="$t('Order summary')"
        class="sf-heading--left sf-heading--no-underline title"
      />
    </div>
    <div class="highlighted">
      <SfProperty
        :name="$t('Products')"
        :value="totalItems"
        class="sf-property--full-width sf-property--large property"
      />
      <SfProperty
        :name="$t('Subtotal')"
        :value="$fc(totals.subtotal)"
        :class="['sf-property--full-width', 'sf-property--large property']"
      />
      <SfProperty
        v-if="hasDiscounts"
        :name="$t('Discount')"
        :value="$fc(discount)"
        class="sf-property--full-width sf-property--small property"
      />
      <SfProperty
        v-if="selectedShippingMethod"
        :name="$t('Shipping')"
        :value="$fc(getShippingMethodPrice(selectedShippingMethod))"
        class="sf-property--full-width sf-property--large property"
      />

      <SfProperty
        v-if="donmoDonation"
        :name="$t('Donmo Donation')"
        :value="$fc(donmoDonation)"
        :class="['sf-property--full-width', 'sf-property--large property']"
      />

      <SfProperty
        :name="$t('Total')"
        :value="$fc(totals.total)"
        class="sf-property--full-width sf-property--large property-total"
      />
    </div>
    <CouponCode class="highlighted" />
    <div id="donmo-roundup"></div>

    <div class="highlighted">
      <SfCharacteristic
        v-for="characteristic in characteristics"
        :key="characteristic.title"
        :title="characteristic.title"
        :description="characteristic.description"
        :icon="characteristic.icon"
        class="characteristic"
      />
    </div>
  </div>
</template>
<script lang="ts">
import { SfHeading, SfProperty, SfCharacteristic } from "@storefront-ui/vue";
import {
  computed,
  ref,
  defineComponent,
  onMounted,
  useContext,
  watch,
} from "@nuxtjs/composition-api";
import cartGetters from "~/modules/checkout/getters/cartGetters";
import useCart from "~/modules/checkout/composables/useCart";
import getShippingMethodPrice from "~/helpers/checkout/getShippingMethodPrice";
import CouponCode from "../../../components/CouponCode.vue";

import { useApi } from "~/composables/useApi";

const CHARACTERISTICS = [
  {
    title: "Safety",
    description: "It carefully packaged with a personal touch",
    icon: "safety",
  },
  {
    title: "Easy shipping",
    description: "You’ll receive dispatch confirmation and an arrival date",
    icon: "shipping",
  },
  {
    title: "Changed your mind?",
    description: "Rest assured, we offer free returns within 30 days",
    icon: "return",
  },
];

export default defineComponent({
  name: "CartPreview",
  components: {
    SfHeading,
    SfProperty,
    SfCharacteristic,
    CouponCode,
  },
  setup() {
    const { cart, removeItem, updateItemQty, load } = useCart();

    const listIsHidden = ref(false);
    const products = computed(() => cartGetters.getItems(cart.value));
    const totalItems = computed(() => cartGetters.getTotalItems(cart.value));
    const totals = computed(() => cartGetters.getTotals(cart.value));
    const discount = computed(() => -cartGetters.getDiscountAmount(cart.value));
    const donmoDonation = computed(
      () => cart.value?.prices?.["donmo_donation"]?.value
    );
    const hasDiscounts = computed(() => Math.abs(discount.value) > 0);
    const selectedShippingMethod = computed(() =>
      cartGetters.getSelectedShippingMethod(cart.value)
    );

    const { mutate } = useApi();
    const {
      app: { i18n },
    } = useContext();

    onMounted(() => {
      load().then(() => {
        const donmo = (window as any).DonmoRoundup({
          publicKey: process.env.DONMO_PUBLIC_KEY,
          isBackendBased: true,
          language: i18n.locale,
          orderId: cart.value.id,
          addDonationAction: async ({ donationAmount }) => {
            const ADD_DONATION_MUTATION = `
              mutation ADD_DONATION_MUTATION($donationAmount: Float!, $cartId: String!) {
                addDonationToQuote(donationAmount: $donationAmount, cartId: $cartId){
                  message
                }
              }
            `;

            await mutate(ADD_DONATION_MUTATION, {
              donationAmount,
              cartId: cart.value.id,
            });

            // refresh cart
            load();
          },
          removeDonationAction: async () => {
            const REMOVE_DONATION_MUTATION = `
              mutation REMOVE_DONATION_MUTATION($cartId: String!) {
                removeDonationFromQuote(cartId: $cartId) {
                  message
                }
              }
            `;

            await mutate(REMOVE_DONATION_MUTATION, { cartId: cart.value.id });

            // refresh cart
            load();
          },
          getExistingDonation: () => {
            return cart.value?.prices?.["donmo_donation"]?.value;
          },
          getGrandTotal: () => cartGetters.getTotals(cart.value).total,
        });

        donmo.build();

        watch(
          () => cartGetters.getTotals(cart.value).total,
          () => {
            load().then(() => {
              donmo.refresh();
            });
          }
        );
      });
    });
    return {
      cart,
      discount,
      donmoDonation,
      hasDiscounts,
      totalItems,
      listIsHidden,
      products,
      totals,
      removeItem,
      updateItemQty,
      cartGetters,
      getShippingMethodPrice,
      characteristics: CHARACTERISTICS,
      selectedShippingMethod,
    };
  },

  head() {
    return {
      title: "Donmo Roundup", // Other meta information
      script: [
        {
          hid: "donmo",
          src: "https://static.donmo.org/integration.js",
          defer: true,
        },
      ],
    };
  },
});
</script>

<style lang="scss" scoped>
#donmo-roundup {
  margin-top: 5px;
  margin-bottom: 15px;
}
.highlighted {
  box-sizing: border-box;
  width: 100%;
  background-color: var(--c-light);
  padding: var(--spacer-xl) var(--spacer-xl) 0;

  &:last-child {
    padding-bottom: var(--spacer-xl);
  }
}

.total-items {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--spacer-xl);
}

.property {
  margin-bottom: var(--spacer-base);
}

.property-total {
  margin-top: var(--spacer-xl);
  padding-top: var(--spacer-lg);
  font-size: var(--font-size-xl);
  border-top: var(--c-white) 1px solid;
  --property-name-font-weight: var(--font-weight--semibold);
  --property-name-color: var(--c-text);
}

.characteristic {
  &:not(:last-child) {
    margin-bottom: var(--spacer-base);
  }
}
</style>
