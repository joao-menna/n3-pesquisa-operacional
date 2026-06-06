<script setup lang="ts">
defineProps({
    title: String,
    class: {
        type: String,
        default: '',
    },
})
</script>

<template>
  <div class="p-4 size-full rounded-lg shadow-md bg-white border border-zinc-300 overflow-clip " :class="class">
    <h2 v-if="title" class="text-xl font-semibold mb-2">{{ title }}</h2>
    <div class="flex flex-col gap-4">
      <slot></slot>
      <slot name="image"></slot>
    </div>
  </div>
</template>
