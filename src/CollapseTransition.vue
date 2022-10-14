<template>
  <transition
    :name="name"
    @before-appear="beforeAppear"
    @appear="appear"
    @after-appear="afterAppear"
    @appear-cancelled="appearCancelled"
    @before-enter="beforeEnter"
    @enter="enter"
    @after-enter="afterEnter"
    @enter-cancelled="enterCancelled"
    @before-leave="beforeLeave"
    @leave="leave"
    @after-leave="afterLeave"
    @leave-cancelled="leaveCancelled"
  >
    <slot></slot>
  </transition>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

interface CachedStyles {
  height?: string;
  paddingTop?: string;
  paddingBottom?: string;
  width?: string;
  paddingLeft?: string;
  paddingRight?: string;
}

interface DataProps {
  cachedStyles: CachedStyles;
}

export default defineComponent({
  name: 'CollapseTransition',

  props: {
    name: {
      type: String,
      required: false,
      default: 'collapse',
    },
    dimension: {
      type: String,
      required: false,
      default: 'height',
      validator: (value: string) => ['height', 'width'].includes(value),
    },
    duration: {
      type: Number,
      required: false,
      default: 300,
    },
    easing: {
      type: String,
      required: false,
      default: 'ease-in',
    },
  },

  watch: {
    dimension() {
      this.clearCachedDimensions();
    },
  },

  data(): DataProps {
    return {
      cachedStyles: {},
    };
  },

  computed: {
    transition() {
      let transitions: string[] = [];
      Object.keys(this.cachedStyles).forEach((key) => {
        transitions.push(
          `${this.convertToCssProperty(key)} ${this.duration}ms ${this.easing}`
        );
      });

      return transitions.join(', ');
    },
  },

  methods: {
    beforeAppear(el: HTMLElement) {
      this.$emit('before-appear', el);
    },

    appear(el: HTMLElement) {
      this.$emit('appear', el);
    },

    afterAppear(el: HTMLElement) {
      this.$emit('after-appear', el);
    },

    appearCancelled(el: HTMLElement) {
      this.$emit('appear-cancelled', el);
    },

    beforeEnter(el: HTMLElement) {
      this.$emit('before-enter', el);
    },

    enter(el: HTMLElement, done: (...args: any) => void) {
      // Because width and height may be 'auto',
      // first detect and cache the dimensions
      this.detectAndCacheDimensions(el);

      // The order of applying styles is important:
      // - 1. Set styles for state before transition
      // - 2. Force repaint
      // - 3. Add transition style
      // - 4. Set styles for state after transition
      // If the order is not right and you open any 2nd level submenu
      // for the first time, the transition will not work.
      this.setClosedDimensions(el);
      this.hideOverflow(el);
      this.forceRepaint(el);
      this.setTransition(el);
      this.setOpenedDimensions(el);

      this.$emit('enter', el, done);

      // Call done() when the transition ends
      // to trigger the @after-enter event.
      setTimeout(done, this.duration);
    },

    afterEnter(el: HTMLElement) {
      // Clean up inline styles
      this.unsetOverflow(el);
      this.unsetTransition(el);
      this.unsetDimensions(el);
      this.clearCachedDimensions();

      this.$emit('after-enter', el);
    },

    enterCancelled(el: HTMLElement) {
      this.$emit('enter-cancelled', el);
    },

    beforeLeave(el: HTMLElement) {
      this.$emit('before-leave', el);
    },

    leave(el: HTMLElement, done: (...args: any) => void) {
      // For some reason, @leave triggered when starting
      // from open state on page load. So for safety,
      // check if the dimensions have been cached.
      this.detectAndCacheDimensions(el);

      // The order of applying styles is less important
      // than in the enter phase, as long as we repaint
      // before setting the closed dimensions.
      // But it is probably best to use the same
      // order as the enter phase.
      this.setOpenedDimensions(el);
      this.hideOverflow(el);
      this.forceRepaint(el);
      this.setTransition(el);
      this.setClosedDimensions(el);

      this.$emit('leave', el, done);

      // Call done() when the transition ends
      // to trigger the @after-leave event.
      // This will also cause v-show
      // to reapply 'display: none'.
      setTimeout(done, this.duration);
    },

    afterLeave(el: HTMLElement) {
      // Clean up inline styles
      this.unsetOverflow(el);
      this.unsetTransition(el);
      this.unsetDimensions(el);
      this.clearCachedDimensions();

      this.$emit('after-leave', el);
    },

    leaveCancelled(el: HTMLElement) {
      this.$emit('leave-cancelled', el);
    },

    detectAndCacheDimensions(el: HTMLElement) {
      // Cache actual dimensions
      // only once to void invalid values when
      // triggering during a transition
      if (Object.keys(this.cachedStyles).length) return;

      const visibility = el.style.visibility;
      const display = el.style.display;

      // Trick to get the width and
      // height of a hidden element
      el.style.visibility = 'hidden';
      el.style.display = '';

      this.cachedStyles = this.detectRelevantDimensions(el);

      // Restore any original styling
      el.style.visibility = visibility;
      el.style.display = display;
    },

    clearCachedDimensions() {
      this.cachedStyles = {};
    },

    detectRelevantDimensions(el: HTMLElement): CachedStyles {
      // These properties will be transitioned
      if (this.dimension === 'height') {
        return {
          height: `${el.offsetHeight}px`,
          paddingTop:
            el.style.paddingTop || this.getCssValue(el, 'padding-top'),
          paddingBottom:
            el.style.paddingBottom || this.getCssValue(el, 'padding-bottom'),
        };
      }

      if (this.dimension === 'width') {
        return {
          width: `${el.offsetWidth}px`,
          paddingLeft:
            el.style.paddingLeft || this.getCssValue(el, 'padding-left'),
          paddingRight:
            el.style.paddingRight || this.getCssValue(el, 'padding-right'),
        };
      }

      return {};
    },

    setTransition(el: HTMLElement) {
      el.style.transition = this.transition;
    },

    unsetTransition(el: HTMLElement) {
      el.style.transition = '';
    },

    hideOverflow(el: HTMLElement) {
      el.style.overflow = 'hidden';
    },

    unsetOverflow(el: HTMLElement) {
      el.style.overflow = '';
    },

    setClosedDimensions(el: HTMLElement) {
      Object.keys(this.cachedStyles).forEach((key: string) => {
        el.style[key as keyof CachedStyles] = '0';
      });
    },

    setOpenedDimensions(el: HTMLElement) {
      Object.keys(this.cachedStyles).forEach((key: string) => {
        el.style[key as keyof CachedStyles] = this.cachedStyles[
          key as keyof CachedStyles
        ] as string;
      });
    },

    unsetDimensions(el: HTMLElement) {
      Object.keys(this.cachedStyles).forEach((key: string) => {
        el.style[key as keyof CachedStyles] = '';
      });
    },

    forceRepaint(el: HTMLElement) {
      // Force repaint to make sure the animation is triggered correctly.
      // Thanks: https://markus.oberlehner.net/blog/transition-to-height-auto-with-vue/
      getComputedStyle(el)[this.dimension as keyof CachedStyles];
    },

    getCssValue(el: HTMLElement, style: string) {
      return getComputedStyle(el, null).getPropertyValue(style);
    },

    convertToCssProperty(style: string) {
      // Example: convert 'paddingTop' to 'padding-top'
      // Thanks: https://gist.github.com/tan-yuki/3450323
      const upperChars = style.match(/([A-Z])/g);

      if (!upperChars) {
        return style;
      }

      for (let i = 0, n = upperChars.length; i < n; i++) {
        style = style.replace(
          new RegExp(upperChars[i]),
          '-' + upperChars[i].toLowerCase()
        );
      }

      if (style.slice(0, 1) === '-') {
        style = style.slice(1);
      }

      return style;
    },
  },
});
</script>
