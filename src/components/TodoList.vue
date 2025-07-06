<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

interface TodoItem {
  id: number
  title: string
  priority: 'high' | 'medium' | 'low'
  completed: boolean
  createdAt: string
  note?: string
}

const todos = ref<TodoItem[]>([])
const showModal = ref(false)
const editingTodo = ref<TodoItem | null>(null)
const filterStatus = ref<'all' | 'completed' | 'active'>('all')
const sortBy = ref<'time' | 'priority'>('time')
const searchQuery = ref('')

const newTodo = ref({
  title: '',
  priority: 'medium',
  note: ''
})

const priorityColors = {
  high: 'text-red-600',
  medium: 'text-yellow-600',
  low: 'text-green-600'
}

onMounted(() => {
  const savedTodos = localStorage.getItem('todos')
  if (savedTodos) {
    todos.value = JSON.parse(savedTodos)
  }
})

const saveTodos = () => {
  localStorage.setItem('todos', JSON.stringify(todos.value))
}

const addTodo = () => {
  if (!newTodo.value.title.trim()) return

  const todo: TodoItem = {
    id: Date.now(),
    title: newTodo.value.title,
    priority: newTodo.value.priority as 'high' | 'medium' | 'low',
    completed: false,
    createdAt: new Date().toLocaleString(),
    note: newTodo.value.note
  }

  todos.value.push(todo)
  saveTodos()
  newTodo.value = {
    title: '',
    priority: 'medium',
    note: ''
  }
}

const deleteTodo = (id: number) => {
  todos.value = todos.value.filter(todo => todo.id !== id)
  saveTodos()
}

const showClearConfirm = ref(false)
const clearTodos = () => {
  todos.value = []
  saveTodos()
  showClearConfirm.value = false
}

const editTodo = (todo: TodoItem) => {
  editingTodo.value = { ...todo }
  showModal.value = true
}

const updateTodo = () => {
  if (!editingTodo.value) return
  
  const index = todos.value.findIndex(t => t.id === editingTodo.value?.id)
  if (index !== -1) {
    todos.value[index] = { ...editingTodo.value }
    saveTodos()
  }
  showModal.value = false
}

const toggleComplete = (todo: TodoItem) => {
  todo.completed = !todo.completed
  saveTodos()
}

const filteredAndSortedTodos = computed(() => {
  let filtered = todos.value

  // Apply search filter
  if (searchQuery.value) {
    filtered = filtered.filter(todo =>
      todo.title.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      todo.note?.toLowerCase().includes(searchQuery.value.toLowerCase())
    )
  }

  // Apply status filter
  if (filterStatus.value !== 'all') {
    filtered = filtered.filter(todo =>
      filterStatus.value === 'completed' ? todo.completed : !todo.completed
    )
  }

  // Apply sorting
  const priorityOrder = { high: 3, medium: 2, low: 1 }
  return filtered.sort((a, b) => {
    if (sortBy.value === 'priority') {
      return priorityOrder[b.priority] - priorityOrder[a.priority]
    } else {
      return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
    }
  })
})
</script>

<template>
  <div class="todo-root">
    <!-- Filters and Search -->
    <div class="todo-panel">
      <div class="todo-panel-row">
        <label class="todo-label">排序依赖</label>
        <select v-model="sortBy" class="todo-select">
          <option value="time">时间</option>
          <option value="priority">优先级</option>
        </select>
        <span class="todo-divider">筛选</span>
        <select v-model="filterStatus" class="todo-select">
          <option value="all">全部</option>
          <option value="completed">已完成</option>
          <option value="active">未完成</option>
        </select>
        <button class="todo-btn-clear" @click="showClearConfirm = true" style="margin-left:auto;">清空所有</button>
      </div>
      <div class="todo-panel-row">
        <input v-model="searchQuery" type="text" placeholder="搜索待办事项..." class="todo-search" />
      </div>
    </div>
    <!-- Todo List -->
    <div class="todo-list">
      <div class="todo-list-header">
        <span class="todo-list-col todo-list-col-checkbox"></span>
        <span class="todo-list-col todo-list-col-priority">优先级</span>
        <span class="todo-list-col todo-list-col-title">标题</span>
        <span class="todo-list-col todo-list-col-time">时间</span>
        <span class="todo-list-col todo-list-col-action">操作</span>
      </div>
      <TransitionGroup name="list">
        <div v-for="todo in filteredAndSortedTodos" :key="todo.id" class="todo-list-row" :class="{ 'todo-completed': todo.completed }">
          <span class="todo-list-col todo-list-col-checkbox">
            <input type="checkbox" :checked="todo.completed" @change="toggleComplete(todo)" />
          </span>
          <span class="todo-list-col todo-list-col-priority">
            <span :class="['todo-priority', todo.priority]">
              {{ {'high': '高', 'medium': '中', 'low': '低'}[todo.priority] }}
            </span>
          </span>
          <span class="todo-list-col todo-list-col-title">
            <span class="todo-title" :class="{ 'todo-title-completed': todo.completed }">{{ todo.title }}</span>
            <span v-if="todo.note" class="todo-note">{{ todo.note }}</span>
          </span>
          <span class="todo-list-col todo-list-col-time">{{ todo.createdAt }}</span>
          <span class="todo-list-col todo-list-col-action">
            <button class="todo-btn-edit" @click="editTodo(todo)">编辑</button>
            <button class="todo-btn-delete" @click="deleteTodo(todo.id)">删除</button>
          </span>
        </div>
      </TransitionGroup>
    </div>
    <!-- Add Todo Form -->
    <div class="todo-form">
      <div class="todo-form-title">添加新待办</div>
      <div class="todo-form-row">
        <input v-model="newTodo.title" type="text" placeholder="标题" class="todo-input" required />
      </div>
      <div class="todo-form-row">
        <select v-model="newTodo.priority" class="todo-select">
          <option value="high">高</option>
          <option value="medium">中</option>
          <option value="low">低</option>
        </select>
      </div>
      <div class="todo-form-row">
        <input v-model="newTodo.note" type="text" placeholder="备注" class="todo-input" />
      </div>
      <div class="todo-form-row">
        <button @click="addTodo" class="todo-btn-add">添加</button>
      </div>
    </div>
    <!-- Edit Modal -->
    <Teleport to="body">
      <div v-if="showModal" class="todo-modal-mask">
        <div class="todo-modal">
          <div class="todo-modal-title">编辑待办</div>
          <div class="todo-form-row">
            <input v-model="editingTodo.title" type="text" placeholder="标题" class="todo-input" required />
          </div>
          <div class="todo-form-row">
            <select v-model="editingTodo.priority" class="todo-select">
              <option value="high">高</option>
              <option value="medium">中</option>
              <option value="low">低</option>
            </select>
          </div>
          <div class="todo-form-row">
            <input v-model="editingTodo.note" type="text" placeholder="备注" class="todo-input" />
          </div>
          <div class="todo-form-row todo-modal-actions">
            <button @click="showModal = false" class="todo-btn-cancel">取消</button>
            <button @click="updateTodo" class="todo-btn-save">保存</button>
          </div>
        </div>
      </div>
      <div v-if="showClearConfirm" class="todo-modal-mask">
        <div class="todo-modal">
          <div class="todo-modal-title">确认清空所有待办？</div>
          <div class="todo-form-row todo-modal-actions">
            <button @click="showClearConfirm = false" class="todo-btn-cancel">取消</button>
            <button @click="clearTodos" class="todo-btn-save">确认清空</button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
.todo-root {
  max-width: 800px;
  margin: 32px auto;
  background: #fafbfc;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  padding: 24px 16px;
}
.todo-panel {
  margin-bottom: 16px;
}
.todo-panel-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}
.todo-label {
  font-size: 15px;
  color: #222;
  font-weight: 500;
}
.todo-divider {
  margin: 0 8px;
  color: #888;
}
.todo-select {
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  padding: 4px 12px;
  font-size: 14px;
  background: #fff;
}
.todo-search {
  width: 100%;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  padding: 6px 12px;
  font-size: 14px;
  background: #fff;
}
.todo-list {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.03);
  margin-bottom: 24px;
}
.todo-list-header, .todo-list-row {
  display: flex;
  align-items: center;
  padding: 0 12px;
}
.todo-list-header {
  height: 40px;
  font-weight: 600;
  color: #888;
  border-bottom: 1px solid #f0f0f0;
  font-size: 15px;
}
.todo-list-row {
  min-height: 48px;
  border-bottom: 1px solid #f0f0f0;
  transition: background 0.2s;
}
.todo-list-row:last-child {
  border-bottom: none;
}
.todo-list-col {
  display: flex;
  align-items: center;
}
.todo-list-col-checkbox {
  width: 40px;
  justify-content: center;
}
.todo-list-col-priority {
  width: 60px;
  justify-content: center;
}
.todo-list-col-title {
  flex: 1;
  min-width: 0;
  font-size: 16px;
  color: #222;
  font-weight: 500;
}
.todo-list-col-time {
  width: 140px;
  font-size: 14px;
  color: #888;
  justify-content: center;
}
.todo-list-col-action {
  width: 110px;
  justify-content: center;
}
.todo-priority {
  display: inline-block;
  padding: 2px 10px;
  border-radius: 12px;
  font-size: 13px;
  font-weight: 600;
  color: #fff;
}
.todo-priority.high {
  background: #e74c3c;
}
.todo-priority.medium {
  background: #f1c40f;
  color: #fff;
}
.todo-priority.low {
  background: #27ae60;
}
.todo-title-completed {
  color: #bbb;
  text-decoration: line-through;
}
.todo-completed {
  opacity: 0.7;
}
.todo-title {
  max-width: 200px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.todo-note {
  display: block;
  font-size: 13px;
  color: #888;
  margin-top: 2px;
  max-width: 180px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.todo-btn-edit {
  background: none;
  border: none;
  color: #1976d2;
  font-size: 15px;
  cursor: pointer;
  padding: 2px 8px;
  border-radius: 6px;
  transition: background 0.15s;
}
.todo-btn-edit:hover {
  background: #f0f7ff;
}
.todo-form {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.03);
  padding: 20px 16px 12px 16px;
  margin-bottom: 8px;
}
.todo-form-title {
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 12px;
  color: #222;
}
.todo-form-row {
  margin-bottom: 10px;
}
.todo-input {
  width: 100%;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  padding: 6px 12px;
  font-size: 15px;
  background: #fff;
}
.todo-btn-add {
  width: 100%;
  background: #1976d2;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 8px 0;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}
.todo-btn-add:hover {
  background: #1565c0;
}
.todo-modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.35);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.todo-modal {
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.10);
  padding: 24px 20px 12px 20px;
  width: 100%;
  max-width: 350px;
}
.todo-modal-title {
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 14px;
  color: #222;
}
.todo-modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}
.todo-btn-cancel {
  background: #f5f5f5;
  color: #888;
  border: none;
  border-radius: 6px;
  padding: 7px 18px;
  font-size: 15px;
  cursor: pointer;
  transition: background 0.15s;
}
.todo-btn-cancel:hover {
  background: #e0e0e0;
}
.todo-btn-save {
  background: #1976d2;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 7px 18px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}
.todo-btn-save:hover {
  background: #1565c0;
}
.list-enter-active,
.list-leave-active {
  transition: all 0.5s cubic-bezier(.55,0,.1,1);
}
.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateY(24px);
}
@media (max-width: 700px) {
  .todo-root {
    padding: 8px 2px;
    max-width: 100vw;
  }
  .todo-list-col-time {
    width: 90px;
    font-size: 12px;
  }
  .todo-list-header, .todo-list-row {
    font-size: 13px;
    padding: 0 4px;
  }
  .todo-modal {
    max-width: 95vw;
    padding: 16px 6px 8px 6px;
  }
}
.todo-btn-delete {
  background: none;
  border: none;
  color: #e74c3c;
  font-size: 15px;
  cursor: pointer;
  padding: 2px 8px;
  border-radius: 6px;
  transition: background 0.15s;
  margin-left: 4px;
}
.todo-btn-delete:hover {
  background: #ffeaea;
}
.todo-btn-clear {
  background: #e74c3c;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 4px 14px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}
.todo-btn-clear:hover {
  background: #c0392b;
}
</style>
