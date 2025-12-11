<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'

// Theme
const theme = ref<'light' | 'dark'>('light')

function initTheme() {
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
  theme.value = prefersDark ? 'dark' : 'light'
}

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

// Listen for system theme changes
let mediaQuery: MediaQueryList | null = null

onMounted(() => {
  initTheme()
  mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
  mediaQuery.addEventListener('change', (e) => {
    theme.value = e.matches ? 'dark' : 'light'
  })
})

onUnmounted(() => {
  if (mediaQuery) {
    mediaQuery.removeEventListener('change', () => {})
  }
})

// Timer state
const inputHours = ref(0)
const inputMinutes = ref(0)
const inputSeconds = ref(0)

const originalTime = ref(0) // in seconds
const remainingTime = ref(0) // in seconds
const isRunning = ref(false)
const hasStarted = ref(false)

let intervalId: number | null = null

// Computed display values
const displayHours = computed(() => Math.floor(remainingTime.value / 3600))
const displayMinutes = computed(() => Math.floor((remainingTime.value % 3600) / 60))
const displaySeconds = computed(() => remainingTime.value % 60)

const formattedTime = computed(() => {
  const h = String(displayHours.value).padStart(2, '0')
  const m = String(displayMinutes.value).padStart(2, '0')
  const s = String(displaySeconds.value).padStart(2, '0')
  return `${h}:${m}:${s}`
})

const canStart = computed(() => {
  if (hasStarted.value) return remainingTime.value > 0
  return inputHours.value > 0 || inputMinutes.value > 0 || inputSeconds.value > 0
})

// Progress ring calculations
const ringRadius = 140
const ringCircumference = 2 * Math.PI * ringRadius

const progressOffset = computed(() => {
  if (!hasStarted.value || originalTime.value === 0) {
    return 0 // Full circle when not started
  }
  const progress = remainingTime.value / originalTime.value
  return ringCircumference * (1 - progress)
})

// Update page title when running
watch([isRunning, formattedTime], ([running, time]) => {
  if (running) {
    document.title = `${time} - Timer`
  } else {
    document.title = 'Timer'
  }
})

// Timer functions
function startTimer() {
  if (!hasStarted.value) {
    // First start - calculate total seconds from input
    const totalSeconds = inputHours.value * 3600 + inputMinutes.value * 60 + inputSeconds.value
    if (totalSeconds <= 0) return
    originalTime.value = totalSeconds
    remainingTime.value = totalSeconds
    hasStarted.value = true
  }

  if (remainingTime.value <= 0) return

  isRunning.value = true
  intervalId = window.setInterval(() => {
    if (remainingTime.value > 0) {
      remainingTime.value--
    }
    if (remainingTime.value <= 0) {
      pauseTimer()
    }
  }, 1000)
}

function pauseTimer() {
  isRunning.value = false
  if (intervalId !== null) {
    clearInterval(intervalId)
    intervalId = null
  }
}

function resetTimer() {
  pauseTimer()
  remainingTime.value = originalTime.value
}

function deleteTimer() {
  pauseTimer()
  hasStarted.value = false
  remainingTime.value = 0
  originalTime.value = 0
  inputHours.value = 0
  inputMinutes.value = 0
  inputSeconds.value = 0
}

// Cleanup on unmount
onUnmounted(() => {
  if (intervalId !== null) {
    clearInterval(intervalId)
  }
})
</script>

<template>
  <div class="app" :class="theme">
    <button class="theme-toggle" @click="toggleTheme" :title="`Switch to ${theme === 'light' ? 'dark' : 'light'} mode`">
      <span v-if="theme === 'light'">&#9790;</span>
      <span v-else>&#9788;</span>
    </button>

    <main class="container">
      <div class="ring-wrapper">
        <svg class="progress-ring" viewBox="0 0 320 320">
          <!-- Background circle (grey) -->
          <circle
            class="progress-ring-bg"
            cx="160"
            cy="160"
            :r="ringRadius"
            fill="none"
            stroke-width="12"
          />
          <!-- Progress circle (colored) -->
          <circle
            class="progress-ring-progress"
            cx="160"
            cy="160"
            :r="ringRadius"
            fill="none"
            stroke-width="12"
            :stroke-dasharray="ringCircumference"
            :stroke-dashoffset="progressOffset"
            stroke-linecap="round"
          />
        </svg>

        <div class="ring-content">
          <h1>Timer</h1>

          <!-- Input form (shown when timer hasn't started) -->
          <div v-if="!hasStarted" class="input-section">
            <div class="time-inputs">
              <div class="input-group">
                <label for="hours">Hours</label>
                <input
                  id="hours"
                  type="number"
                  v-model.number="inputHours"
                  min="0"
                  max="99"
                />
              </div>
              <div class="input-group">
                <label for="minutes">Minutes</label>
                <input
                  id="minutes"
                  type="number"
                  v-model.number="inputMinutes"
                  min="0"
                  max="59"
                />
              </div>
              <div class="input-group">
                <label for="seconds">Seconds</label>
                <input
                  id="seconds"
                  type="number"
                  v-model.number="inputSeconds"
                  min="0"
                  max="59"
                />
              </div>
            </div>
            <button class="btn btn-primary" @click="startTimer" :disabled="!canStart">
              Start
            </button>
          </div>

          <!-- Timer display (shown when timer has started) -->
          <div v-else class="timer-section">
            <div class="timer-display" :class="{ finished: remainingTime === 0 }">
              {{ formattedTime }}
            </div>
            <div class="controls">
              <button
                v-if="!isRunning && remainingTime > 0"
                class="btn btn-primary"
                @click="startTimer"
              >
                {{ remainingTime === originalTime ? 'Start' : 'Resume' }}
              </button>
              <button
                v-if="isRunning"
                class="btn btn-secondary"
                @click="pauseTimer"
              >
                Pause
              </button>
              <button
                class="btn btn-secondary"
                @click="resetTimer"
                :disabled="remainingTime === originalTime"
              >
                Reset
              </button>
              <button class="btn btn-danger" @click="deleteTimer">
                Delete
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  height: 100%;
}

#app {
  height: 100%;
}

.app {
  min-height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  transition: background-color 0.3s, color 0.3s;
  position: relative;
}

/* Light theme */
.app.light {
  background-color: #f5f5f5;
  color: #1a1a1a;
}

.app.light .btn {
  background-color: #e0e0e0;
  color: #1a1a1a;
}

.app.light .btn-primary {
  background-color: #2563eb;
  color: white;
}

.app.light .btn-danger {
  background-color: #dc2626;
  color: white;
}

.app.light input {
  background-color: white;
  border-color: #d1d5db;
  color: #1a1a1a;
}

/* Dark theme */
.app.dark {
  background-color: #1a1a1a;
  color: #f5f5f5;
}

.app.dark .btn {
  background-color: #374151;
  color: #f5f5f5;
}

.app.dark .btn-primary {
  background-color: #3b82f6;
  color: white;
}

.app.dark .btn-danger {
  background-color: #ef4444;
  color: white;
}

.app.dark input {
  background-color: #374151;
  border-color: #4b5563;
  color: #f5f5f5;
}

/* Theme toggle */
.theme-toggle {
  position: absolute;
  top: 1rem;
  right: 1rem;
  background: transparent;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0.5rem;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.theme-toggle:hover {
  opacity: 1;
}

.app.light .theme-toggle {
  color: #1a1a1a;
}

.app.dark .theme-toggle {
  color: #f5f5f5;
}

/* Container */
.container {
  text-align: center;
  width: 100%;
  max-width: 400px;
}

/* Ring wrapper */
.ring-wrapper {
  position: relative;
  width: 320px;
  height: 320px;
  margin: 0 auto;
}

.progress-ring {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transform: rotate(-90deg);
}

.progress-ring-bg {
  stroke: #d1d5db;
}

.progress-ring-progress {
  stroke: #2563eb;
  transition: stroke-dashoffset 0.3s ease;
}

.app.dark .progress-ring-bg {
  stroke: #374151;
}

.app.dark .progress-ring-progress {
  stroke: #3b82f6;
}

.ring-content {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 240px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

h1 {
  font-size: 1.5rem;
  margin-bottom: 1rem;
  font-weight: 300;
}

/* Input section */
.input-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.time-inputs {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.input-group label {
  font-size: 0.7rem;
  opacity: 0.8;
}

.input-group input {
  width: 60px;
  padding: 0.5rem;
  font-size: 1rem;
  text-align: center;
  border: 1px solid;
  border-radius: 0.375rem;
  transition: border-color 0.2s, background-color 0.3s;
}

.input-group input:focus {
  outline: none;
  border-color: #3b82f6;
}

/* Timer display */
.timer-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.timer-display {
  font-size: 2.5rem;
  font-family: 'Courier New', monospace;
  font-weight: bold;
  letter-spacing: 0.05em;
}

.timer-display.finished {
  opacity: 0.5;
}

/* Controls */
.controls {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
  flex-wrap: wrap;
}

/* Buttons */
.btn {
  padding: 0.5rem 0.75rem;
  font-size: 0.85rem;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: opacity 0.2s, transform 0.1s;
}

.btn:hover:not(:disabled) {
  opacity: 0.9;
}

.btn:active:not(:disabled) {
  transform: scale(0.98);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Responsive */
@media (max-width: 400px) {
  .ring-wrapper {
    width: 280px;
    height: 280px;
  }

  .ring-content {
    width: 200px;
  }

  .time-inputs {
    flex-direction: column;
    align-items: center;
  }

  .timer-display {
    font-size: 2rem;
  }
}
</style>
