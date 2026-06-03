<script setup lang="ts">
defineProps<{
    title?: string;
}>();
</script>

<template>
    <div class="rounded-lg border border-zinc-300 bg-zinc-50 p-3">
        <h3 class="font-bold mb-1">{{ title }}</h3>
        <slot></slot>
    </div>
</template>