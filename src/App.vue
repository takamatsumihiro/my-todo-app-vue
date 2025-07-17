<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import TaskItem from './TaskItem.vue'

const STORAGE_KEY = 'my-tasks'
const FILTER_KEY = 'task-filter'

const tasks = ref([])
const filter = ref(localStorage.getItem(FILTER_KEY) || 'all')
const newTask = ref({ text: '', due: '', note: '' })
const showTimer = ref(true)

function saveTasks() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks.value))
}

function loadTasks() {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    tasks.value = JSON.parse(saved)
  }
}

function addTask() {
  if (!newTask.value.text.trim()) return
  const due = newTask.value.due || ''
  tasks.value.push({
    id: Date.now(),
    text: newTask.value.text.trim(),
    due,
    note: newTask.value.note || '',
    done: false
  })
  newTask.value = { text: '', due: '', note: '' }
  saveTasks()
}

function deleteTask(index) {
  tasks.value.splice(index, 1)
  saveTasks()
}

function isOverdue(task) {
  if (!task.due || task.done) return false
  return new Date(task.due) < new Date()
}

function setFilter(value) {
  filter.value = value
  localStorage.setItem(FILTER_KEY, value)
}

function markAllDone() {
  tasks.value.forEach(t => (t.done = true))
  saveTasks()
}

const filteredTasks = computed(() => {
  if (filter.value === 'active') return tasks.value.filter(t => !t.done)
  if (filter.value === 'done') return tasks.value.filter(t => t.done)
  return tasks.value
})

const sortedTasks = computed(() => {
  return [...filteredTasks.value].sort((a, b) => {
    if (!a.due) return 1
    if (!b.due) return -1
    return new Date(a.due) - new Date(b.due)
  })
})

const nearestTask = computed(() => {
  return tasks.value
    .filter(t => !t.done && t.due)
    .sort((a, b) => new Date(a.due) - new Date(b.due))[0] || null
})

const timeLeft = ref({ total: 0, hours: 0, minutes: 0, seconds: 0 })
let timerInterval

function updateTimer() {
  if (!nearestTask.value || !nearestTask.value.due) return

  const now = new Date()
  const deadline = new Date(nearestTask.value.due)
  const diff = deadline - now

  timeLeft.value.total = diff
  timeLeft.value.hours = Math.floor(diff / (1000 * 60 * 60))
  timeLeft.value.minutes = Math.floor((diff / (1000 * 60)) % 60)
  timeLeft.value.seconds = Math.floor((diff / 1000) % 60)
}

onMounted(() => {
  loadTasks()
  updateTimer()
  timerInterval = setInterval(updateTimer, 1000)
})

onBeforeUnmount(() => {
  clearInterval(timerInterval)
})

watch(tasks, saveTasks, { deep: true })
</script>

<template>
  <div class="app-container">
    <h1>タスク管理アプリ</h1>

    <label class="timer-toggle">
      <input type="checkbox" v-model="showTimer" />
      1番期限の近いタスクの締切タイマーを表示
    </label>

    <div v-if="showTimer && nearestTask" class="deadline-timer" role="region" aria-live="polite" aria-atomic="true">
      🕒 締切間近: 「{{ nearestTask.text }}」<br />
      <span v-if="timeLeft.total > 0">
        残り: {{ timeLeft.hours }}時間 {{ timeLeft.minutes }}分 {{ timeLeft.seconds }}秒
      </span>
      <span v-else class="overdue-text">締切超過！</span>
    </div>

    <form class="task-form" @submit.prevent="addTask" aria-label="新しいタスクの追加フォーム">
      <input
        v-model="newTask.text"
        type="text"
        placeholder="タスク内容"
        required
        autocomplete="off"
      />
      <input
        v-model="newTask.due"
        type="datetime-local"
        autocomplete="off"
      />
      <input
        v-model="newTask.note"
        placeholder="メモ"
        autocomplete="off"
      />
      <button type="submit" aria-label="タスクを追加">追加</button>
    </form>

    <div class="bulk-actions">
      <button @click="markAllDone" aria-label="すべてのタスクを完了にする">すべて完了にする</button>
    </div>

    <div class="filters" role="group" aria-label="タスクの表示フィルター">
      <button
        @click="setFilter('all')"
        :class="{ active: filter === 'all' }"
        :aria-pressed="filter === 'all'"
      >すべて</button>
      <button
        @click="setFilter('active')"
        :class="{ active: filter === 'active' }"
        :aria-pressed="filter === 'active'"
      >未完了</button>
      <button
        @click="setFilter('done')"
        :class="{ active: filter === 'done' }"
        :aria-pressed="filter === 'done'"
      >完了</button>
    </div>

    <ul class="task-list" role="list" aria-label="タスクリスト">
      <TaskItem
        v-for="(task, index) in sortedTasks"
        :key="task.id"
        :task="task"
        :isNearest="nearestTask && task.id === nearestTask.id"
        @updateTask="saveTasks"
        @deleteTask="deleteTask(index)"
      />
    </ul>
  </div>
</template>

<style scoped>
.app-container {
  max-width: 600px;
  margin: auto;
  padding: 1rem;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

h1 {
  font-size: 2rem;
  margin-bottom: 1rem;
  color: #333;
  border-bottom: 2px solid #ddd;
  padding-bottom: 0.3rem;
}

.task-form {
  background: #f0f8ff;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  border: 1px solid #ccd;
}

.task-form input,
.task-form button {
  margin: 0.3rem;
  padding: 0.4rem;
  font-size: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.task-form button {
  background-color: #007bff;
  color: white;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.task-form button:hover {
  background-color: #0056b3;
}

.filters button {
  margin-right: 0.5rem;
  padding: 0.3rem 0.8rem;
  border-radius: 4px;
  border: 1px solid #ccc;
  background: #f4f4f4;
  cursor: pointer;
  transition: background 0.2s ease;
}

.filters button:hover {
  background: #e2e2e2;
}

.filters .active {
  background-color: #007bff !important;
  color: white;
}

.bulk-actions {
  margin: 1rem 0;
}

.deadline-timer {
  margin: 1rem 0;
  padding: 0.6rem;
  background: #fff4cc;
  border: 1px solid #f2d96b;
  border-left: 5px solid #f0b400;
  border-radius: 6px;
  font-weight: bold;
}

.overdue-text {
  color: red;
}

.timer-toggle {
  display: block;
  margin-bottom: 1rem;
}

.task-list {
  list-style: none;
  padding: 0;
}

/* モバイル対応 */
@media (max-width: 600px) {
  .task-form input,
  .task-form button,
  .filters button {
    width: 100%;
    margin-bottom: 0.4rem;
  }
}
</style>
