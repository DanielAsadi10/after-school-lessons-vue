<script setup>
import { onMounted, ref } from 'vue'

const lessons = ref([])
const isLoading = ref(true)
const loadError = ref('')
const apiBaseUrl = import.meta.env.VITE_API_URL || '/api'

async function loadLessons() {
  isLoading.value = true
  loadError.value = ''

  try {
    const response = await fetch(`${apiBaseUrl}/lessons`)

    if (!response.ok) {
      throw new Error(`The lessons request failed with status ${response.status}.`)
    }

    lessons.value = await response.json()
  } catch (error) {
    console.error('Could not load lessons:', error)
    loadError.value = 'We could not load the classes. Check that the API is running, then try again.'
  } finally {
    isLoading.value = false
  }
}

onMounted(loadLessons)
</script>

<template>
  <main class="page">
    <p class="eyebrow">AFTER SCHOOL PROGRAMME</p>
    <h1>Coding classes for curious minds</h1>
    <p class="intro">Explore practical classes and find a new skill to learn.</p>

    <p v-if="isLoading" class="status-message" role="status">Loading classes…</p>

    <div v-else-if="loadError" class="error-message" role="alert">
      <p>{{ loadError }}</p>
      <button type="button" @click="loadLessons">Try again</button>
    </div>

    <p v-else-if="lessons.length === 0" class="status-message">
      No classes are available right now.
    </p>

    <section v-else class="lesson-grid" aria-label="Available coding classes">
      <article v-for="lesson in lessons" :key="lesson.id" class="lesson-card">
        <div class="lesson-icon" aria-hidden="true">{{ lesson.icon || '📚' }}</div>
        <p class="lesson-location">{{ lesson.location }}</p>
        <h2>{{ lesson.subject }}</h2>
        <div class="lesson-details">
          <span>&pound;{{ lesson.price }} per term</span>
          <span>{{ lesson.spaces }} spaces</span>
        </div>
        <button type="button" disabled>Add to cart</button>
      </article>
    </section>
  </main>
</template>
