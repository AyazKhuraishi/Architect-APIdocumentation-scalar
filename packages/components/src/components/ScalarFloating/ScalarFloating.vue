<script setup lang="ts">
import { getSideAxis } from '@floating-ui/utils'
import {
  type MiddlewareData,
  autoUpdate,
  flip,
  offset,
  shift,
  size,
  useFloating,
} from '@floating-ui/vue'
import { type Ref, computed, ref } from 'vue'

import type { FloatingOptions } from './types'
import { useResizeWithTarget } from './useResizeWithTarget'

const props = defineProps<FloatingOptions>()

defineSlots<{
  /** The reference element for the element in the #floating slot */
  default(): any
  /** The floating element */
  floating(props: {
    /** The width of the reference element if `resize` is true and placement is on the y axis */
    width?: string
    /** The height of the reference element if `resize` is true and placement is on the x axis */
    height?: string
    /** The middleware data return by Floating UI */
    data?: MiddlewareData
  }): any
}>()

defineOptions({ inheritAttrs: false })

const floatingRef: Ref<HTMLElement | null> = ref(null)
const wrapperRef: Ref<HTMLElement | null> = ref(null)

/** Fallback to div wrapper if a button element is not provided */
const targetRef = computed(
  () =>
    (props.targetRef || wrapperRef.value?.children?.[0] || wrapperRef.value) ??
    undefined,
)

const targetSize = useResizeWithTarget(targetRef, {
  enabled: computed(() => props.resize),
})

const targetWidth = computed(() =>
  getSideAxis(props.placement || 'bottom') === 'y'
    ? targetSize.width.value
    : undefined,
)

const targetHeight = computed(() =>
  getSideAxis(props.placement || 'bottom') === 'x'
    ? targetSize.height.value
    : undefined,
)

const { floatingStyles, middlewareData } = useFloating(targetRef, floatingRef, {
  placement: computed(() => props.placement),
  whileElementsMounted: autoUpdate,
  middleware: computed(() => [
    offset(5),
    flip(),
    shift(),
    size({
      apply({ availableWidth, availableHeight, elements }) {
        // Assign the max width and height to the floating element
        // @see https://floating-ui.com/docs/size
        Object.assign(elements.floating.style, {
          maxWidth: `${Math.max(0, availableWidth) - 25}px`,
          maxHeight: `${Math.max(0, availableHeight) - 25}px`,
        })
      },
    }),
    ...(props.middleware ?? []),
  ]),
})
</script>
<template>
  <div
    ref="wrapperRef"
    :class="{ contents: !!$slots.default }">
    <slot />
  </div>
  <Teleport
    v-if="isOpen"
    :disabled="!teleport"
    :to="typeof teleport === 'string' ? teleport : 'body'">
    <div class="scalar-app">
      <div
        ref="floatingRef"
        class="relative z-overlay"
        :style="floatingStyles">
        <slot
          :data="middlewareData"
          :height="targetHeight"
          name="floating"
          :width="targetWidth" />
      </div>
    </div>
  </Teleport>
</template>
