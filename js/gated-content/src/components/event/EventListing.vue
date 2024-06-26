<template>
  <div class="gated-containerV2 my-40-20 px--20-10">
    <div class="listing-header" :class="{'with-date-filter': withDateFilter}">
      <h2 class="title text-gray">{{ title }}</h2>
      <h2 class="videos__date-filter"
          v-if="withDateFilter"
      >
        <button v-on:click.stop="backOneDay" class="left" role="button"
                :style="[hidePrevDateButton ? {'visibility':'hidden'}:'']"
                aria-label="previous date"><i class="fa fa-angle-left"></i></button>
        <span class="date" v-cloak>{{ dateFormatted }}</span>
        <button v-on:click.stop="forwardOneDay" class="right"
                role="button" aria-label="next date"><i class="fa fa-angle-right"></i></button>
      </h2>
      <template v-if="hasMoreItems">
        <router-link :to="{ name: 'Schedule' }" v-if="viewAll" class="view-all">
          More
        </router-link>
        <slot name="filterButton"></slot>
      </template>
    </div>
    <div v-if="loading" class="text-center">
      <Spinner></Spinner>
    </div>
    <template v-else-if="listingIsNotEmpty">
      <div v-if="error">Error loading</div>
      <div v-else :class="layoutClass">
        <EventTeaser
          v-for="video in listing"
          :key="video.id"
          :video="video"
        />
      </div>
    </template>
    <div v-else class="empty-listing">{{ emptyBlockMsg }}</div>
    <Pagination
      v-if="pagination"
      :links="links"
    ></Pagination>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import client from '@/client';
import EventTeaser from '@/components/event/EventTeaser.vue';
import Spinner from '@/components/Spinner.vue';
import Pagination from '@/components/Pagination.vue';
import { JsonApiCombineMixin } from '@/mixins/JsonApiCombineMixin';
import { FavoritesMixin } from '@/mixins/FavoritesMixin';
import { ListingMixin } from '@/mixins/ListingMixin';

export default {
  name: 'EventListing',
  mixins: [JsonApiCombineMixin, FavoritesMixin, ListingMixin],
  components: {
    EventTeaser,
    Spinner,
    Pagination,
  },
  props: {
    title: {
      type: String,
      default: 'Live streams',
    },
    eventType: {
      type: String,
      default: 'live_stream',
    },
    excludedVideoId: {
      type: String,
      default: '',
    },
    withDateFilter: {
      type: Boolean,
      default: false,
    },
    featured: {
      type: Boolean,
      default: false,
    },
    categories: {
      type: Array,
      default: null,
    },
    instructor: {
      type: String,
      default: '',
    },
    sort: {
      type: Object,
      default() {
        return { path: 'date.value', direction: 'ASC' };
      },
    },
    msg: {
      String,
      default: 'Events not found.',
    },
  },
  data() {
    return {
      component: this.eventType,
      loading: true,
      error: false,
      params: [
        'field_ls_image',
        'field_ls_image.field_media_image',
        'field_ls_level',
        'field_gc_instructor_reference',
        'image',
        'image.field_media_image',
        'level',
        'instructor_reference',
      ],
      date: new Date(),
      oneDay: 86400000,
    };
  },
  watch: {
    $route: 'load',
    excludedVideoId: 'load',
    date: 'load',
    eventType: 'load',
    sort: 'load',
    isCategoriesLoaded() {
      if (this.categories !== null) {
        this.load();
      }
    },
  },
  async mounted() {
    // By default emit that listing not empty to the parent component.
    this.$emit('listing-not-empty', true);
    await this.load();
  },
  computed: {
    ...mapGetters([
      'isCategoriesLoaded',
    ]),
    dateFormatted() {
      const weekDay = this.date.toLocaleDateString('en', { weekday: 'long' });
      const monthName = this.date.toLocaleDateString('en', { month: 'long' });
      return `${monthName} ${this.date.getDate()}, ${weekDay}`;
    },
    hidePrevDateButton() {
      const isToday = (someDate) => {
        const today = new Date();
        return (someDate.getDate() === today.getDate()
          && someDate.getMonth() === today.getMonth()
          && someDate.getFullYear() === today.getFullYear());
      };

      return isToday(this.date);
    },
  },
  methods: {
    async load() {
      this.loading = true;
      const params = {};
      if (this.params) {
        params.include = this.params.join(',');
      }

      params.filter = {
        dateFilter: {
          condition: {
            path: 'date.end_value',
            operator: '>=',
            value: new Date().toISOString(),
          },
        },
      };

      if (this.withDateFilter) {
        params.filter.dateFilterStart = {
          condition: {
            path: 'date.value',
            operator: '>',
            value: new Date(
              this.date.getFullYear(),
              this.date.getMonth(),
              this.date.getDate(),
              0,
              0,
              1,
            ),
          },
        };
        params.filter.dateFilterEnd = {
          condition: {
            path: 'date.value',
            operator: '<',
            value: new Date(
              this.date.getFullYear(),
              this.date.getMonth(),
              this.date.getDate(),
              23,
              59,
              59,
            ),
          },
        };
      }

      if (this.excludedVideoId.length > 0) {
        params.filter.excludeSelf = {
          condition: {
            path: 'id',
            operator: '<>',
            value: this.excludedVideoId,
          },
        };
      }

      if (this.favorites) {
        if (this.isFavoritesTypeEmpty('eventinstance', this.eventType)) {
          this.loading = false;
          return;
        }
        params.filter.includeFavorites = {
          condition: {
            path: 'drupal_internal__id',
            operator: 'IN',
            value: this.getFavoritesTypeIds('eventinstance', this.eventType),
          },
        };
      }

      params.page = this.getPageParam;

      params.filter.status = 1;

      params.sort = this.featured
        ? { featured: { path: 'field_ls_featured', direction: 'DESC' } }
        : {};
      params.sort.sortBy = this.sort;

      if (this.categories !== null) {
        if (!this.isCategoriesLoaded) {
          return;
        }
        const termsIds = [];
        this.categories.forEach((tid) => {
          const subcategories = this.$store.getters.getSubcategories(tid);
          termsIds.push(tid, ...this.$store.getters.getNestedTids(subcategories));
        });
        params.filter.in = {
          condition: {
            path: 'eventseries_id.field_ls_category.entity.tid',
            operator: 'IN',
            value: termsIds,
          },
        };
      }

      if (this.instructor) {
        params.filter.orGroup = {
          group: {
            conjunction: 'OR',
          },
        };
        params.filter.instructor = {
          condition: {
            path: 'field_gc_instructor_reference.entity.tid',
            operator: '=',
            value: this.instructor,
            memberOf: 'orGroup',
          },
        };
        params.filter.parentInstructor = {
          condition: {
            path: 'eventseries_id.field_gc_instructor_reference.entity.tid',
            operator: '=',
            value: this.instructor,
            memberOf: 'orGroup',
          },
        };
      }
      this.loadFromJsonApi(params);
    },
    loadFromJsonApi(params) {
      client
        .get(`jsonapi/eventinstance/${this.eventType}`, { params })
        .then((response) => {
          this.links = response.data.links;
          this.listing = this.combineMultiple(
            response.data.data,
            response.data.included,
            this.params,
          );
          if (!this.listingIsNotEmpty) {
            // Emit that listing empty to the parent component.
            this.$emit('listing-not-empty', false);
          }
          this.loading = false;
        })
        .catch((error) => {
          this.error = true;
          this.loading = false;
          console.error(error);
          throw error;
        });
    },
    backOneDay() {
      this.date = new Date(this.date.setTime(this.date.getTime() - this.oneDay));
    },
    forwardOneDay() {
      this.date = new Date(this.date.setTime(this.date.getTime() + this.oneDay));
    },
  },
};
</script>
