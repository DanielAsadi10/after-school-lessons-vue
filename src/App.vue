<script setup>
import { computed, onMounted, ref } from 'vue'

const lessons = ref([])
const isLoading = ref(true)
const loadError = ref('')
const apiBaseUrl = import.meta.env.VITE_API_URL || '/api'
const sortBy = ref('subject')
const sortDirection = ref('asc')
const searchTerm = ref('')
const cartItems = ref([])
const isCartOpen = ref(false)
const customerName = ref('')
const customerPhone = ref('')
const checkoutError = ref('')
const orderMessage = ref('')
const isSubmitting = ref(false)
const teachers = ref([])
const selectedTeacherId = ref('')
const teacherLessons = ref([])
const teacherEnrolment = ref(null)
const teacherError = ref('')
// Keep checkout enabled only when the name and phone match the accepted formats.
const checkoutValid = computed(() => {
  const name = customerName.value.trim()
  const phone = customerPhone.value.trim()
  return /^[A-Za-z ]{2,}$/.test(name) && /[A-Za-z]/.test(name) && /^\d{7,20}$/.test(phone)
})

// Sort either the full catalogue or the currently selected teacher's lessons.
const displayedLessons = computed(() => {
  const source = selectedTeacherId.value ? teacherLessons.value : lessons.value
  return [...source].sort((left, right) => {
    const leftValue = left[sortBy.value]
    const rightValue = right[sortBy.value]
    const comparison = typeof leftValue === 'number' && typeof rightValue === 'number'
      ? leftValue - rightValue
      : String(leftValue).localeCompare(String(rightValue), undefined, { sensitivity: 'base' })

    return sortDirection.value === 'asc' ? comparison : -comparison
  })
})

// Calculate the current cart total from its lesson entries.
const cartTotal = computed(() => {
  return cartItems.value.reduce((total, lesson) => total + Number(lesson.price || 0), 0)
})

// Resolve a lesson's teacher id to the name shown on its card.
function teacherName(teacherId) {
  const teacher = teachers.value.find((item) => item.id === teacherId)
  return teacher ? `${teacher.firstName} ${teacher.lastName}` : ''
}

// Add one available place to the cart and reserve it in the displayed stock.
function addToCart(lesson) {
  if (lesson.spaces <= 0) return

  lesson.spaces -= 1
  cartItems.value.push({ ...lesson })
}

// Remove a cart entry and return its place to the displayed lesson stock.
function removeFromCart(index) {
  const [removedLesson] = cartItems.value.splice(index, 1)
  const lesson = lessons.value.find((item) => item.id === removedLesson.id)

  if (lesson) lesson.spaces += 1
}

// Persist updated stock and the order, then refresh the catalogue on success.
async function checkout() {
  checkoutError.value = ''
  orderMessage.value = ''
  if (!checkoutValid.value) {
    checkoutError.value = 'Enter a name using letters and a phone number using 7–20 digits.'
    return
  }
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
    if (selectedTeacherId.value) await loadTeacherLessons()
  } catch (error) {
    checkoutError.value = error.message || 'We could not submit your order. Please try again.'
  } finally {
    isSubmitting.value = false
  }
}

// Fetch the lesson catalogue and expose loading or API errors to the page.
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

// Fetch teacher names and lesson counts for the teacher selector and cards.
async function loadTeachers() {
  try {
    const response = await fetch(`${apiBaseUrl}/teachers`)
    const result = await response.json()
    if (!response.ok) throw new Error(result.error || 'Could not load teachers.')
    teachers.value = result
  } catch (error) {
    console.error('Could not load teachers:', error)
    teacherError.value = error.message || 'Could not load teachers right now.'
  }
}

// Fetch a selected teacher's lessons and total enrolment count.
async function loadTeacherLessons() {
  teacherError.value = ''
  teacherEnrolment.value = null
  if (!selectedTeacherId.value) {
    teacherLessons.value = []
    return
  }

  try {
    const [lessonsResponse, enrolmentResponse] = await Promise.all([
      fetch(`${apiBaseUrl}/teachers/${selectedTeacherId.value}/lessons`),
      fetch(`${apiBaseUrl}/teachers/${selectedTeacherId.value}/enrolment`),
    ])
    const lessonResult = await lessonsResponse.json()
    const enrolmentResult = await enrolmentResponse.json()
    if (!lessonsResponse.ok) throw new Error(lessonResult.error || 'Could not load this teacher’s lessons.')
    if (!enrolmentResponse.ok) throw new Error(enrolmentResult.error || 'Could not load enrolment information.')
    teacherLessons.value = lessonResult
    teacherEnrolment.value = enrolmentResult.enrolment
  } catch (error) {
    console.error('Could not load a teacher’s lessons:', error)
    teacherLessons.value = []
    teacherError.value = error.message || 'Could not load this teacher’s lessons right now.'
  }
}

// Search the API by subject, location, price, or available spaces.
async function searchLessons() {
  selectedTeacherId.value = ''
  teacherLessons.value = []
  const query = searchTerm.value.trim()
  if (!query) {
    await loadLessons()
    return
  }

  isLoading.value = true
  loadError.value = ''
  try {
    const response = await fetch(`${apiBaseUrl}/lessons/search?q=${encodeURIComponent(query)}`)
    const result = await response.json()
    if (!response.ok) throw new Error(result.error || 'The lesson search failed.')
    lessons.value = result
  } catch (error) {
    console.error('Could not search lessons:', error)
    loadError.value = error.message || 'We could not search classes right now. Please try again.'
  } finally {
    isLoading.value = false
  }
}

// Load the catalogue and teacher list when Vue first displays this page.
onMounted(() => {
  loadLessons()
  loadTeachers()
})
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
            Full name (letters only)
            <input v-model="customerName" type="text" required minlength="2" pattern="[A-Za-z ]{2,}" autocomplete="name" />
          </label>
          <label>
            Phone number (7–20 digits)
            <input v-model="customerPhone" type="tel" required pattern="[0-9]{7,20}" autocomplete="tel" />
          </label>
          <button type="submit" :disabled="isSubmitting || !checkoutValid">
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
        <form class="search-form" @submit.prevent="searchLessons">
          <label for="lesson-search">Search by subject, location, price or spaces</label>
          <div class="search-input-row">
            <input id="lesson-search" v-model="searchTerm" type="search" maxlength="100" />
            <button type="submit">Search</button>
            <button class="clear-search" type="button" @click="searchTerm = ''; loadLessons()">Clear</button>
          </div>
        </form>
        <label>
          Browse by teacher
          <select v-model="selectedTeacherId" @change="loadTeacherLessons">
            <option value="">All teachers</option>
            <option v-for="teacher in teachers" :key="teacher.id" :value="String(teacher.id)">
              {{ teacher.firstName }} {{ teacher.lastName }} ({{ teacher.lessonCount }} classes)
            </option>
          </select>
        </label>
        <p v-if="teacherError" class="error-message teacher-message" role="alert">{{ teacherError }}</p>
        <p v-else-if="selectedTeacherId && teacherEnrolment !== null" class="teacher-message" role="status">
          Enrolments across this teacher’s classes: {{ teacherEnrolment }}
        </p>
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
        <article v-for="lesson in displayedLessons" :key="lesson.id" class="lesson-card">
          <div class="lesson-icon" aria-hidden="true">
            <i :class="lesson.icon || 'fa-solid fa-book-open'"></i>
          </div>
          <p class="lesson-location">{{ lesson.location }}</p>
          <p v-if="teacherName(lesson.teacherId)" class="lesson-teacher">
            Teacher: {{ teacherName(lesson.teacherId) }}
          </p>
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
