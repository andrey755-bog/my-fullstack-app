<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import api from '../services/api';

const tasks = ref([]);
const newTaskTitle = ref('');
const errorMessage = ref('');

const filter = ref(localStorage.getItem('taskFilter') || 'all');
const sortBy = ref(localStorage.getItem('taskSort') || 'none');

watch(filter, v => localStorage.setItem('taskFilter', v));
watch(sortBy, v => localStorage.setItem('taskSort', v));

const handleError = (error, fallback) => {
  if (!error.response) errorMessage.value = 'Ошибка сети';
  else if (error.response.status === 400) errorMessage.value = 'Ошибка валидации';
  else if (error.response.status === 500) errorMessage.value = 'Ошибка сервера';
  else errorMessage.value = fallback;
  console.error(error);
};

const fetchTasks = async () => {
  try {
    const response = await api.getTasks();
    tasks.value = response.data;
  } catch (error) {
    handleError(error, 'Не удалось загрузить задачи');
  }
};

const addTask = async () => {
  if (!newTaskTitle.value.trim()) {
    errorMessage.value = 'Название не может быть пустым';
    return;
  }
  try {
    await api.createTask({ title: newTaskTitle.value, completed: false });
    newTaskTitle.value = '';
    errorMessage.value = '';
    await fetchTasks();
  } catch (error) {
    handleError(error, 'Не удалось создать задачу');
  }
};

const toggleTask = async (task) => {
  try {
    await api.patchTask(task.id, { completed: !task.completed });
    await fetchTasks();
  } catch (error) {
    handleError(error, 'Не удалось изменить статус');
  }
};

const editTask = async (task) => {
  const newTitle = prompt('Новое название:', task.title);
  if (newTitle === null) return;
  if (!newTitle.trim()) {
    errorMessage.value = 'Название пустое';
    return;
  }
  try {
    await api.patchTask(task.id, { title: newTitle.trim() });
    await fetchTasks();
  } catch (error) {
    handleError(error, 'Не удалось отредактировать');
  }
};

const deleteTask = async (id) => {
  if (!confirm('Удалить задачу?')) return;
  try {
    await api.deleteTask(id);
    await fetchTasks();
  } catch (error) {
    handleError(error, 'Не удалось удалить');
  }
};

const displayedTasks = computed(() => {
  let list = tasks.value;
  if (filter.value === 'active') list = list.filter(t => !t.completed);
  if (filter.value === 'completed') list = list.filter(t => t.completed);
  list = [...list];
  if (sortBy.value === 'title-asc') list.sort((a, b) => a.title.localeCompare(b.title, 'ru'));
  if (sortBy.value === 'title-desc') list.sort((a, b) => b.title.localeCompare(a.title, 'ru'));
  if (sortBy.value === 'active-first') list.sort((a, b) => a.completed - b.completed);
  if (sortBy.value === 'completed-first') list.sort((a, b) => b.completed - a.completed);
  return list;
});

const stats = computed(() => ({
  total: tasks.value.length,
  active: tasks.value.filter(t => !t.completed).length,
  completed: tasks.value.filter(t => t.completed).length,
}));

onMounted(fetchTasks);
</script>

<template>
  <div class="container">
    <h1>Мои задачи</h1>

    <div v-if="errorMessage" class="error">
      ⚠️ {{ errorMessage }}
      <button @click="errorMessage = ''">✕</button>
    </div>

    <div class="add-form">
      <input v-model="newTaskTitle" @keyup.enter="addTask" placeholder="Новая задача" />
      <button @click="addTask">Добавить</button>
    </div>

    <p class="stats">
      Всего: {{ stats.total }} · Активных: {{ stats.active }} · Выполненных: {{ stats.completed }}
    </p>

    <div class="filters">
      <button :class="{ active: filter === 'all' }" @click="filter = 'all'">Все</button>
      <button :class="{ active: filter === 'active' }" @click="filter = 'active'">Активные</button>
      <button :class="{ active: filter === 'completed' }" @click="filter = 'completed'">Выполненные</button>
    </div>

    <select v-model="sortBy">
      <option value="none">Без сортировки</option>
      <option value="title-asc">А–Я</option>
      <option value="title-desc">Я–А</option>
      <option value="active-first">Сначала активные</option>
      <option value="completed-first">Сначала выполненные</option>
    </select>

    <ul class="task-list">
      <li
        v-for="task in displayedTasks"
        :key="task.id"
        class="task-card"
        :class="{ completed: task.completed }"
      >
        <input type="checkbox" :checked="task.completed" @change="toggleTask(task)" />
        <span>{{ task.title }}</span>
        <button @click="editTask(task)">изменить</button>
        <button class="delete" @click="deleteTask(task.id)">Удалить</button>
      </li>
    </ul>
  </div>
</template>

<style scoped>
@import '../assets/tasklist.css';
</style>