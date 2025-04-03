<template>
  <div class="app-container">
    <div class="app-content">
      <h1 class="app-title">TideTask</h1>

      <div class="main-grid">
        <!-- 计时器面板 -->
        <div class="timer-panel">
          <h2 class="panel-title">Timer</h2>

          <!-- 计时器显示容器 -->
          <div class="timer-display-container">
            <!-- 计时器显示，根据当前模式添加不同的CSS类 -->
            <div class="timer-display" :class="timerMode">
              {{ formatTime(timer) }}
            </div>
          </div>

          <!-- 计时器模式按钮组 -->
          <div class="timer-mode-buttons">
            <!-- 专注模式按钮（番茄工作时段） -->
            <!-- 通过active类切换模式 -->
            <button @click="setTimerMode('pomodoro')" class="mode-button" :class="{ active: timerMode === 'pomodoro' }">Focus</button>
            <!-- 短休息模式按钮 -->
            <button @click="setTimerMode('shortBreak')" class="mode-button" :class="{ active: timerMode === 'shortBreak' }">Short</button>
            <!-- 长休息模式按钮 -->
            <button @click="setTimerMode('longBreak')" class="mode-button" :class="{ active: timerMode === 'longBreak' }">Long</button>
            <!-- 考虑增加专注 - 短修 - 专注 - 长修 循环 -->
          </div>

          <!-- 计时器控制按钮组 -->
          <div class="timer-control-buttons">
            <!-- 开始按钮 - 计时器运行时禁用 -->
            <button @click="startTimer" class="control-button start-button" :disabled="isRunning">Start</button>
            <!-- 暂停按钮 - 计时器未运行时禁用 -->
            <button @click="pauseTimer" class="control-button pause-button" :disabled="!isRunning">Pause</button>
            <!-- 重置按钮 -->
            <button @click="resetTimer" class="control-button reset-button">Reset</button>
          </div>
        </div>

        <!-- 待办事项列表面板 -->
        <div class="todo-panel">
          <h2 class="panel-title">Tasks</h2>

          <!-- 添加待办事项表单 -->
          <div class="add-todo-form">
            <!-- 输入框，按Enter键可添加任务 -->
            <input v-model="newTodo" @keyup.enter="addTodo" type="text" placeholder="Add a new task..." class="todo-input" />
            <!-- 添加按钮 -->
            <button @click="addTodo" class="add-button">Add</button>
          </div>

          <!-- 进度条区域 -->
          <div class="progress-container">
            <!-- 进度信息：已完成/总数 和百分比 -->
            <div class="progress-info">
              <span>Progress: {{ completedCount }} of {{ todos.length }}</span>
              <span>{{ Math.round((completedCount / Math.max(todos.length, 1)) * 100) }}%</span>
            </div>
            <!-- 进度条背景 -->
            <div class="progress-bar-bg">
              <!-- 进度条填充部分，宽度根据完成百分比动态计算 -->
              <div class="progress-bar-fill" :style="{ width: `${(completedCount / Math.max(todos.length, 1)) * 100}%` }"></div>
            </div>
          </div>

          <!-- 待办事项列表容器 -->
          <div class="todo-list-container">
            <!-- 当有待办事项时显示列表 -->
            <div v-if="todos.length > 0" class="todo-list">
              <!-- 遍历待办事项数组，为每个项目创建元素 -->
              <div v-for="(todo, index) in todos" :key="index" class="todo-item">
                <!-- 任务完成状态复选框 -->
                <input type="checkbox" :checked="todo.completed" @change="toggleTodo(index)" class="todo-checkbox" />
                <!-- 任务文本，完成时添加completed类 -->
                <span class="todo-text" :class="{ completed: todo.completed }">
                  {{ todo.text }}
                  {{ todo.createdAt ? `🗓️  ${new Date(todo.createdAt).toLocaleDateString()}` : "" }}
                </span>
                <!-- 任务操作按钮区域 -->
                <div class="todo-actions">
                  <!-- 删除任务按钮 -->
                  <button @click="removeTodo(index)" class="todo-action-button delete-button">
                    <!-- 垃圾桶图标 SVG -->
                    <svg xmlns="http://www.w3.org/2000/svg" class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <polyline points="3 6 5 6 21 6"></polyline>
                      <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                      <line x1="10" y1="11" x2="10" y2="17"></line>
                      <line x1="14" y1="11" x2="14" y2="17"></line>
                    </svg>
                  </button>
                </div>
              </div>
            </div>

            <!-- 空状态 - 无任务时显示 -->
            <div v-else class="empty-state">
              <!-- 空列表图标 SVG -->
              <svg xmlns="http://www.w3.org/2000/svg" class="empty-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                <polyline points="14 2 14 8 20 8"></polyline>
                <line x1="16" y1="13" x2="8" y2="13"></line>
                <line x1="16" y1="17" x2="8" y2="17"></line>
                <polyline points="10 9 9 9 8 9"></polyline>
              </svg>
              <p>No tasks yet. Add one above!</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { appDataDir, join } from "@tauri-apps/api/path";
import { exists, mkdir, readTextFile, writeTextFile } from "@tauri-apps/plugin-fs";
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";
const todos = ref([]);
const newTodo = ref("");

const timerMode = ref("pomodoro");
const timerModes = {
  pomodoro: 25 * 60, // 25分钟的专注时间
  shortBreak: 5 * 60, // 5分钟的短休息
  longBreak: 10 * 60, // 10分钟的长休息
};

// 当前计时器的剩余时间
const timer = ref(timerModes.pomodoro);
const isRunning = ref(false);

// 计时器间隔引用，用于清除定时器
const timerInterval = ref(null);

const completedCount = computed(() => {
  return todos.value.filter((todo) => todo.completed).length;
});

const addTodo = () => {
  const text = newTodo.value.trim();
  if (text) {
    todos.value.push({
      text,
      completed: false,
      createdAt: new Date().toISOString(), // 创建时间
    });
    newTodo.value = ""; // 清空输入框
  }
};

const toggleTodo = (index) => {
  todos.value[index].completed = !todos.value[index].completed;
};

const removeTodo = (index) => {
  todos.value.splice(index, 1);
};

const startTimer = () => {
  if (isRunning.value) return;

  isRunning.value = true;
  timerInterval.value = setInterval(() => {
    if (timer.value > 0) {
      timer.value--;
    } else {
      pauseTimer();
    }
  }, 1000);
};

const pauseTimer = () => {
  if (!isRunning.value) return;

  isRunning.value = false;
  clearInterval(timerInterval.value);
};

const resetTimer = () => {
  pauseTimer();
  timer.value = timerModes[timerMode.value];
};

const setTimerMode = (mode) => {
  timerMode.value = mode;
  resetTimer();
};

const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins.toString().padStart(2, "0")}:${secs.toString().padStart(2, "0")}`;
};

// 确保loads目录存在
const ensureLoadsDirectory = async () => {
  try {
    const appDataDirPath = await appDataDir();
    const loadsDir = await join(appDataDirPath, "loads");

    // 检查目录是否存在，如果不存在则创建
    const dirExists = await exists(loadsDir);
    if (!dirExists) {
      await mkdir(loadsDir, { recursive: true });
      console.log("创建loads目录成功:", loadsDir);
    }

    return loadsDir;
  } catch (error) {
    console.error("确保loads目录存在时出错:", error);
    throw error;
  }
};

const saveTodos = async () => {
  try {
    const loadsDir = await ensureLoadsDirectory();
    const filePath = await join(loadsDir, "todos.json");

    const todosJson = JSON.stringify(todos.value);

    await writeTextFile(filePath, todosJson);
    console.log("待办事项保存成功:", filePath);
  } catch (error) {
    console.error("保存待办事项时出错:", error);
  }
};

const loadTodos = async () => {
  try {
    const loadsDir = await ensureLoadsDirectory();
    const filePath = await join(loadsDir, "todos.json");

    // 检查文件是否存在
    const fileExists = await exists(filePath);
    if (!fileExists) {
      console.log("待办事项文件尚不存在");
      return;
    }

    // 从文件读取待办事项
    const todosJson = await readTextFile(filePath);

    // 解析JSON并更新待办事项引用
    if (todosJson) {
      todos.value = JSON.parse(todosJson);
      console.log("待办事项加载成功:", filePath);
    }
  } catch (error) {
    console.error("加载待办事项时出错:", error);
    // 如果加载出错，初始化为空数组
    todos.value = [];
  }
};

// 组件挂载后，加载待办事项和请求通知权限
onMounted(() => {
  loadTodos();
});

// 组件卸载前，清除计时器
onBeforeUnmount(() => {
  if (timerInterval.value) {
    clearInterval(timerInterval.value);
  }
});

// 监听待办事项数组的变化，保存到文件
watch(
  todos,
  () => {
    saveTodos();
  },
  { deep: true }
);
</script>

<style>
@import "./assets/styles/main.css";
</style>
