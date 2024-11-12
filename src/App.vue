<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
import ChatWindow from './components/ChatWindow.vue'
</script>

<template>
  <main>
    <ChatWindow />
  </main>
</template>

<style scoped>
main {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}
</style>
