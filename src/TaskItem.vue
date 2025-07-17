<script setup>
import { defineProps, defineEmits } from 'vue'

const props = defineProps({
  task: Object,
  isNearest: Boolean
})

const emit = defineEmits(['updateTask', 'deleteTask'])

function isOverdue(task) {
  if (!task.due || task.done) return false
  return new Date(task.due) < new Date()
}

function onDoneChange(event) {
  props.task.done = event.target.checked
  emit('updateTask')
}

function onTextInput(event) {
  props.task.text = event.target.value
}

function onTextBlur() {
  emit('updateTask')
}

function onDueChange(event) {
  props.task.due = event.target.value
  emit('updateTask')
}

function onNoteInput(event) {
  props.task.note = event.target.value
}

function onNoteBlur() {
  emit('updateTask')
}

function onDeleteClick() {
  emit('deleteTask')
}
</script>

<template>
  <li
    :class="[
      'task-item',
      { overdue: isOverdue(task) && !task.done },
      { nearest: isNearest },
      { done: task.done }
    ]"
  >
    <input
      type="checkbox"
      :checked="task.done"
      @change="onDoneChange"
      :aria-label="`タスク完了状態 ${task.text}`"
    />
    <input
      type="text"
      :value="task.text"
      @input="onTextInput"
      @blur="onTextBlur"
      aria-label="タスク内容編集"
    />
    <input
      type="datetime-local"
      :value="task.due"
      @change="onDueChange"
      aria-label="締切日時編集"
    />
    <input
      type="text"
      :value="task.note"
      @input="onNoteInput"
      @blur="onNoteBlur"
      aria-label="メモ編集"
    />
    <button @click="onDeleteClick" :aria-label="`タスク削除 ${task.text}`">削除</button>
  </li>
</template>

<style scoped>
.task-item {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  margin: 0.5rem 0;
  padding: 0.5rem;
  background: #ffffff;
  border: 1px solid #ddd;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.06);
  border-left: 5px solid #007bff;
  border-radius: 6px;
  gap: 0.5rem;
  transition: background 0.3s ease;
}

.task-item:hover {
  background: #f9f9f9;
}

.task-item.done input[type="text"] {
  text-decoration: line-through;
  color: #888;
}

.task-item.overdue {
  color: red;
}

.task-item.nearest {
  background-color: #ffecec !important;
}

.task-item input[type="text"],
.task-item input[type="datetime-local"] {
  flex: 1 1 100px;
  padding: 0.3rem;
  font-size: 0.9rem;
}

.task-item button {
  padding: 0.3rem 0.6rem;
  font-size: 0.9rem;
  cursor: pointer;
}
</style>
