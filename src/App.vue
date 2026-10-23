<script setup>
const lessons = [
  { id: 1, subject: 'Python for Beginners', location: 'Hendon', price: 45, spaces: 5, icon: '🐍' },
  { id: 2, subject: 'Build Your First Website', location: 'Colindale', price: 50, spaces: 5, icon: '🌐' },
  { id: 3, subject: 'Scratch Game Design', location: 'Brent Cross', price: 35, spaces: 5, icon: '🎮' },
  { id: 4, subject: 'Robotics Club', location: 'Golders Green', price: 60, spaces: 5, icon: '🤖' },
  { id: 5, subject: 'Creative Coding', location: 'Hendon', price: 40, spaces: 5, icon: '🎨' },
  { id: 6, subject: 'JavaScript Explorers', location: 'Colindale', price: 55, spaces: 5, icon: '✨' },
  { id: 7, subject: 'Make a Mobile App', location: 'Brent Cross', price: 65, spaces: 5, icon: '📱' },
  { id: 8, subject: 'Minecraft Modding', location: 'Golders Green', price: 50, spaces: 5, icon: '🧱' },
  { id: 9, subject: 'Web Design Studio', location: 'Hendon', price: 45, spaces: 5, icon: '🖌️' },
  { id: 10, subject: 'AI and Machine Learning Basics', location: 'Colindale', price: 70, spaces: 5, icon: '🧠' },
]
</script>

<template>
  <main class="page">
    <p class="eyebrow">AFTER SCHOOL PROGRAMME</p>
    <h1>Coding classes for curious minds</h1>
    <p class="intro">Explore practical classes and find a new skill to learn.</p>
    <section class="lesson-grid" aria-label="Available coding classes">
      <article v-for="lesson in lessons" :key="lesson.id" class="lesson-card">
        <div class="lesson-icon" aria-hidden="true">{{ lesson.icon }}</div>
        <p class="lesson-location">{{ lesson.location }}</p>
        <h2>{{ lesson.subject }}</h2>
        <div class="lesson-details">
          <span>£{{ lesson.price }} per term</span>
          <span>{{ lesson.spaces }} spaces</span>
        </div>
        <button type="button" disabled>Add to cart</button>
      </article>
    </section>
  </main>
</template>
