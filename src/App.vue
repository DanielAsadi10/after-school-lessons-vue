<script setup>
import { computed, onMounted, ref } from 'vue'

const lessons = ref([])
const isLoading = ref(true)
const loadError = ref('')
const apiBaseUrl = import.meta.env.VITE_API_URL || '/api'
const sortBy = ref('subject')
const sortDirection = ref('asc')
const cartItems = ref([])
const isCartOpen = ref(false)
const customerName = ref('')
const customerPhone = ref('')
const checkoutError = ref('')
const orderMessage = ref('')
const isSubmitting = ref(false)

const sortedLessons = computed(() => {
  return [...lessons.value].sort((left, right) => {
    const leftValue = left[sortBy.value]
    const rightValue = right[sortBy.value]
    const comparison = typeof leftValue === 'number' && typeof rightValue === 'number'
      ? leftValue - rightValue
      : String(leftValue).localeCompare(String(rightValue), undefined, { sensitivity: 'base' })

    return sortDirection.value === 'asc' ? comparison : -comparison
  })
})

const cartTotal = computed(() => {
  return cartItems.value.reduce((total, lesson) => total + Number(lesson.price || 0), 0)
})

function addToCart(lesson) {
  if (lesson.spaces <= 0) return

  lesson.spaces -= 1
  cartItems.value.push({ ...lesson })
}

function removeFromCart(index) {
  const [removedLesson] = cartItems.value.splice(index, 1)
  const lesson = lessons.value.find((item) => item.id === removedLesson.id)

  if (lesson) lesson.spaces += 1
}

async function checkout() {
  checkoutError.value = ''
  orderMessage.value = ''
  isSubmitting.value = true

  const quantities = new Map()
  for (const item of cartItems.value) quantities.set(item.id, (quantities.get(item.id) || 0) + 1)

  try {
    for (const [id, quantity] of quantities) {
      const lesson = lessons.value.find((item) => item.id === id)
      const response = await fetch(`${apiBaseUrl}/lessons/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ spaces: lesson.spaces }),
      })
      if (!response.ok) throw new Error(`Could not update spaces for ${lesson.subject}.`)
    }

    const orderResponse = await fetch(`${apiBaseUrl}/orders`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: customerName.value.trim(),
        phone: customerPhone.value.trim(),
        lessons: [...quantities].map(([id, quantity]) => {
          const lesson = lessons.value.find((item) => item.id === id)
          return { id, subject: lesson.subject, quantity, price: lesson.price }
        }),
        total: cartTotal.value,
      }),
    })
    const orderResult = await orderResponse.json()
    if (!orderResponse.ok) throw new Error(orderResult.error || 'Could not submit your order.')

    orderMessage.value = 'Thanks, ' + customerName.value.trim() + '! Your order has been saved.'
    cartItems.value = []
    customerName.value = ''
    customerPhone.value = ''
    await loadLessons()
  } catch (error) {
    checkoutError.value = error.message || 'We could not submit your order. Please try again.'
  } finally {
    isSubmitting.value = false
  }
}

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

    <button
      class="cart-toggle"
      type="button"
      :disabled="cartItems.length === 0 && !isCartOpen"
      @click="isCartOpen = !isCartOpen"
    >
      {{ isCartOpen ? 'Back to classes' : 'Shopping cart (' + cartItems.length + ')' }}
    </button>

    <section v-if="isCartOpen" class="cart-panel" aria-label="Shopping cart">
      <h2>Your shopping cart</h2>
      <p v-if="orderMessage" class="success-message" role="status">{{ orderMessage }}</p>
      <p v-if="checkoutError" class="error-message" role="alert">{{ checkoutError }}</p>
      <p v-if="cartItems.length === 0" class="cart-empty">Your cart is empty.</p>
      <template v-else>
        <ul class="cart-items">
          <li v-for="(lesson, index) in cartItems" :key="index" class="cart-item">
            <span>{{ lesson.subject }} — &pound;{{ lesson.price }} per term</span>
            <button type="button" @click="removeFromCart(index)">Remove</button>
          </li>
        </ul>
        <p class="cart-total">Cart total: &pound;{{ cartTotal }}</p>
        <form class="checkout-form" @submit.prevent="checkout">
          <h3>Checkout</h3>
          <label>
            Full name
            <input v-model="customerName" type="text" required minlength="2" autocomplete="name" />
          </label>
          <label>
            Phone number
            <input v-model="customerPhone" type="tel" required pattern="[0-9+() .-]{7,20}" autocomplete="tel" />
          </label>
          <button type="submit" :disabled="isSubmitting">
            {{ isSubmitting ? 'Submitting order…' : 'Submit order' }}
          </button>
        </form>
      </template>
    </section>

    <template v-else>
    <p v-if="isLoading" class="status-message" role="status">Loading classes…</p>

    <div v-else-if="loadError" class="error-message" role="alert">
      <p>{{ loadError }}</p>
      <button type="button" @click="loadLessons">Try again</button>
    </div>

    <p v-else-if="lessons.length === 0" class="status-message">
      No classes are available right now.
    </p>

    <template v-else>
      <div class="sort-controls">
        <label>
          Sort lessons by
          <select v-model="sortBy">
            <option value="subject">Subject</option>
            <option value="location">Location</option>
            <option value="price">Price</option>
            <option value="spaces">Spaces</option>
          </select>
        </label>
        <label>
          Order
          <select v-model="sortDirection">
            <option value="asc">Ascending</option>
            <option value="desc">Descending</option>
          </select>
        </label>
      </div>

      <section class="lesson-grid" aria-label="Available coding classes">
        <article v-for="lesson in sortedLessons" :key="lesson.id" class="lesson-card">
          <div class="lesson-icon" aria-hidden="true">
            <i :class="lesson.icon || 'fa-solid fa-book-open'"></i>
          </div>
          <p class="lesson-location">{{ lesson.location }}</p>
          <h2>{{ lesson.subject }}</h2>
          <div class="lesson-details">
            <span>&pound;{{ lesson.price }} per term</span>
            <span>{{ lesson.spaces }} spaces</span>
          </div>
          <button type="button" :disabled="lesson.spaces <= 0" @click="addToCart(lesson)">
            Add to cart
          </button>
        </article>
      </section>
    </template>
    </template>
  </main>
</template>
