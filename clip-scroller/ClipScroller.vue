<template>
  <div
    class="ClipScroller"
    ref="scrollContainer"
    :style="`--top-spinner-container-height: ${topSpinnerContainerHeight}px;`"
  >
    <recycle-scroller
      class="scroller"
      ref="clipScroller"
      :items="events"
      :item-size="itemHeight"
    >
      <template v-slot:before>
        <div
          class="ClipScroller__loading-container ClipScroller__loading-container__before"
        >
          <div
            class="ClipScroller__loading-container__before__load-previous"
            :style="intersectStyles"
            ref="fetchNewerTrigger"
            v-show="showFetchNewerTrigger"
          />

          <div
            class="ClipScroller__loading-container__loading-spinner"
            v-show="showLoadingSpinnerTop"
          >
            <fa-icon
              :icon="['fas', 'spinner']"
              class="LiveWidgetLoading__spinner fa-pulse"
            />
          </div>

          <div
            class="ClipScroller__loading-container__before__dateTimeAnchor"
            v-show="showDateTimeAnchor"
            :style="{ top: anchorTopOffset + 'px' }"
          >
            <div
              class="ClipScroller__loading-container__before__dateTimeAnchor__label"
            >
              {{ dateTimeText }}
            </div>
          </div>
        </div>
      </template>

      <template v-slot="{ item }">
        <event-card
          v-if="item"
          :event="item"
          :selected="item.id === selectedClipId"
          :key="item.id"
          :id="item.id"
          :height="itemHeight"
          @click.native="() => handleEventClicked(item)"
        />
      </template>

      <template v-slot:after>
        <div class="ClipScroller__loading-container">
          <div
            class="ClipScroller__loading-container__load-more"
            :style="intersectStyles"
            ref="fetchOlderTrigger"
            v-show="showFetchOlderTrigger"
          />
          <div class="ClipScroller__loading-container__loading-spinner">
            <fa-icon
              :icon="['fas', 'spinner']"
              class="LiveWidgetLoading__spinner fa-pulse"
              v-show="showLoadingSpinnerBottom"
            />

            <div
              class="ClipScroller__loading-container__loading-spinner__load-end-message"
              :class="`ClipScroller__loading-container__loading-spinner__load-end-message${
                isEmptyState ? '--empty' : ''
              }`"
              v-show="showEndOfListMessage"
            >
              No More Events
              <button
                color="blue"
                v-tooltip="{
                  content: 'Clear Timeframe and Tag Filters',
                  trigger: 'hover focus',
                  delay: 1000,
                }"
                @click.native.prevent="handleClearFiltersClicked"
              >
                Clear Filters
              </button>
            </div>
          </div>
        </div>
      </template>
    </recycle-scroller>
  </div>
</template>

<script lang="ts">
import { useDebounceFn } from "@vueuse/core";
import { DateTime } from "luxon";
import { RecycleScroller } from "vue-virtual-scroller";
import ClipCard from "./ClipCard.vue";
import { createClipSearchStore } from "./store";

export default {
  components: {
    RecycleScroller,
    ClipCard,
  },
  mounted() {
    this.$nextTick(() => {
      // Set recycle-scroller as the intersectionObservers' root
      this.observerOptions.root = this.$refs["clipScroller"]?.$el;

      // Create triggers to fetch more events  when user scrolls to list bookends
      this.fetchNewerObserver = new IntersectionObserver(
        this.handleFetchNewer,
        this.observerOptions
      );
      this.fetchNewerObserver.observe(this.$refs["fetchNewerTrigger"]);

      this.fetchOlderObserver = new IntersectionObserver(
        this.handleFetchOlder,
        this.observerOptions
      );
      this.fetchOlderObserver.observe(this.$refs["fetchOlderTrigger"]);

      // Keep track of user-set dateTimeAnchor with UI list anchor
      this.dateTimeAnchor = this.$refs["dateTimeAnchor"];

      // Set item height for current viewport width
      this.getItemHeight();

      // For keyboard navigation
      window.addEventListener("keydown", this.handleKeyDown);

      // For relative list item sizing
      window.addEventListener("resize", this.getItemHeight);

      /**
       * For showing/hiding AppToolbar on scroll, record initial scrollTop
       * as a reference for calculating scroll
       * direction, add scroll listener
       */
      this.previousScrollTop = this.observerOptions.root.scrollTop;
      this.observerOptions.root.addEventListener(
        "scroll",
        useDebounceFn(this.handleScroll, 25)
      );
    });
  },
  beforeDestroy() {
    window.removeEventListener("keydown", this.handleKeyDown);
    window.removeEventListener("resize", this.getItemHeight);
    document
      .getElementsByClassName("AppToolbar")[0]
      ?.classList?.remove("AppToolbar--hidden");
    document
      .getElementsByClassName("AppContent")[0]
      ?.classList?.remove("AppContent--hidden-app-toolbar");
    document
      .getElementsByClassName("AppNavigation")[0]
      ?.classList?.remove("AppNavigation--hidden-app-toolbar");
    document
      .getElementsByClassName("ClipSearchView")[0]
      ?.classList?.remove("ClipSearchView--hidden-app-toolbar");
    this.observerOptions?.root?.removeEventListener(
      "scroll",
      useDebounceFn(this.handleScroll, 25)
    );
    this.fetchOlderObserver?.disconnect();
    this.fetchNewerObserver?.disconnect();
  },
  data: () => ({
    topSpinnerContainerHeight: 80,
    itemHeight: 90,
    dateTimeText: null,
    observerOptions: {
      root: null,
      rootMargin: "0px",
      threshold: 0,
    },
    anchorTopOffset: 0,
    fetchOlderObserver: null,
    fetchNewerObserver: null,
    dateTimeAnchor: null,
    fetchNewerIntersected: false,
    fetchOlderIntersected: false,
    allowScrollHandling: true,
    previousScrollTop: null,
    loading: false,
    scrollToIndex: 0,
    isEndOfListTop: false,
    isEndOfListBottom: false,
    isNow: true,
    events: [
      {
        id: 0123456789,
        hasVideo: true,
        avatar: null,
        place: "home",
        duration: "0:09",
        startDateTime: DateTime.now(),
        type: "Vehicle Recognized",
        snapshot: null,
        hasPerson: true,
        firstName: "Patrick",
        lastName: "O'Neill"
      }
    ],
    paramState: {
      lastEventDir: "newer",
      startDateTime: DateTime.now()
    },
  }),
  watch: {
    events: {
      immediate: true,
      handler(newVal, oldVal) {
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        /**
         * After loading initial events, after changing startDateTime,
         * the event closest to startDateTime is centered in the
         * scroller along with startDateTime's UI anchor
         */
        const { scrollToIndex } = this;
        const hasScrollToIndex = scrollToIndex && scrollToIndex >= 0;

        // When event list resets, ensure Anchor position is reset
        if (!newVal?.length) {
          this.anchorTopOffset = 0;
        }

        if (!newVal?.length || !hasScrollToIndex) return;

        if (!oldVal?.length) {
          this.$nextTick(() => {
            // Center target event in scroller
            const paddedIndex = scrollToIndex > 3 ? scrollToIndex - 2 : 0;
            this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
            this.$refs?.clipScroller?.scrollToItem(paddedIndex);
          });
          this.anchorTopOffset =
            scrollToIndex * this.itemHeight + this.topSpinnerContainerHeight;

          return;
        }

        /**
         * Calculate how far the newly loaded events would move events-in-viewport, then:
         * - adjust scroll position to keep visible events in the viewport
         * - update scrollToIndex for quick navigation to startDateTime
         * - persist startDateTime's position in the list
         */
        if (this.paramState.lastEventDir !== "newer") return;

        const newEventsLength = newVal.length - oldVal.length;
        const scrollAdjust = newEventsLength * this.itemHeight;

        // Increment scrollToIndex by the number of new events
        this.scrollToIndex += newEventsLength;

        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        this.observerOptions.root.scrollTop =
          this.observerOptions.root.scrollTop + scrollAdjust;
        this.anchorTopOffset = this.anchorTopOffset + scrollAdjust;
      },
    },
    // Format startDateTime for Anchor label
    "paramState.startDateTime": {
      immediate: true,
      handler(startDateTime) {
        // If ISO
        if (startDateTime.includes("T")) {
          const dateTime = DateTime.fromISO(startDateTime);
          this.dateTimeText = dateTime.toLocaleString(DateTime.DATETIME_SHORT);

          // If preformatted
        } else {
          const dateTime = DateTime.fromFormat(
            startDateTime,
            "yyyy-MM-dd hh:mm a"
          );
          this.dateTimeText = dateTime.toLocaleString(DateTime.DATETIME_SHORT);
        }
      },
    },
    /**
     * Adjust Anchor and scroll position when top loadingSpinner is destroyed to prevent
     * anchor mis-positioning and scroller jittering
     */
    isEndOfListTop: {
      immediate: false,
      handler(isEndOfListTop) {
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent

        /**
         * When end-of-list-top, loadSpinner container was destroyed.
         * Adjust scroll and anchor position to compensate
         */
        if (isEndOfListTop) {
          if (!this.observerOptions?.root) return;

          const scroller = this.observerOptions.root;

          /**
           * Adjust positioning by the height the destroyed
           * loader elem, or to top of
           * scroller (scrollTop: 0)
           */
          this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent

          // Scroll position
          scroller.scrollTop =
            scroller.scrollTop >= this.topSpinnerContainerHeight
              ? scroller.scrollTop - this.topSpinnerContainerHeight
              : 0;

          // Anchor position
          this.anchorTopOffset =
            this.anchorTopOffset >= this.topSpinnerContainerHeight
              ? this.anchorTopOffset - this.topSpinnerContainerHeight
              : 0;
        }
      },
    },

    // Adjust scrollTop and anchorTopOffset when ClipCard size changes
    itemHeight: {
      immediate: false,
      handler(newVal, oldVal) {
        if (oldVal !== newVal) {
          if (!this.observerOptions?.root || this.isNow) return;

          const scroller = this.observerOptions.root;

          // Shrink/growth factor is positive when ClipCards grew, negative when they shrank
          const sizeChangeFactor = newVal - oldVal;

          const scrollTopAdjust =
            Math.floor(scroller.scrollTop / oldVal) * sizeChangeFactor; // ClipCards above scrollTop

          const anchorAdjust =
            Math.floor(this.anchorTopOffset / oldVal) * sizeChangeFactor; // ClipCards above DateTime Anchor

          // Adjust positioning by the change in height above the user's scroll position
          this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
          scroller.scrollTop = scroller.scrollTop + scrollTopAdjust;
          this.anchorTopOffset = this.anchorTopOffset + anchorAdjust;
        }
      },
    },
    selectedClipId: {
      immediate: false,
      // Used to scroll selected clip into view during autoPlay
      handler(id) {
        if (!this.events.length || !this.$refs.clipScroller) return;

        const selectedIndex = this.events.findIndex((e) => e.id === id);

        // Selected ClipCard's DOM element
        const selectedCard = document.getElementById(id);
        if (!selectedCard) return;

        // Distance in pixels between top of the selectedCard and top of the scroller
        const offsetDistance =
          selectedCard.getBoundingClientRect().top -
          this.$refs.clipScroller.$el.getBoundingClientRect()?.top;

        /**
         * If selectedCard clips the top of the scroller, scroll it into view unless
         * user scrolled more than 2 cards away--allows user to 'browse' the event
         * list without autoPlay reverting their scroll position
         */
        if (this.itemHeight * -2 < offsetDistance && offsetDistance < 0) {
          this.$refs.clipScroller.scrollToItem(selectedIndex);
        }
      },
    },
  },
  computed: {
    intersectStyles() {
      // Set observer target height to 20 events, or 1 event if less than 47 events are loaded
      const thresholdFactor = this.events?.length > 47 ? 20 : 1;
      const thresholdDistance = this.itemHeight * thresholdFactor;

      return {
        height: `${thresholdDistance}px`,
      };
    },
    showFetchNewerTrigger() {
      return (
        !this.fetchNewerIntersected &&
        !this.loading &&
        !this.isNow &&
        !this.isEndOfListTop
      );
    },
    showFetchOlderTrigger() {
      return (
        !this.fetchOlderIntersected && !this.loading && !this.isEndOfListBottom
      );
    },
    showLoadingSpinnerTop() {
      return this.events.length && !this.isEndOfListTop && !this.isNow;
    },
    showLoadingSpinnerBottom() {
      return (this.events.length && this.loading) || !this.isEndOfListBottom;
    },
    showDateTimeAnchor() {
      return !!this.events.length && !!this.anchorTopOffset && !this.isNow;
    },
    showEndOfListMessage() {
      return this.isEndOfListBottom;
    },
    isEmptyState() {
      return !this.events.length;
    },
    selectedClipId() {
      return "0123456789";
    },
  },
  methods: {
    async handleFetchNewer(entries) {
      try {
        const [entry] = entries;

        if (this.events.length && entry?.isIntersecting) {
          /**
           * Redundant intersection logic patches bug where observer
           * does not detect intersection when user
           * scrolls faster than render cycle
           */
          this.fetchNewerIntersected = true;
          await setTimeout(() => {
            console.log("Fetched new clips");
          }, 50);
        }
      } catch (error) {
        throw error;
      } finally {
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        this.fetchNewerIntersected = false;
      }
    },

    async handleFetchOlder(entries) {
      try {
        const [entry] = entries;

        if (entry?.isIntersecting) {
          /**
           * Redundant intersection logic patches bug where observer
           * does not detect intersection when user
           * scrolls faster than render cycle
           */
          this.fetchOlderIntersected = true;
          await setTimeout(() => {
            console.log("Fetched older clips");
          }, 50);
        }
      } catch (error) {
        throw error;
      } finally {
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        this.fetchOlderIntersected = false;
      }
    },
    /**
     * Hide/Show AppToolbar when scrolling
     */
    handleScroll() {
      if (!this.observerOptions?.root) return;

      const appToolbar = document.getElementsByClassName("AppToolbar")[0];
      const appContent = document.getElementsByClassName("AppContent")[0];
      const appNavigation = document.getElementsByClassName("AppNavigation")[0];
      const clipSearchView =
        document.getElementsByClassName("ClipSearchView")[0];
      const scroller = this.observerOptions.root;

      /**
       * User scrolled down, at least 20px from the top.  20px buffer protects from cases
       * where AppToolbar is hidden and cannot be scrolled back into visibility
       *
       * Prevent operation if event was created by JS (allowScrollHandling is false)
       */
      if (
        this.allowScrollHandling &&
        scroller.scrollTop > this.previousScrollTop &&
        scroller.scrollTop > 20
      ) {
        // Class animates app toolbar above the viewport
        appToolbar.classList.add("AppToolbar--hidden");

        // Classes remove top padding/styling
        appContent.classList.add("AppContent--hidden-app-toolbar");
        appNavigation.classList.add("AppNavigation--hidden-app-toolbar");
        clipSearchView.classList.add("ClipSearchView--hidden-app-toolbar");
      } else if (
        this.allowScrollHandling &&
        scroller.scrollTop < this.previousScrollTop
      ) {
        // User scrolled up, apply inverse styling
        appToolbar.classList.remove("AppToolbar--hidden");
        appContent.classList.remove("AppContent--hidden-app-toolbar");
        appNavigation.classList.remove("AppNavigation--hidden-app-toolbar");
        clipSearchView.classList.remove("ClipSearchView--hidden-app-toolbar");
      }

      // Record scroll position for next method call/calculation, allow scroll handling
      this.previousScrollTop = scroller.scrollTop;
      this.allowScrollHandling = true;
    },

    /**
     * Allows keyboard navigation through events
     */
    handleKeyDown(event: KeyboardEvent) {
      if (!this.observerOptions?.root) return;

      // ClipScroller Element
      const scroller: Element = this.observerOptions.root;

      // ClipScroller's viewport height, used for page scrolling
      let pageScrollHeight: number = scroller.parentElement?.clientHeight
        ? scroller.parentElement?.clientHeight
        : 0;

      switch (event.key) {
        case "ArrowUp": {
          event.preventDefault();
          // Last event will always scroll to the top
          if (scroller.scrollTop < this.itemHeight) {
            scroller.scrollTop = 0;
          }

          // Scroll up 1 ClipCard length
          scroller.scrollTop = scroller.scrollTop - this.itemHeight;
          break;
        }
        case "ArrowDown": {
          event.preventDefault();
          // Last event will always scroll to bottom
          if (scroller.scrollTop > scroller.scrollHeight - this.itemHeight) {
            scroller.scrollTop = scroller.scrollHeight;
          }

          // Scroll down 1 ClipCard length
          scroller.scrollTop = scroller.scrollTop + this.itemHeight;
          break;
        }
        case "PageUp": {
          event.preventDefault();
          // Last event will always scroll to the top
          if (scroller.scrollTop < pageScrollHeight) {
            scroller.scrollTop = 0;
          }

          // Scroll up 1 "page"
          scroller.scrollTop = scroller.scrollTop - pageScrollHeight;
          break;
        }
        case "PageDown": {
          event.preventDefault();
          // Last event will always scroll to bottom
          if (scroller.scrollTop > scroller.scrollHeight - pageScrollHeight) {
            scroller.scrollTop = scroller.scrollHeight;
          }

          // Scroll down 1 "page"
          scroller.scrollTop = scroller.scrollTop + pageScrollHeight;
          break;
        }
        case "Home": {
          event.preventDefault();

          const { scrollToIndex, events } = this;
          const hasScrollToIndex = scrollToIndex && scrollToIndex >= 0;

          // Scroll back to StartDateTime Anchor (scrollToIndex) or 'Now' (index 0)
          this.$refs?.clipScroller?.scrollToItem(
            hasScrollToIndex
              ? events?.length > 20
                ? scrollToIndex - 2
                : scrollToIndex
              : 0
          );
          break;
        }
        default:
          break;
      }
    },
    handleEventClicked(event: Event) {
      console.log("Event selected");
    },
    handleClearFiltersClicked() {
      console.log("Filters cleared");
    },
    getItemHeight() {
      /**
       * Using direct if statements to set itemHeight at each device breakpoint, for performance
       * Thanks https://stackoverflow.com/questions/6665997/switch-statement-for-greater-than-less-than
       */
      // Mobile viewport, mobile landscape
      if (window.innerWidth < 370 || window.innerHeight < 370) {
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        this.itemHeight = 90;
      }
      // Mobile-hd, tablet viewport
      else if (window.innerWidth >= 370 && window.innerWidth < 1100) {
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        this.itemHeight = 100;
      } else {
        // desktop viewport, >= 1100
        this.allowScrollHandling = false; // Prevent handling of JS-created scrollEvent
        this.itemHeight = 125;
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.ClipScroller {
  &__loading-container {
    position: relative;

    &__load-more {
      width: 100%;
      position: absolute;
      bottom: 0;
    }
    &__loading-spinner {
      width: 100%;
      height: var(--top-spinner-container-height);
      display: flex;
      align-items: center;

      #{$fa} {
        color: var(--color-grey-300);
        font-size: $size-5;
        margin: 0 auto;
        display: block;
        z-index: 1;
      }
      &__load-end-message {
        width: 100%;
        font-size: $text-base;
        color: var(--color-grey-400);
        text-align: center;
        .Button {
          margin: $size-1 auto;
          display: block;
        }
        &--empty {
          position: absolute;
          top: 25%;
        }
      }
    }
    &__before {
      position: relative;
      width: 100%;

      &__load-previous {
        width: 100%;
        position: absolute;
        top: 0;
      }
      &__dateTimeAnchor {
        position: absolute;
        color: white;
        top: $size-0;
        width: 100%;
        height: $size-1;
        background: var(--color-red-600);
        z-index: 2;

        &__label {
          display: flex;
          align-items: center;
          justify-content: center;
          width: $size-32;
          height: $size-4;
          font-size: $text-xs;
          border-radius: $size-2;
          position: absolute;
          margin: 0 auto;
          top: -0.375rem;
          left: 0;
          right: 0;
          background: var(--color-red-600);
        }
      }
    }
  }
}
.vue-recycle-scroller {
  height: 100%;
  width: 100%;
  scrollbar-color: var(--color-background-100) var(--color-background-600);
  scrollbar-width: 8px;

  &::-webkit-scrollbar {
    width: 8px !important;
  }

  &::-webkit-scrollbar-track {
    border-radius: 4px !important;
    background: var(--color-background-600) !important;
  }

  &::-webkit-scrollbar-thumb {
    border-radius: 4px !important;
    background: var(--color-background-100) !important;
  }

  &::-webkit-scrollbar-thumb:hover {
    background: var(--color-primary-400) !important;
  }

  &::-webkit-scrollbar-thumb:window-inactive {
    background: var(--color-background-100) !important;
  }

  &__item-view.hover {
    .CardBase {
      border-color: var(--color-primary-300);
    }
  }
}
</style>
