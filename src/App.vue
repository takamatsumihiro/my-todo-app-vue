<template>
  <div class="app-container">
    <h1>タスク管理アプリ</h1>

    <!-- 🔧 タイマー表示切替 -->
    <label class="timer-toggle">
      <input type="checkbox" v-model="showTimer" />
      1番期限の近いタスクの締切タイマーを表示
    </label>

    <!-- ⏱ タイマー -->
    <div v-if="showTimer && nearestTask" class="deadline-timer">
      🕒 締切間近: 「{{ nearestTask.text }}」
      <br />
      <span v-if="timeLeft.total > 0">
        残り: {{ timeLeft.hours }}時間 {{ timeLeft.minutes }}分 {{ timeLeft.seconds }}秒
      </span>
      <span v-else class="overdue-text">締切超過！</span>
    </div>

    <!-- フォーム -->
    <form class="task-form" @submit.prevent="addTask">
      <input v-model="newTask.text" placeholder="タスク内容" required />
      <input v-model="newTask.due" type="datetime-local" />
      <input v-model="newTask.note" placeholder="メモ" />
      <button type="submit">追加</button>
    </form>

    <!-- 一括操作 -->
    <div class="bulk-actions">
      <button @click="markAllDone">すべて完了にする</button>
    </div>

    <!-- フィルター -->
    <div class="filters">
      <button @click="setFilter('all')" :class="{ active: filter === 'all' }">すべて</button>
      <button @click="setFilter('active')" :class="{ active: filter === 'active' }">未完了</button>
      <button @click="setFilter('done')" :class="{ active: filter === 'done' }">完了</button>
    </div>

    <!-- タスクリスト -->
    <ul class="task-list">
      <li
        v-for="(task, index) in sortedTasks"
        :key="task.id"
        :class="{
          overdue: isOverdue(task) && !task.done,
          nearest: nearestTask && task.id === nearestTask.id,
          done: task.done
        }"
      >
        <input type="checkbox" v-model="task.done" @change="saveTasks" />
        <input v-model="task.text" @blur="saveTasks" />
        <input type="datetime-local" v-model="task.due" @change="saveTasks" />
        <input v-model="task.note" @blur="saveTasks" />
        <button @click="deleteTask(index)">削除</button>
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'

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

// タイマー機能
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

<style scoped>
.app-container {
  max-width: 600px;
  margin: auto;
  padding: 1rem;
  font-family: sans-serif;
}

h1 {
  text-align: center;
}

.task-form input,
.task-form button {
  margin: 0.3rem;
  padding: 0.4rem;
  font-size: 1rem;
}

.task-list {
  list-style: none;
  padding: 0;
}

.task-list li {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  margin: 0.5rem 0;
  padding: 0.5rem;
  background: #f7f7f7;
  border-radius: 6px;
  gap: 0.5rem;
}

.task-list li input[type="text"],
.task-list li input[type="datetime-local"],
.task-list li input[type="note"] {
  flex: 1 1 100px;
  padding: 0.3rem;
  font-size: 0.9rem;
}

.task-list li button {
  padding: 0.3rem 0.6rem;
  font-size: 0.9rem;
}

.done input[type="text"] {
  text-decoration: line-through;
  color: #888;
}

.overdue {
  color: red;
}

.nearest {
  background-color: #ffecec !important;
}

.filters button {
  margin-right: 0.5rem;
  padding: 0.3rem 0.8rem;
}

.filters .active {
  background-color: #007bff;
  color: white;
}

.bulk-actions {
  margin: 1rem 0;
}

.deadline-timer {
  margin: 1rem 0;
  padding: 0.6rem;
  background: #fff8e1;
  border: 1px solid #f2d96b;
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

@media (max-width: 600px) {
  .task-list li {
    flex-direction: column;
    align-items: flex-start;
  }

  .task-form input,
  .task-form button {
    width: 100%;
    margin-bottom: 0.4rem;
  }

  .task-list li input,
  .task-list li button {
    width: 100%;
  }
}
</style>
