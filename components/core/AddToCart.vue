<template>
  <button-full
    :disabled="isProductDisabled"
    data-testid="addToCart"
    @click.native="addToCart(product)"
  >
    {{ $t("Add to cart") }}
  </button-full>
</template>

<script>
import { formatProductMessages } from '@vue-storefront/core/filters/product-messages';
import { notifications } from '@vue-storefront/core/modules/cart/helpers';
import { mapGetters } from 'vuex';

export default {
  props: {
    product: {
      required: true,
      type: Object
    },
    disabled: {
      type: Boolean,
      default: false
    }
  },
  computed: {
    ...mapGetters({
      isAddingToCart: 'cart/getIsAdding'
    }),
    isProductDisabled () {
      return (
        this.disabled ||
        formatProductMessages(this.product.errors) !== '' ||
        this.isAddingToCart
      );
    }
  },
  beforeMount () {
    this.$bus.$on('product-after-removevariant', this.onAfterRemovedVariant);
  },
  beforeDestroy () {
    this.$bus.$off('product-after-removevariant');
  },
  methods: {
    onAfterRemovedVariant () {
      this.$forceUpdate();
    },
    async addToCart () {
      try {
        const diffLog = await this.$store.dispatch('cart/addItem', {
          productToAdd: this.product
        });
        diffLog.clientNotifications.forEach(notificationData => {
          this.$store.dispatch(
            'notification/spawnNotification',
            notificationData,
            { root: true }
          );
        });
      } catch (message) {
        this.$store.dispatch(
          'notification/spawnNotification',
          notifications.createNotification({ type: 'error', message }),
          { root: true }
        );
      }
    }
  }
};
</script>
