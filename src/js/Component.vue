<template>
  <transition
    :enter-active-class="transition.enter"
    :leave-active-class="transition.leave">
    <div
      ref="root"
      role="alert"
      v-show="isActive"
      class="v-toast__item"
      :class="[`v-toast__item--${type}`, `v-toast__item--${position}`]"
      @mouseover="toggleTimer(true)"
      @mouseleave="toggleTimer(false)"
      @click="whenClicked">
      <div class="v-toast__icon"></div>
      <p class="v-toast__text" v-html="message"></p>
      <button
        class="v-toast__close"
        v-if="dismissible"
        @click="dismiss"
      >
        <CloseX />
      </button>
    </div>
  </transition>
</template>

<script>
import {defineComponent, render} from 'vue';
import {removeElement} from './helpers.js';
import Timer from "./timer.js";
import Positions from './positions.js'
import eventBus from './bus.js'
import CloseX from './CloseX.vue'

export default defineComponent({
  name: 'toast',
  components: { CloseX },
  props: {
    message: {
      type: String,
      required: true
    },
    type: {
      type: String,
      default: 'success'
    },
    position: {
      type: String,
      default: Positions.BOTTOM_RIGHT,
      validator(value) {
        return Object.values(Positions).includes(value)
      }
    },
    duration: {
      type: Number,
      default: 3000
    },
    dismissible: {
      type: Boolean,
      default: true
    },
    onDismiss: {
      type: Function,
      default: () => {
      }
    },
    onClick: {
      type: Function,
      default: () => {
      }
    },
    queue: Boolean,
    pauseOnHover: {
      type: Boolean,
      default: true
    },
  },
  data() {
    return {
      isActive: false,
      parentTop: null,
      parentBottom: null,
      isHovered: false,
    }
  },
  beforeMount() {
    this.setupContainer()
  },
  mounted() {
    this.showNotice();
    eventBus.on('toast-clear', this.dismiss)
  },
  methods: {
    setupContainer() {
      this.parentTop = document.querySelector('.v-toast.v-toast--top');
      this.parentBottom = document.querySelector('.v-toast.v-toast--bottom');
      // No need to create them, they already exists
      if (this.parentTop && this.parentBottom) return;

      if (!this.parentTop) {
        this.parentTop = document.createElement('div');
        this.parentTop.className = 'v-toast v-toast--top';
      }

      if (!this.parentBottom) {
        this.parentBottom = document.createElement('div');
        this.parentBottom.className = 'v-toast v-toast--bottom'
      }

      const container = document.body;
      container.appendChild(this.parentTop);
      container.appendChild(this.parentBottom);
    },

    shouldQueue() {
      if (!this.queue) return false;

      return (
        this.parentTop.childElementCount > 0 ||
        this.parentBottom.childElementCount > 0
      )
    },

    dismiss() {
      if (this.timer) this.timer.stop();
      clearTimeout(this.queueTimer);
      this.isActive = false;

      // Timeout for the animation complete before destroying
      setTimeout(() => {
        this.onDismiss.apply(null, arguments);

        const wrapper = this.$refs.root;
        // unmount the component
        render(null, wrapper);
        removeElement(wrapper)
      }, 150)
    },

    showNotice() {
      if (this.shouldQueue()) {
        // Call recursively if should queue
        this.queueTimer = setTimeout(this.showNotice, 250);
        return
      }
      const wrapper = this.$refs.root.parentElement
      this.correctParent.insertAdjacentElement('afterbegin', this.$refs.root);
      removeElement(wrapper);

      this.isActive = true;

      if (this.duration) {
        this.timer = new Timer(this.dismiss, this.duration);
      }
    },

    whenClicked() {
      this.onClick.apply(null, arguments);
    },

    toggleTimer(newVal) {
      if (!this.pauseOnHover || !this.timer) return;
      newVal ? this.timer.pause() : this.timer.resume();
    }
  },
  computed: {
    correctParent() {
      switch (this.position) {
        case Positions.TOP:
        case Positions.TOP_RIGHT:
        case Positions.TOP_LEFT:
          return this.parentTop;

        case Positions.BOTTOM:
        case Positions.BOTTOM_RIGHT:
        case Positions.BOTTOM_LEFT:
          return this.parentBottom;
      }
    },
    transition() {
      switch (this.position) {
        case Positions.TOP:
        case Positions.TOP_RIGHT:
        case Positions.TOP_LEFT:
          return {
            enter: 'v-toast--fade-in-down',
            leave: 'v-toast--fade-out'
          };

        case Positions.BOTTOM:
        case Positions.BOTTOM_RIGHT:
        case Positions.BOTTOM_LEFT:
          return {
            enter: 'v-toast--fade-in-up',
            leave: 'v-toast--fade-out'
          }
      }
    },
  },
  beforeUnmount() {
    eventBus.off('toast-clear', this.dismiss)
  }
})
</script>
