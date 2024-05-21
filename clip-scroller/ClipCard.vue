<template>
  <div
    class="EventCard"
    :style="`--card-height : ${height}px`"
    :class="{ 'EventCard--selected': selected }"
  >
    <div class="EventCardThumbnail">
      <img
        class="EventCardThumbnail__snapshot"
        v-if="!!event.snapshot"
        :src="event.snapshot"
      />

      <div
        v-if="showPersonTag"
        class="PersonTag"
        :class="{ 'PersonTag--unknown': isUnknownPerson }"
      >
        <div
          v-if="showPersonTag"
          class="PersonTag__avatar"
          :class="{ 'PersonTag__avatar--empty': !event.avatar }"
        >
          <img
            v-if="!!event.avatar"
            :src="event.avatar"
            class="PersonTag__avatar__image"
          />

          <fa-icon
            v-if="!event.avatar"
            class="PersonTag__avatar__icon"
            :icon="['fas', 'user']"
          />
        </div>

        <div v-if="showPersonTag" class="PersonTag__label">
          {{ personName }}
        </div>
      </div>

      <fa-icon
        class="EventCardThumbnail__no-video-icon"
        v-show="!event.hasVideo"
        :icon="['fas', 'video-slash']"
      />
    </div>

    <div class="EventCardLabels">
      <div class="EventCardLabels__event-date">
        {{ eventDateTime.date }}
      </div>

      <div class="EventCardLabels__event-time">
        {{ eventDateTime.time }}
      </div>

      <div class="EventCardLabels__data">
        <place-icon class="EventCardLabels__data__place-icon" />

        <div class="EventCardLabels__data__place">
          {{ event.place }}
        </div>

        <component
          v-if="eventSource.icon"
          :is="eventSource.icon"
          class="EventCardLabels__data__event-source-icon"
        />
        <fa-icon
          v-else
          class="EventCardLabels__data__event-source-icon"
          :class="[
            {
              'EventCardLabels__data__event-source-icon--locked-down':
                isLockedDown,
            },
            {
              'EventCardLabels__data__event-source-icon--lockdown-lifted':
                isLockdownLifted,
            },
          ]"
          :icon="['fas', eventSource.customIcon]"
        />

        <div
          class="EventCardLabels__data__event-source-name"
          :class="[
            {
              'EventCardLabels__data__event-source-name--locked-down':
                isLockedDown,
            },
            {
              'EventCardLabels__data__event-source-name--lockdown-lifted':
                isLockdownLifted,
            },
          ]"
        >
          {{ eventSource.name }}
        </div>

        <fa-icon
          class="EventCardLabels__data__event-type-icon"
          :icon="['fas', 'circle-info']"
        />

        <div class="EventCardLabels__data__event-type">
          {{ event.type }}
        </div>
      </div>

      <div
        v-show="event.hasVideo && event.duration"
        class="EventCardLabels__duration"
      >
        <camera-icon
          class="EventCardLabels__duration__has-video-icon"
          v-show="event.hasVideo"
        />

        {{ duration }}
      </div>
    </div>
  </div>
</template>

<script>
import { DateTime } from "luxon";

export default {
  props: {
    selected: {
      type: Boolean,
      required: false,
      default: false,
    },
    height: {
      type: Number,
      required: true,
    },
  },
  computed: {
    eventSource() {
      return {
        name: "patrick-camera",
        icon: "camera-icon",
      };
    },
    isLockedDown() {
      return false;
    },
    isLockdownLifted() {
      return false;
    },
    eventDateTime() {
      // Return event date, time strings separately
      const localeDateTimeString = event.startDateTime.toLocaleString(
        DateTime.DATETIME_SHORT_WITH_SECONDS
      );

      return {
        date: localeDateTimeString.split(", ")[0],
        time: localeDateTimeString.split(", ")[1],
      };
    },
    isUnknownPerson() {
      const { firstName, lastName } = this.event;
      return !firstName && !lastName;
    },
    personName() {
      const { firstName, lastName } = this.event;
      return `${firstName} ${lastName}`;
    },
    showPersonTag() {
      return this.event.hasPerson;
    },
    duration() {
      return event.duration;
    },
  },
};
</script>

<style lang="scss" scoped>
.EventCard {
  height: var(--card-height);
  width: 100%;
  display: flex;
  flex-direction: row;
  color: var(--color-white);
  padding: $size-px;
  border-radius: $size-1;
  background-image: linear-gradient(
    0deg,
    var(--color-background-400) 0%,
    var(--color-background-500) 100%
  );
  -webkit-font-smoothing: subpixel-antialiased;
  letter-spacing: 0.03rem;
  cursor: pointer;

  &:hover {
    background-image: linear-gradient(
      0deg,
      var(--color-primary-300) 0%,
      var(--color-primary-500) 100%
    );
  }

  &--selected {
    background-image: linear-gradient(
      0deg,
      var(--color-primary-300) 0%,
      var(--color-primary-500) 100%
    );
  }
}

.EventCardThumbnail {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-grow: 0;
  position: relative;
  width: 156.44px;
  min-width: 156.44px;
  height: 100%;
  border-top-left-radius: $size-1;
  border-bottom-left-radius: $size-1;
  background-color: var(--color-background-700);

  // Hardcoded to protect source aspect ratio
  @include breakpoint(min-width, 370px) {
    width: 174px;
    min-width: 174px;
  }

  // Mobile-landscape-xs, using height query over orientation/aspect-ratio for mobiles with tall viewports
  @media screen and (max-height: 370px) {
    // Hardcoded to protect source aspect ratio
    width: 156.44px;
    min-width: 156.44px;
  }

  // Hardcoded to protect source aspect ratio
  @include breakpoint(desktop) {
    width: 218.66px;
    min-width: 218.66px;
  }

  &__snapshot {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  &__no-video-icon {
    position: absolute;
    height: $size-8;
    width: $size-10;
    padding: 0.313rem $size-2;
    background-color: var(--color-background-700);
    border-radius: $size-4;
    font-size: $size-5;
  }
}

.PersonTag {
  position: absolute;
  top: $size-1;
  left: $size-1;
  display: flex;
  align-items: center;
  max-width: calc(100% - 0.5rem);
  border-radius: $size-6;
  height: $size-6;
  background: hsla(240, 8%, 5%, 0.7);

  $personTag: &;

  @include breakpoint(desktop) {
    border-radius: $size-8;
    height: $size-8;
  }

  &__avatar {
    display: flex;
    align-items: center;
    justify-content: center;
    width: $size-6;
    height: $size-6;
    border-radius: 50%;
    overflow: hidden;

    &__image {
      width: 100%;
    }

    &__icon {
      width: 0.8rem;
      font-size: 0.8rem;
      color: var(--text-light);
      margin: auto;

      @include breakpoint(desktop) {
        width: $size-4;
        font-size: $size-4;
      }
    }

    &--empty {
      margin-left: $size-2;
      width: $size-3;
    }

    @include breakpoint(desktop) {
      width: $size-8;
      height: $size-8;

      &--empty {
        margin-left: $size-2;
        width: $size-4;
      }
    }
  }

  &__label {
    height: $size-6;
    line-height: $size-6;
    padding: 0 $size-2;
    font-size: 0.625rem;
    font-weight: 600;
    @extend %line-clamp;
    word-break: break-word;

    @include breakpoint(desktop) {
      height: $size-8;
      line-height: $size-8;
      font-size: 0.825rem;
    }
  }

  &--unknown {
    color: var(--color-warning-400);

    #{$personTag}__avatar__icon {
      color: var(--color-warning-400);
    }
  }
}

.EventCardLabels {
  height: 100%;
  flex-grow: 1;
  border-top-right-radius: $size-1;
  border-bottom-right-radius: $size-1;
  font-size: 0.625rem;
  background-color: var(--color-background-600);
  color: var(--color-grey-400);
  padding: $size-2;
  overflow: visible;

  @include breakpoint(min-width, 370px) {
    font-size: 0.7rem;
  }

  // Mobile-landscape-xs, using height query over orientation/aspect-ratio for mobiles with tall viewports
  @media screen and (max-height: 370px) {
    font-size: 0.625rem;
  }

  @include breakpoint(desktop) {
    font-size: 0.85rem;
  }

  &__event-date {
    font-weight: 600;
    line-height: 0.625rem;
    color: var(--color-grey-300);

    @include breakpoint(min-width, 370px) {
      line-height: $size-3;
    }

    // Mobile-landscape-xs, using height query over orientation/aspect-ratio for mobiles with tall viewports
    @media screen and (max-height: 370px) {
      line-height: 0.625rem;
    }

    @include breakpoint(desktop) {
      line-height: 0.85rem;
    }
  }

  &__event-time {
    font-size: 0.825rem;
    color: var(--text-light);
    font-weight: 700;

    @include breakpoint(min-width, 370px) {
      font-size: $size-4;
    }

    // Mobile-landscape-xs, using height query over orientation/aspect-ratio for mobiles with tall viewports
    @media screen and (max-height: 370px) {
      font-size: 0.825rem;
    }

    @include breakpoint(desktop) {
      font-size: $size-5;
    }
  }

  &__data {
    display: grid;
    grid-template-columns: $size-3 auto;
    grid-template-rows: repeat(4, auto);
    column-gap: $size-2;
    margin-left: $size-1;

    &__place {
      grid-column: 2;
      grid-row: 1;
      display: flex;
      align-items: center;
      @extend %line-clamp;
    }

    &__place-icon {
      grid-column: 1;
      grid-row: 1;
      color: var(--text-light);
      margin: auto;
    }

    &__event-source-name {
      grid-column: 2;
      grid-row: 2;
      display: flex;
      align-items: center;
      @extend %line-clamp;

      &--locked-down {
        color: var(--color-danger-400);
      }

      &--lockdown-lifted {
        color: var(--color-primary-400);
      }
    }

    &__event-source-icon {
      grid-column: 1;
      grid-row: 2;
      width: 0.625rem;
      font-size: 0.625rem;
      color: var(--text-light);
      margin: auto;

      @include breakpoint(min-width, 370px) {
        width: 0.7rem;
        font-size: 0.7rem;
      }

      // Mobile-landscape-xs, using height query over orientation/aspect-ratio for mobiles with tall viewports
      @media screen and (max-height: 370px) {
        width: 0.625rem;
        font-size: 0.625rem;
      }

      @include breakpoint(desktop) {
        width: 0.85rem;
        font-size: 0.85rem;
      }

      &--locked-down {
        color: var(--color-danger-400);
      }

      &--lockdown-lifted {
        color: var(--color-primary-400);
      }
    }

    &__event-type {
      grid-column: 2;
      grid-row: 3;
      display: flex;
      align-items: center;
      @extend %line-clamp;
    }
    &__event-type-icon {
      grid-column: 1;
      grid-row: 3;
      width: 0.625rem;
      font-size: 0.625rem;
      color: var(--text-light);
      margin: auto;

      @include breakpoint(min-width, 370px) {
        width: 0.7rem;
        font-size: 0.7rem;
      }

      // Mobile-landscape-xs, using height query over orientation/aspect-ratio for mobiles with tall viewports
      @media screen and (max-height: 370px) {
        width: 0.625rem;
        font-size: 0.625rem;
      }

      @include breakpoint(desktop) {
        width: 0.85rem;
        font-size: 0.85rem;
      }
    }
  }

  &__duration {
    position: absolute;
    left: $size-1;
    bottom: $size-1;
    color: var(--color-white);
    font-weight: 600;
    background: hsla(240, 8%, 5%, 0.7);
    padding: $size-0 $size-2;
    border-radius: $size-2;

    &__has-video-icon {
      color: var(--baseColors-green);
    }
  }
}
</style>
