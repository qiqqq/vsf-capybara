<template>
  <div id="home">
    <SfHero class="section">
      <SfHeroItem
        v-for="(hero, i) in heroes"
        :key="i"
        :title="hero.title"
        :subtitle="hero.subtitle"
        :image="hero.image"
      />
    </SfHero>

    <SfBannerGrid :banner-grid="1" class="banners section">
      <template v-for="(banner, i) in banners" v-slot:[banner.slot]>
        <router-link :key="i" :to="banner.link">
          <SfBanner
            :subtitle="banner.subtitle"
            :title="banner.title"
            :description="banner.description"
            :button-text="banner.buttonText"
            :image="banner.image"
            class="sf-banner--slim"
          />
        </router-link>
      </template>
    </SfBannerGrid>

    <SfCallToAction
      title="Subscribe to Newsletters"
      description="Be aware of upcoming sales and events. Receive gifts and special offers!"
      image="/assets/newsletter.png"
      class="call-to-action-newsletter"
    >
      <template #button>
        <SfButton @click="showNewsletterPopup">
          {{ $t("Subscribe") }}
        </SfButton>
      </template>
    </SfCallToAction>

    <newsletter-popup v-if="loadNewsletterPopup" />

    <SfSection title-heading="Best Sellers" class="section">
      <lazy-hydrate :trigger-hydration="!loading">
        <m-product-carousel :products="newCollection" />
      </lazy-hydrate>
    </SfSection>

    <SfSection
      v-if="isOnline"
      title-heading="Share Your Look"
      subtitle-heading="#YOURLOOK"
      class="section"
    >
      <AImagesGrid :images="dummyInstaImages" />
    </SfSection>
    <Onboard />
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import LazyHydrate from 'vue-lazy-hydration';
import { Logger } from '@vue-storefront/core/lib/logger';
import Home from '@vue-storefront/core/pages/Home';
import Onboard from 'theme/components/theme/blocks/Home/Onboard';
import { registerModule } from '@vue-storefront/core/lib/modules';
import { RecentlyViewedModule } from '@vue-storefront/core/modules/recently-viewed';
import { Wishlist } from '@vue-storefront/core/modules/wishlist/components/Wishlist';
import { isServer, onlineHelper } from '@vue-storefront/core/helpers';
import MProductCarousel from 'theme/components/molecules/m-product-carousel';
import AImagesGrid from 'theme/components/atoms/a-images-grid';

import {
  SfHero,
  SfButton,
  SfBanner,
  SfSection,
  SfBannerGrid,
  SfCallToAction
} from '@storefront-ui/vue';

const NewsletterPopup = () =>
  import(
    /* webpackChunkName: "vsf-newsletter-modal" */ 'theme/components/core/NewsletterPopup'
  );

export default {
  components: {
    Onboard,
    LazyHydrate,
    NewsletterPopup,
    SfHero,
    SfButton,
    SfBanner,
    SfSection,
    SfBannerGrid,
    SfCallToAction,
    MProductCarousel,
    AImagesGrid
  },
  mixins: [Home, Wishlist],
  data () {
    return {
      loading: true,
      loadNewsletterPopup: false,
      dummyInstaImages: [
        {
          mobile: { url: `/assets/ig/ig01.jpg` },
          desktop: { url: `/assets/ig/ig01.jpg` }
        },
        {
          mobile: { url: `/assets/ig/ig02.jpg` },
          desktop: { url: `/assets/ig/ig02.jpg` }
        },
        {
          mobile: { url: `/assets/ig/ig03.jpg` },
          desktop: { url: `/assets/ig/ig03.jpg` }
        },
        {
          mobile: { url: `/assets/ig/ig04.jpg` },
          desktop: { url: `/assets/ig/ig04.jpg` }
        },
        {
          mobile: { url: `/assets/ig/ig05.jpg` },
          desktop: { url: `/assets/ig/ig05.jpg` }
        },
        {
          mobile: { url: `/assets/ig/ig06.jpg` },
          desktop: { url: `/assets/ig/ig06.jpg` }
        }
      ]
    };
  },
  computed: {
    ...mapGetters({
      isLoggedIn: 'user/isLoggedIn',
      heroImage: 'promoted/getHeadImage',
      promotedOffers: 'promoted/getPromotedOffers',
      newCollection: 'homepage/getEverythingNewCollection'
    }),
    isOnline () {
      return onlineHelper.isOnline;
    },
    heroes () {
      const hero = {
        ...this.heroImage
      };

      return [hero, hero, hero];
    },
    banners () {
      const slots = ['bannerA', 'bannerB', 'bannerC', 'bannerD'];

      return this.promotedOffers.mainBanners.reduce((result, banner, i) => {
        if (slots[i]) {
          banner.slot = slots[i];
          result.push(banner);
        }

        return result;
      }, []);
    }
  },
  watch: {
    isLoggedIn () {
      const redirectObj = localStorage.getItem('redirect');
      if (redirectObj) this.$router.push(redirectObj);
      localStorage.removeItem('redirect');
    }
  },
  async asyncData ({ store }) {
    Logger.info('Calling asyncData in Home (theme)')();

    await Promise.all([
      store.dispatch('homepage/fetchNewCollection'),
      store.dispatch('promoted/updateHeadImage'),
      store.dispatch('promoted/updatePromotedOffers')
    ]);
  },
  beforeCreate () {
    registerModule(RecentlyViewedModule);
  },
  async beforeMount () {
    if (this.$store.state.__DEMO_MODE__) {
      const onboardingClaim = await this.$store.dispatch('claims/check', {
        claimCode: 'onboardingAccepted'
      });

      if (!onboardingClaim) {
        this.$bus.$emit('modal-toggle', 'modal-onboard');
        this.$store.dispatch('claims/set', {
          claimCode: 'onboardingAccepted',
          value: true
        });
      }
    }
  },
  mounted () {
    if (!this.isLoggedIn && localStorage.getItem('redirect')) { this.$bus.$emit('modal-show', 'modal-signup'); }
  },
  beforeRouteEnter (to, from, next) {
    if (!isServer && !from.name) {
      next(vm => {
        vm.$store.dispatch('homepage/fetchNewCollection').then(() => {
          vm.loading = false;
        });
      });
    } else {
      next();
    }
  },
  methods: {
    showNewsletterPopup () {
      this.loadNewsletterPopup = true;
      this.$bus.$emit('modal-show', 'modal-newsletter');
    }
  }
};
</script>

<style lang="scss" scoped>
@import "~@storefront-ui/shared/styles/_variables.scss";

@mixin for-desktop {
  @media screen and (min-width: $desktop-min) {
    @content;
  }
}

#home {
  box-sizing: border-box;
  @include for-desktop {
    max-width: 1240px;
    margin: auto;
  }
}

.call-to-action-newsletter {
  margin: $spacer-big 0;
  box-sizing: border-box;
  @include for-desktop {
    margin: $spacer-extra-big * 2 0;
  }
}

.sf-hero-item {
  background-position: top;
}

.sf-banner-sf-banner-grid {
  margin: $spacer-big 0;
  @include for-desktop {
    margin: $spacer-extra-big 0;
  }
}

.banners {
  margin: $spacer-big 0;
  @include for-desktop {
    margin: $spacer-extra-big 0;
  }
}

.product-card {
  max-width: unset;
  &:hover {
    @include for-desktop {
      box-shadow: 0px 4px 20px rgba(168, 172, 176, 0.19);
    }
  }
}

.product-carousel {
  margin: -20px -#{$spacer-big} -20px 0;
  @include for-desktop {
    margin: -20px 0;
  }
  ::v-deep .sf-carousel__wrapper {
    padding: 20px 0;
    @include for-desktop {
      padding: 20px;
      max-width: calc(100% - 216px);
    }
  }
}

.section {
  padding-left: $spacer-big;
  padding-right: $spacer-big;
  @include for-desktop {
    padding-left: 0;
    padding-right: 0;
  }
}
</style>
