<template>
  <div :id="elementId"></div>
</template>

<script lang="ts">
import {
  defineComponent,
  onMounted,
  useContext,
  watch,
  ref,
} from "@nuxtjs/composition-api";

import { useApi } from "~/composables/useApi";
import useCart from "~/modules/checkout/composables/useCart";
import cartGetters from "~/modules/checkout/getters/cartGetters";

export default defineComponent({
  name: "DonmoRoundupComponent",
  props: {
    publicKey: {
      type: String,
      required: true,
    },

    integrationTitle: {
      type: String,
    },
    roundupMessage: {
      type: String,
    },
    thankMessage: {
      type: String,
    },
    errorMessage: {
      type: String,
    },
    language: {
      type: String,
    },
    isBackendBased: {
      type: Boolean,
      default: true,
    },
  },
  setup(props) {
    const { mutate } = useApi();
    const { cart, load } = useCart();
    const { app } = useContext();

    const addDonationAction = async ({ donationAmount }) => {
      const ADD_DONATION_MUTATION = `
              mutation ADD_DONATION_MUTATION($donationAmount: Float!, $cartId: String!) {
                addDonationToCart(input: {
                  donation_amount: $donationAmount,
                  cart_id: $cartId
                }){
                  message
                }
              }
            `;

      const { errors } = await mutate(ADD_DONATION_MUTATION, {
        donationAmount,
        cartId: cart.value.id,
      });

      if (errors) {
        throw new Error("Failed to add donation");
      }
      // refresh cart
      load();
    };

    const removeDonationAction = async () => {
      const REMOVE_DONATION_MUTATION = `
              mutation REMOVE_DONATION_MUTATION($cartId: String!) {
                removeDonationFromCart(cart_id: $cartId) {
                  message
                }
              }
            `;

      const { errors } = await mutate(REMOVE_DONATION_MUTATION, {
        cartId: cart.value.id,
      });

      if (errors) {
        throw new Error("Failed to remove donation");
      }
      // refresh cart
      load();
    };

    const getExistingDonation = () => {
      return cart.value?.prices?.["donmo_donation"]?.value;
    };

    const getGrandTotal = () => cartGetters.getTotals(cart.value).total;

    const elementId = ref(null);

    onMounted(() => {
      elementId.value = "donmo-roundup-" + Math.floor(Math.random() * 1000);

      load().then(() => {
        const donmo = (window as any).DonmoRoundup({
          publicKey: props.publicKey,
          elementId: elementId.value,
          isBackendBased: props.isBackendBased,
          language: props.language || app.i18n.locale,
          currency: app.$vsf.$magento.config.state.getCurrency(),
          orderId: cart.value.id,
          addDonationAction,
          removeDonationAction,
          getExistingDonation,
          getGrandTotal,

          integrationTitle: props.integrationTitle,
          roundupMessage: props.roundupMessage,
          thankMessage: props.thankMessage,
          errorMessage: props.errorMessage,
        });

        donmo.build();

        watch(
          () => cartGetters.getTotals(cart.value).total,
          () =>
            load().then(() => {
              if (cartGetters.getTotals(cart.value).total) {
                donmo.refresh();
              }
            })
        );
      });
    });

    return {
      elementId,
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
