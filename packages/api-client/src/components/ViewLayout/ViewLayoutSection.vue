<script setup lang="ts">
import { nanoid } from 'nanoid'

const id = nanoid()
</script>
<template>
  <section
    :aria-labelledby="id"
    class="flex h-full xl:min-w-0 xl:flex-1 flex-col xl:custom-scroll bg-b-1">
    <div
      :id="id"
      class="min-h-11 xl:min-h-header flex items-center border-b-1/2 px-3.5 py-1.5 md:px-4 md:py-2.5 xl:px-6 xl:pr-5 text-sm font-medium sticky top-0 bg-b-1 xl:rounded-none">
      <slot name="title" />
    </div>
    <slot />
  </section>
</template>
