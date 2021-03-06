<template>
  <div id="category">
    <SfBreadcrumbs class="breadcrumbs desktop-only" :breadcrumbs="breadcrumbs">
      <template #link="{breadcrumb}">
        <router-link :to="breadcrumb.route.link">
          {{ breadcrumb.text }}
        </router-link>
      </template>
    </SfBreadcrumbs>
    <div class="navbar section">
      <div class="navbar__aside desktop-only">
        <h1 class="navbar__title">
          {{ $t("Categories") }}
        </h1>
      </div>
      <div class="navbar__main">
        <SfButton
          class="navbar__filters-button"
          @click="isFilterSidebarOpen = true"
        >
          <AIconFilter size="15px" styles="margin-right:10px" />
          {{ $t("Filters") }}
        </SfButton>
        <div class="navbar__sort desktop-only">
          <span class="navbar__label">{{ $t("Sort By") }}:</span>
          <SfSelect
            class="sort-by"
            :selected="sortOrder"
            @change="changeSortOder"
          >
            <SfSelectOption
              v-for="option in sortOptions"
              :key="option.id"
              :value="option.id"
              class="sort-by__option"
            >
              {{ option.label }}
            </SfSelectOption>
          </SfSelect>
        </div>
        <div class="navbar__counter">
          <span class="navbar__label desktop-only">
            {{ $t("Products found") }}:
          </span>
          <strong class="desktop-only">{{ getCategoryProductsTotal }}</strong>
          <span class="navbar__label mobile-only">
            {{ $t("{count} items", { count: getCategoryProductsTotal }) }}
          </span>
        </div>
        <div class="navbar__view desktop-only">
          <span>{{ $t("View") }} </span>
          <AIconViewGrid size="10px" styles="margin-left:10px" />
          <AIconViewRow size="11px" styles="margin-left:10px" />
        </div>
        <SfButton class="navbar__filters-button mobile-only">
          {{ $t("Sort By") }}
          <AIconSort size="15px" styles="margin-left:10px" />
        </SfButton>
      </div>
    </div>
    <div class="main section">
      <div class="sidebar desktop-only">
        <SfAccordion :show-chevron="false">
          <SfAccordionItem
            v-for="category in categories"
            :key="category.id"
            :header="category.name"
          >
            <SfList>
              <SfListItem v-for="item in category.items" :key="item.id">
                <router-link :to="item.link">
                  <SfMenuItem :label="item.name" :count="item.count" />
                </router-link>
              </SfListItem>
            </SfList>
          </SfAccordionItem>
        </SfAccordion>
      </div>
      <div class="products">
        <SfHeading
          v-if="isCategoryEmpty"
          :title="$t('No products found!')"
          :subtitle="
            $t(
              'Please change Your search criteria and try again. If still not finding anything relevant, please visit the Home page and try out some of our bestsellers!'
            )
          "
        />
        <template v-else>
          <lazy-hydrate :trigger-hydration="!loading">
            <div class="products__list">
              <SfProductCard
                v-for="product in products"
                :key="product.id"
                :title="product.title"
                :image="product.image"
                :regular-price="product.price.regular"
                :special-price="product.price.special"
                :max-rating="product.rating.max"
                :score-rating="product.rating.score"
                :link="product.link"
                link-tag="a"
                :is-on-wishlist="isOnWishlist(product.data)"
                class="products__product-card"
                @click:wishlist="toggleWishlist(product.data)"
              />
            </div>
          </lazy-hydrate>
          <SfPagination
            v-if="totalPages > 1"
            class="products__pagination desktop-only"
            :current="currentPage"
            :total="totalPages"
            :visible="3"
            @click="changePage"
          />
        </template>
      </div>
    </div>
    <SfSidebar
      :visible="isFilterSidebarOpen"
      @close="isFilterSidebarOpen = false"
    >
      <div class="filters">
        <template v-for="(filters, filterType) in availableFilters">
          <h3 :key="filterType" class="filters__title">
            {{ $t(filterType) }}
          </h3>
          <SfFilter
            v-for="filter in filters"
            :key="filter.id"
            :label="filter.label"
            :count="filter.count"
            :color="filter.color"
            :selected="isFilterActive(filter)"
            class="filters__item"
            @change="changeFilter(filter)"
          />
        </template>
        <div class="filters__buttons">
          <SfButton
            class="sf-button--full-width"
            @click="isFilterSidebarOpen = false"
          >
            {{ $t("Done") }}
          </SfButton>
          <SfButton
            class="sf-button--full-width filters__button-clear"
            @click="clearAllFilters"
          >
            {{ $t("Clear all") }}
          </SfButton>
        </div>
      </div>
    </SfSidebar>
  </div>
</template>

<script>
import LazyHydrate from 'vue-lazy-hydration';
import { mapGetters } from 'vuex';
import castArray from 'lodash-es/castArray';
import config from 'config';
import {
  buildFilterProductsQuery,
  productThumbnailPath,
  isServer
} from '@vue-storefront/core/helpers';
import i18n from '@vue-storefront/i18n';
import onBottomScroll from '@vue-storefront/core/mixins/onBottomScroll';
import { price, htmlDecode } from '@vue-storefront/core/filters';
import { quickSearchByQuery } from '@vue-storefront/core/lib/search';
import { getSearchOptionsFromRouteParams } from '@vue-storefront/core/modules/catalog-next/helpers/categoryHelpers';
import { catalogHooksExecutors } from '@vue-storefront/core/modules/catalog-next/hooks';
import AIconFilter from 'theme/components/atoms/a-icon-filter';
import AIconSort from 'theme/components/atoms/a-icon-sort';
import AIconViewGrid from 'theme/components/atoms/a-icon-view-grid';
import AIconViewRow from 'theme/components/atoms/a-icon-view-row';
import {
  formatCategoryLink,
  formatProductLink
} from '@vue-storefront/core/modules/url/helpers';
import {
  localizedRoute,
  currentStoreView
} from '@vue-storefront/core/lib/multistore';
import {
  SfList,
  SfButton,
  SfSelect,
  SfFilter,
  SfSidebar,
  SfHeading,
  SfMenuItem,
  SfAccordion,
  SfPagination,
  SfBreadcrumbs,
  SfProductCard
} from '@storefront-ui/vue';

const THEME_PAGE_SIZE = 12;
const LAZY_LOADING_ACTIVATION_BREAKPOINT = 1024;

const composeInitialPageState = async (store, route, forceLoad = false) => {
  try {
    const filters = getSearchOptionsFromRouteParams(route.params);
    const cachedCategory = store.getters['category-next/getCategoryFrom'](
      route.path
    );
    const currentCategory =
      cachedCategory && !forceLoad
        ? cachedCategory
        : await store.dispatch('category-next/loadCategory', { filters });
    await store.dispatch('category-next/loadCategoryProducts', {
      route,
      category: currentCategory,
      pageSize: THEME_PAGE_SIZE
    });
    const breadCrumbsLoader = store.dispatch(
      'category-next/loadCategoryBreadcrumbs',
      {
        category: currentCategory,
        currentRouteName: currentCategory.name,
        omitCurrent: true
      }
    );

    if (isServer) await breadCrumbsLoader;
    catalogHooksExecutors.categoryPageVisited(currentCategory);
  } catch (e) {
    //
  }
};

export default {
  name: 'CategoryPage',
  components: {
    LazyHydrate,
    AIconSort,
    AIconFilter,
    AIconViewRow,
    AIconViewGrid,
    SfList,
    SfButton,
    SfSelect,
    SfFilter,
    SfSidebar,
    SfHeading,
    SfMenuItem,
    SfAccordion,
    SfPagination,
    SfBreadcrumbs,
    SfProductCard
  },
  mixins: [onBottomScroll],
  data () {
    return {
      loading: true,
      loadingProducts: false,
      currentPage: 1,
      getMoreCategoryProducts: [],
      browserWidth: 0,
      isFilterSidebarOpen: false,
      unsubscribeFromStoreAction: null,
      aggregations: null
    };
  },
  computed: {
    ...mapGetters({
      getCurrentSearchQuery: 'category-next/getCurrentSearchQuery',
      getCategoryProducts: 'category-next/getCategoryProducts',
      getCurrentCategory: 'category-next/getCurrentCategory',
      getCategoryProductsTotal: 'category-next/getCategoryProductsTotal',
      getAvailableFilters: 'category-next/getAvailableFilters',
      getCurrentFilters: 'category-next/getCurrentFilters',
      getSystemFilterNames: 'category-next/getSystemFilterNames',
      getCategories: 'category-next/getCategories',
      categoryList: 'category/getCategories',
      getBreadcrumbsRoutes: 'breadcrumbs/getBreadcrumbsRoutes',
      getBreadcrumbsCurrent: 'breadcrumbs/getBreadcrumbsCurrent',
      isOnWishlist: 'wishlist/isOnWishlist'
    }),
    isLazyHydrateEnabled () {
      return config.ssr.lazyHydrateFor.includes('category-next.products');
    },
    isCategoryEmpty () {
      return this.getCategoryProductsTotal === 0;
    },
    isLazyLoadingEnabled () {
      return this.browserWidth < LAZY_LOADING_ACTIVATION_BREAKPOINT;
    },
    breadcrumbs () {
      return this.getBreadcrumbsRoutes
        .map(route => ({
          text: htmlDecode(route.name),
          route: {
            link: route.route_link
          }
        }))
        .concat({
          text: htmlDecode(this.getBreadcrumbsCurrent)
        });
    },
    categories () {
      return this.prepareCategories(this.getCategories[0].children_data);
    },
    products () {
      // lazy loading is disabled for desktop screen width (>= 1024px)
      // so products from store have to be filtered out because there could
      // be more than THEME_PAGE_SIZE of them - they could be fetched earlier
      // when lazy loading was enabled
      return this.isLazyLoadingEnabled || this.currentPage === 1
        ? this.getCategoryProducts
          .filter((product, i) => {
            return this.isLazyLoadingEnabled || i < THEME_PAGE_SIZE;
          })
          .map(this.prepareCategoryProduct)
        : this.getMoreCategoryProducts.map(this.prepareCategoryProduct);
    },
    totalPages () {
      return Math.ceil(this.getCategoryProductsTotal / THEME_PAGE_SIZE);
    },
    sortOrder () {
      return (
        this.getCurrentSearchQuery.sort ||
        `${config.products.defaultSortBy.attribute}:${config.products.defaultSortBy.order}`
      );
    },
    sortOptions () {
      return Object.entries(config.products.sortByAttributes).map(attribute => {
        const [label, id] = attribute;
        return { id, label };
      });
    },
    availableFilters () {
      return Object.entries(this.getAvailableFilters)
        .filter(([filterType, filters]) => {
          return (
            filters.length && !this.getSystemFilterNames.includes(filterType)
          );
        })
        .reduce((result, [filterType, filters]) => {
          result[`${filterType}_filter`] = filters.map(filter => ({
            ...filter,
            count: this.getFilterCount(filter) || '',
            color:
              filterType === 'color'
                ? (config.products.colorMappings &&
                    config.products.colorMappings[filter.label]) ||
                  filter.label
                : undefined
          }));
          return result;
        }, {});
    },
    isFilterActive () {
      return filter =>
        castArray(this.getCurrentFilters[filter.type]).find(
          variant => variant && variant.id === filter.id
        ) !== undefined;
    }
  },
  watch: {
    sortOrder () {
      if (this.currentPage > 1) {
        this.changePage();
      }
    }
  },
  async asyncData ({ store, route }) {
    // this is for SSR purposes to prefetch data - and it's always executed before parent component methods
    await composeInitialPageState(store, route);
  },
  async beforeRouteEnter (to, from, next) {
    if (isServer) next();
    // SSR no need to invoke SW caching here
    else if (!from.name) {
      // SSR but client side invocation, we need to cache products and invoke requests from asyncData for offline support
      next(async vm => {
        vm.loading = true;
        await composeInitialPageState(vm.$store, to, true);
        await vm.$store.dispatch('category-next/cacheProducts', { route: to }); // await here is because we must wait for the hydration
        vm.loading = false;
      });
    } else {
      // Pure CSR, with no initial category state
      next(async vm => {
        vm.loading = true;
        vm.$store.dispatch('category-next/cacheProducts', { route: to });
        vm.loading = false;
      });
    }
  },
  mounted () {
    this.unsubscribeFromStoreAction = this.$store.subscribeAction(action => {
      if (action.type === 'category-next/loadAvailableFiltersFrom') {
        this.aggregations = action.payload.aggregations;
      }
    });
    this.$bus.$on('product-after-list', this.initPagination);
    window.addEventListener('resize', this.getBrowserWidth);
    this.getBrowserWidth();
  },
  beforeDestroy () {
    this.unsubscribeFromStoreAction();
    this.$bus.$off('product-after-list', this.initPagination);
    window.removeEventListener('resize', this.getBrowserWidth);
  },
  methods: {
    toggleWishlist (product) {
      const isProductOnWishlist = this.isOnWishlist(product);
      const message = isProductOnWishlist
        ? 'Product {productName} has been removed from wishlist!'
        : 'Product {productName} has been added to wishlist!';
      const action = isProductOnWishlist
        ? 'wishlist/removeItem'
        : 'wishlist/addItem';

      this.$store.dispatch(action, product);
      this.$store.dispatch(
        'notification/spawnNotification',
        {
          type: 'success',
          message: i18n.t(message, { productName: htmlDecode(product.name) }),
          action1: { label: i18n.t('OK') }
        },
        { root: true }
      );
    },
    getBrowserWidth () {
      return (this.browserWidth = window.innerWidth);
    },
    async onBottomScroll () {
      if (!this.isLazyLoadingEnabled || this.loadingProducts) {
        return;
      }

      this.loadingProducts = true;
      await this.$store.dispatch('category-next/loadMoreCategoryProducts');
      this.loadingProducts = false;
    },
    async changePage (page = this.currentPage) {
      const start = (page - 1) * THEME_PAGE_SIZE;

      if (
        start < 0 ||
        start >= this.getCategoryProductsTotal ||
        this.getCategoryProductsTotal < THEME_PAGE_SIZE
      ) {
        return;
      }

      const { includeFields, excludeFields } = config.entities.productList;
      const { filters } = this.getCurrentSearchQuery;
      const filterQuery = buildFilterProductsQuery(
        this.getCurrentCategory,
        filters
      );

      const searchResult = await quickSearchByQuery({
        query: filterQuery,
        sort: this.sortOrder,
        start: start,
        size: THEME_PAGE_SIZE,
        includeFields: includeFields,
        excludeFields: excludeFields
      });

      this.getMoreCategoryProducts = await this.$store.dispatch(
        'category-next/processCategoryProducts',
        {
          products: searchResult.items,
          filters: filters
        }
      );

      this.currentPage = page;
    },
    initPagination () {
      this.currentPage = 1;
    },
    prepareCategories (categories, firstItem = []) {
      return categories
        ? categories
          .reduce((result, subCategory) => {
            const category = this.categoryList.find(
              c => c.id === subCategory.id
            );

            if (!category || !category.is_active) {
              return result;
            }

            return result.concat({
              id: category.id,
              name: category.name,
              link: formatCategoryLink(category),
              count: String(category.product_count),
              position: category.position,
              items: this.prepareCategories(subCategory.children_data, [
                {
                  id: category.id,
                  name: i18n.t('View all'),
                  link: formatCategoryLink(category),
                  count: String(category.product_count),
                  position: 0
                }
              ])
            });
          }, firstItem)
          .sort((a, b) => a.position - b.position)
        : firstItem;
    },
    prepareCategoryProduct (product) {
      return {
        data: product,
        id: product.id,
        title: htmlDecode(product.name),
        image: this.getThumbnail(
          productThumbnailPath(product),
          config.products.thumbnails.width,
          config.products.thumbnails.height
        ),
        link: formatProductLink(product, currentStoreView().storeCode),
        price: {
          regular: price(parseFloat(product.priceInclTax)),
          special: price(parseFloat(product.specialPriceInclTax))
        },
        rating: {
          max: 5,
          score: 5
        }
      };
    },
    changeSortOder (sortOrder) {
      if (this.getCurrentSearchQuery.sort !== sortOrder) {
        this.$store.dispatch('category-next/switchSearchFilters', [
          { id: sortOrder, type: 'sort' }
        ]);
      }
    },
    changeFilter (filter) {
      this.$store.dispatch('category-next/switchSearchFilters', [filter]);
    },
    clearAllFilters () {
      this.$store.dispatch('category-next/resetSearchFilters');
    },
    getFilterCount (filter) {
      const aggregations = [
        `agg_range_${filter.type}`,
        `agg_terms_${filter.type}`,
        `agg_terms_${filter.type}_options`
      ];

      return aggregations
        .reduce((result, aggregation) => {
          const bucket =
            this.aggregations &&
            this.aggregations[aggregation] &&
            this.aggregations[aggregation].buckets.find(
              bucket => String(bucket.key) === String(filter.id)
            );

          return bucket ? result + bucket.doc_count : result;
        }, 0);
    }
  },
  metaInfo () {
    const storeView = currentStoreView();
    const {
      meta_title,
      meta_description,
      name,
      slug
    } = this.getCurrentCategory;
    const meta = meta_description
      ? [
        {
          vmid: 'description',
          name: 'description',
          content: htmlDecode(meta_description)
        }
      ]
      : [];
    const categoryLocaliedLink = localizedRoute(
      {
        name: 'category-amp',
        params: { slug }
      },
      storeView.storeCode
    );
    const ampCategoryLink = this.$router.resolve(categoryLocaliedLink).href;

    return {
      link: [{ rel: 'amphtml', href: ampCategoryLink }],
      title: htmlDecode(meta_title || name),
      meta
    };
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

#category {
  box-sizing: border-box;
  @include for-desktop {
    max-width: 1240px;
    margin: auto;
  }
}
.breadcrumbs {
  padding: $spacer-big $spacer-extra-big $spacer-extra-big;
}
.main {
  display: flex;
}
.navbar {
  position: relative;
  display: flex;
  @include for-desktop {
    border-top: 1px solid $c-light;
    border-bottom: 1px solid $c-light;
  }
  &::after {
    position: absolute;
    bottom: 0;
    left: $spacer-big;
    width: calc(100% - (#{$spacer-big} * 2));
    height: 1px;
    background-color: $c-light;
    content: "";
    @include for-desktop {
      content: none;
    }
  }
  &__aside {
    display: flex;
    align-items: center;
    flex: 0 0 15%;
    padding: $spacer-big $spacer-extra-big;
    border-right: 1px solid $c-light;
  }
  &__main {
    flex: 1;
    display: flex;
    align-items: center;
    padding: $spacer-medium 0;
    font-size: $font-size-small-desktop;
    @include for-desktop {
      padding: $spacer-big 0;
    }
  }
  &__title {
    padding: 0;
    font-size: $font-size-big-desktop;
    line-height: 2.23;
  }
  &__filters-button {
    display: flex;
    align-items: center;
    margin: 0;
    padding: 0;
    background: transparent;
    color: inherit;
    font-size: inherit;
    font-weight: 500;
    @include for-desktop {
      margin: 0 0 0 $spacer-extra-big;
      font-weight: 400;
      text-transform: none;
    }
    svg {
      fill: $c-dark;
      @include for-desktop {
        fill: $c-gray-variant;
      }
    }
    &:hover {
      color: $c-primary;
      svg {
        fill: $c-primary;
      }
    }
  }
  &__label {
    color: $c-gray-variant;
  }
  &__sort {
    display: flex;
    align-items: center;
    margin-left: $spacer-extra-big;
    margin-right: auto;
  }
  &__counter {
    margin: auto;
    @include for-desktop {
      margin-right: 0;
    }
  }
  &__view {
    display: flex;
    align-items: center;
    margin: 0 $spacer-extra-big;
    &-icon {
      margin-left: 10px;
    }
  }
}

.products {
  box-sizing: border-box;
  flex: 1;
  margin: 0 -#{$spacer};
  @include for-desktop {
    margin: $spacer-big;
  }
  &__list {
    display: flex;
    flex-wrap: wrap;
    margin-top: 1.875rem - 0.5rem;
  }
  &__product-card {
    flex: 0 0 50%;
    padding: $spacer;
    @include for-desktop {
      flex: 0 0 25%;
      padding: $spacer-big;
    }
  }
  &__pagination {
    @include for-desktop {
      display: flex;
      justify-content: center;
      margin-top: $spacer-extra-big;
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
.sidebar {
  flex: 0 0 15%;
  padding: $spacer-extra-big;
  border-right: 1px solid $c-light;
}
.sort-by {
  flex: unset;
  width: 190px;
  padding: 0 10px;
  font-size: inherit;
  &__option {
    padding: 10px;
    font-size: inherit;
  }
}
.filters {
  &__title {
    margin: $spacer-big * 3 0 $spacer-big;
    font-size: $font-size-big-desktop;
    line-height: 1.6;
    &:first-child {
      margin: 0 0 $spacer-big 0;
    }
  }
  &__item {
    padding: $spacer-small 0;
  }
  &__buttons {
    margin: $spacer-big * 3 0 0 0;
  }
  &__button-clear {
    color: #a3a5ad;
    margin-top: 10px;
    background-color: $c-light;
  }
}
</style>
