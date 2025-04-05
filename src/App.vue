<template>
  <div class="app-container">
    <div class="app-content">
      <h1 class="app-title">Nautilus</h1>

      <div class="main-grid">
        <div class="timer-panel">
          <h2 class="panel-title">Timer</h2>

          <div class="timer-display-container">
            <div class="timer-display" :class="timerMode">
              {{ formatTime(timer) }}
            </div>
          </div>

          <!-- 第一个新增区域：当前模式状态显示 -->
          <div class="timer-status-container">
            <div class="current-mode-status">
              <span class="status-label">Current Mode :</span>
              <span class="status-value" :class="timerMode">
                {{ getModeDisplayName(timerMode) }}
              </span>
            </div>
            <div class="circle-mode-status">
              <span class="status-label">Circle Mode:</span>
              <span class="status-value" :class="{ active: isInCircleMode }">
                {{ isInCircleMode ? "On" : "Off" }}
              </span>
            </div>
            <div v-if="isInCircleMode" class="circle-pattern-status">
              <span class="status-label">Circle Progress:</span>
              <span class="status-value"> {{ circleIndex + 1 }}/{{ circlePattern.length }} - {{ getModeDisplayName(circlePattern[circleIndex]) }} </span>
            </div>
          </div>

          <div class="timer-mode-buttons">
            <button @click="setTimerMode('pomodoro')" class="mode-button" :class="{ active: timerMode === 'pomodoro' }">Focus</button>
            <button @click="setTimerMode('shortBreak')" class="mode-button" :class="{ active: timerMode === 'shortBreak' }">Short</button>
            <button @click="setTimerMode('longBreak')" class="mode-button" :class="{ active: timerMode === 'longBreak' }">Long</button>
          </div>

          <div class="timer-control-buttons">
            <button @click="startTimer" class="control-button start-button" :disabled="isRunning">Start</button>
            <button @click="pauseTimer" class="control-button pause-button" :disabled="!isRunning">Pause</button>
            <button @click="resetTimer" class="control-button reset-button">Reset</button>
          </div>

          <div class="timer-control-buttons">
            <!-- 循环按钮 -->
            <button @click="circleTimer" class="control-button circle-button">Start a circle</button>
          </div>
        </div>

        <div class="todo-panel">
          <h2 class="panel-title">Tasks</h2>

          <div class="add-todo-form">
            <input v-model="newTodo" @keyup.enter="addTodo" type="text" placeholder="Add a new task..." class="todo-input" />
            <button @click="addTodo" class="add-button">Add</button>
          </div>

          <div class="progress-container">
            <!-- 进度信息：已完成/总数 和百分比 -->
            <div class="progress-info">
              <span>Progress: {{ completedCount }} of {{ todos.length }}</span>
              <span>{{ Math.round((completedCount / Math.max(todos.length, 1)) * 100) }}%</span>
            </div>
            <div class="progress-bar-bg">
              <div class="progress-bar-fill" :style="{ width: `${(completedCount / Math.max(todos.length, 1)) * 100}%` }"></div>
            </div>
          </div>

          <div class="todo-list-container">
            <div v-if="todos.length > 0" class="todo-list">
              <div v-for="(todo, index) in todos" :key="index" class="todo-item">
                <input type="checkbox" :checked="todo.completed" @change="toggleTodo(index)" class="todo-checkbox" />
                <span class="todo-text" :class="{ completed: todo.completed }">
                  {{ todo.text }}
                  {{ todo.createdAt ? `🗓️  ${new Date(todo.createdAt).toLocaleDateString()}` : "" }}
                </span>
                <div class="todo-actions">
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

// todo列表和输入框
const todos = ref([]);
const newTodo = ref("");

// 状态变量
const timerMode = ref("pomodoro");
const timerModes = {
  pomodoro: 25 * 60, // 25分钟
  shortBreak: 5 * 60, // 5分钟
  longBreak: 10 * 60, // 10分钟
};

// 当前计时器的剩余时间
const timer = ref(timerModes.pomodoro);
const isRunning = ref(false);

// 计时器间隔引用，用于清除定时器
const timerInterval = ref(null);

// 循环模式的状态
const circleIndex = ref(0);
const circlePattern = ["pomodoro", "shortBreak", "pomodoro", "longBreak"];
const isInCircleMode = ref(false);

// 计算已完成的任务数量
const completedCount = computed(() => {
  return todos.value.filter((todo) => todo.completed).length;
});

// 获取模式的显示名称
const getModeDisplayName = (mode) => {
  const modeNames = {
    pomodoro: "Focus",
    shortBreak: "Short",
    longBreak: "Long",
  };
  return modeNames[mode] || mode;
};

// 增加待办事项
const addTodo = () => {
  const text = newTodo.value.trim();
  if (text) {
    todos.value.push({
      text,
      completed: false,
      createdAt: new Date().toISOString(),
    });
    newTodo.value = "";
  }
};

// 切换待办事项的完成状态
const toggleTodo = (index) => {
  todos.value[index].completed = !todos.value[index].completed;
};

// 删除待办事项
const removeTodo = (index) => {
  todos.value.splice(index, 1);
};

// 开始当前计时器
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

// 暂停当前计时器
const pauseTimer = () => {
  if (!isRunning.value) return;

  isRunning.value = false;
  clearInterval(timerInterval.value);
};

// 重置当前计时器
const resetTimer = () => {
  pauseTimer();
  isInCircleMode.value = false;
  timer.value = timerModes[timerMode.value];
};

// 循环计时器模式
const circleTimer = () => {
  isInCircleMode.value = true;
  circleIndex.value = 0;

  setTimerMode(circlePattern[circleIndex.value]);
  isInCircleMode.value = true;

  startTimer();

  const checkTimerInterval = setInterval(() => {
    if (!isRunning.value && isInCircleMode.value && timer.value === 0) {
      circleIndex.value = (circleIndex.value + 1) % circlePattern.length;

      setTimerMode(circlePattern[circleIndex.value]);
      isInCircleMode.value = true;

      setTimeout(() => {
        startTimer();
      }, 1000);
    }

    if (!isInCircleMode.value) {
      clearInterval(checkTimerInterval);
    }
  }, 500);
};

// 设置计时器模式
const setTimerMode = (mode) => {
  timerMode.value = mode;
  resetTimer();
};

// 格式化时间为 mm:ss，用于时钟
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

// 保存待办事项到文件
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

// 从文件加载待办事项
const loadTodos = async () => {
  try {
    const loadsDir = await ensureLoadsDirectory();
    const filePath = await join(loadsDir, "todos.json");

    const fileExists = await exists(filePath);
    if (!fileExists) {
      console.log("待办事项文件尚不存在");
      return;
    }

    const todosJson = await readTextFile(filePath);

    if (todosJson) {
      todos.value = JSON.parse(todosJson);
      console.log("待办事项加载成功:", filePath);
    }
  } catch (error) {
    console.error("加载待办事项时出错:", error);
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
