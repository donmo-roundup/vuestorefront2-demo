<template>
  <div>
    <IconSprite />
    <CartSidebar v-if="isCartSidebarOpen" />
    <WishlistSidebar v-if="isWishlistSidebarOpen" />
    <LoginModal v-if="isLoginModalOpen" @close="toggleLoginModal" />
    <LazyHydrate when-visible>
      <Notification />
    </LazyHydrate>
    <div class="donmo-header">
      <a href="https://donmo.org" target="_blank">Donmo Vue Storefront Demo</a>
    </div>
    <TopBar class="desktop-only" />
    <AppHeader />
    <div id="layout">
      <nuxt :key="route.fullPath" />
    </div>
    <BottomNavigation />
    <LoadWhenVisible>
      <AppFooter />
    </LoadWhenVisible>
  </div>
</template>
<script lang="ts">
import LazyHydrate from "vue-lazy-hydration";
import { useRoute, defineComponent, onMounted } from "@nuxtjs/composition-api";
import { useUiState } from "~/composables";
import AppHeader from "~/components/AppHeader.vue";
import BottomNavigation from "~/components/BottomNavigation.vue";
import IconSprite from "~/components/General/IconSprite.vue";
import LoadWhenVisible from "~/components/utils/LoadWhenVisible.vue";
import TopBar from "~/components/TopBar/TopBar.vue";

import { Product } from "~/modules/catalog/product/types";
import { useProduct } from "~/modules/catalog/product/composables/useProduct";
import useCart from "~/modules/checkout/composables/useCart";
import useShipping from "~/modules/checkout/composables/useShipping";
import useShippingProvider from "~/modules/checkout/composables/useShippingProvider";
import useBilling from "~/modules/checkout/composables/useBilling";

export default defineComponent({
  name: "DefaultLayout",

  components: {
    LoadWhenVisible,
    LazyHydrate,
    AppHeader,
    BottomNavigation,
    IconSprite,
    TopBar,
    AppFooter: () =>
      import(/* webpackPrefetch: true */ "~/components/AppFooter.vue"),
    CartSidebar: () =>
      import(
        /* webpackPrefetch: true */ "~/modules/checkout/components/CartSidebar.vue"
      ),
    WishlistSidebar: () =>
      import(
        /* webpackPrefetch: true */ "~/modules/wishlist/components/WishlistSidebar.vue"
      ),
    LoginModal: () =>
      import(
        /* webpackPrefetch: true */ "~/modules/customer/components/LoginModal/LoginModal.vue"
      ),
    Notification: () =>
      import(/* webpackPrefetch: true */ "~/components/Notification.vue"),
  },

  setup() {
    onMounted(async () => {
      const { getProductDetails } = useProduct();
      const { addItem, cart, loadTotalQty } = useCart();

      const { save: saveShipping } = useShipping();

      const { save: saveBilling } = useBilling();

      const { save: saveShippingProvider } = useShippingProvider();

      await loadTotalQty();

      const shippingDetails = {
        firstname: "Vanessa",
        lastname: "Lombardi",
        city: "San Diego",
        street: "123 Kettner Boulevard",
        postcode: "12345",
        company: "Alpha Company",
        telephone: "1234567890",
        country_code: "US",
        apartment: "12",
        region_id: "12",
      };
      if (!cart.value?.total_quantity) {
        const result = await getProductDetails({
          filter: {
            sku: {
              eq: "24-MB01",
            },
          },
        });

        const product = result.items[0] as Product;

        await addItem({ product, quantity: 1 });

        // add acount data
        localStorage.setItem(
          "vsf-checkout",
          JSON.stringify({
            "user-account": {
              firstname: "Vanessa",
              lastname: "Lombardi",
              email: "vanessa@lombardi.com",
              is_subscribed: false,
            },
          })
        );

        // add shipping data
        await saveShipping({
          shippingDetails,
        });

        await saveShippingProvider({
          shippingMethod: {
            carrier_code: "flatrate",
            method_code: "flatrate",
          },
        });

        saveBilling({
          billingDetails: {
            ...shippingDetails,
            customerAddressId: null,
            sameAsShipping: false,
          },
        });
      }
    });
    const route = useRoute();
    const {
      isCartSidebarOpen,
      isWishlistSidebarOpen,
      isLoginModalOpen,
      toggleLoginModal,
    } = useUiState();

    return {
      isCartSidebarOpen,
      isWishlistSidebarOpen,
      isLoginModalOpen,
      toggleLoginModal,
      route,
    };
  },
  head: {
    link: [{ rel: "stylesheet", href: "/_nuxt/fonts.css" }],
  },
});
</script>

<style lang="scss">
@import "~@storefront-ui/vue/styles";

.donmo-header {
  background-color: #f6c728;
  padding: 5px;
}
.donmo-header a {
  color: #005cb3;
  font-weight: bold;
}

#layout {
  box-sizing: border-box;
  @include for-desktop {
    max-width: 1270px;
    margin: auto;
  }
}

.no-scroll {
  overflow: hidden;
  height: 100vh;
}

// Reset CSS
html {
  width: auto;
  @include for-mobile {
    overflow-x: hidden;
  }
}

body {
  overflow-x: hidden;
  color: var(--c-text);
  font-size: var(--font-size--base);
  font-family: var(--font-family--primary);
  margin: 0;
  padding: 0;
}

a {
  text-decoration: none;
  color: var(--c-link);

  &:hover {
    color: var(--c-link-hover);
  }
}

h1 {
  font-family: var(--font-family--secondary);
  font-size: var(--h1-font-size);
  line-height: 1.6;
  margin: 0;
}

h2 {
  font-family: var(--font-family--secondary);
  font-size: var(--h2-font-size);
  line-height: 1.6;
  margin: 0;
}

h3 {
  font-family: var(--font-family--secondary);
  font-size: var(--h3-font-size);
  line-height: 1.6;
  margin: 0;
}

h4 {
  font-family: var(--font-family--secondary);
  font-size: var(--h4-font-size);
  line-height: 1.6;
  margin: 0;
}
</style>
