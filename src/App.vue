<script setup>
import { computed, onMounted, ref, watch } from 'vue'

const STORAGE_KEY = 'task-kanban-v1'

const COLUMNS = [
  { id: 'todo', title: '待办', hint: '还没开始的事', accent: '#6366f1' },
  { id: 'doing', title: '进行中', hint: '正在推进的事', accent: '#f59e0b' },
  { id: 'done', title: '已完成', hint: '已经搞定的事', accent: '#10b981' }
]

const tasks = ref([])
const activeTag = ref('all')
const keyword = ref('')
const showForm = ref(false)
const form = ref({ title: '', desc: '', tag: '', status: 'todo' })
const draggingId = ref(null)
const dragOverCol = ref(null)

/* ---------- 持久化 ---------- */
function load() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw) {
      const parsed = JSON.parse(raw)
      if (Array.isArray(parsed)) return parsed
    }
  } catch (e) {
    console.warn('读取本地数据失败', e)
  }
  return null
}

function save() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks.value))
  } catch (e) {
    console.warn('保存本地数据失败', e)
  }
}

onMounted(() => {
  const stored = load()
  if (stored && stored.length) {
    tasks.value = stored
  } else {
    // 首次打开时给一组示例任务，方便直接体验
    tasks.value = [
      {
        id: uid(),
        title: '梳理本周开发计划',
        desc: '把需求拆成可执行的小任务，标注优先级。',
        tag: '计划',
        status: 'todo',
        createdAt: Date.now() - 1000 * 60 * 60 * 20
      },
      {
        id: uid(),
        title: '实现任务卡片拖拽',
        desc: '支持在「待办 / 进行中 / 已完成」之间自由拖动。',
        tag: '开发',
        status: 'doing',
        createdAt: Date.now() - 1000 * 60 * 60 * 8
      },
      {
        id: uid(),
        title: '界面视觉走查',
        desc: '检查配色、间距与移动端适配。',
        tag: '设计',
        status: 'done',
        createdAt: Date.now() - 1000 * 60 * 60 * 3
      }
    ]
  }
})

watch(tasks, save, { deep: true })

/* ---------- 工具 ---------- */
function uid() {
  return Math.random().toString(36).slice(2, 10) + Date.now().toString(36).slice(-4)
}

function tagColor(tag) {
  if (!tag) return '#94a3b8'
  const palette = ['#6366f1', '#ec4899', '#0ea5e9', '#f59e0b', '#10b981', '#8b5cf6', '#ef4444', '#14b8a6']
  let hash = 0
  for (let i = 0; i < tag.length; i++) hash = (hash * 31 + tag.charCodeAt(i)) % 9973
  return palette[hash % palette.length]
}

function formatDate(ts) {
  const d = new Date(ts)
  const pad = (n) => String(n).padStart(2, '0')
  return `${d.getMonth() + 1}月${d.getDate()}日 ${pad(d.getHours())}:${pad(d.getMinutes())}`
}

/* ---------- 派生数据 ---------- */
const allTags = computed(() => {
  const map = new Map()
  tasks.value.forEach((t) => {
    const tag = (t.tag || '').trim()
    if (!tag) return
    map.set(tag, (map.get(tag) || 0) + 1)
  })
  return [...map.entries()].map(([name, count]) => ({ name, count }))
})

const filtered = computed(() => {
  const kw = keyword.value.trim().toLowerCase()
  return tasks.value.filter((t) => {
    const tagOk = activeTag.value === 'all' || (t.tag || '') === activeTag.value
    const kwOk =
      !kw ||
      (t.title || '').toLowerCase().includes(kw) ||
      (t.desc || '').toLowerCase().includes(kw)
    return tagOk && kwOk
  })
})

function columnTasks(colId) {
  return filtered.value
    .filter((t) => t.status === colId)
    .sort((a, b) => b.createdAt - a.createdAt)
}

const stats = computed(() => {
  const total = tasks.value.length
  const done = tasks.value.filter((t) => t.status === 'done').length
  return {
    total,
    done,
    doing: tasks.value.filter((t) => t.status === 'doing').length,
    percent: total ? Math.round((done / total) * 100) : 0
  }
})

/* ---------- 交互 ---------- */
function openForm(status = 'todo') {
  form.value = { title: '', desc: '', tag: activeTag.value === 'all' ? '' : activeTag.value, status }
  showForm.value = true
}

function submitForm() {
  const title = form.value.title.trim()
  if (!title) return
  tasks.value.push({
    id: uid(),
    title,
    desc: form.value.desc.trim(),
    tag: form.value.tag.trim(),
    status: form.value.status,
    createdAt: Date.now()
  })
  showForm.value = false
}

function removeTask(id) {
  tasks.value = tasks.value.filter((t) => t.id !== id)
}

function clearDone() {
  tasks.value = tasks.value.filter((t) => t.status !== 'done')
}

function onDragStart(e, task) {
  draggingId.value = task.id
  e.dataTransfer.effectAllowed = 'move'
  try {
    e.dataTransfer.setData('text/plain', task.id)
  } catch (err) {
    /* ignore */
  }
}

function onDragEnd() {
  draggingId.value = null
  dragOverCol.value = null
}

function onDragOver(e, colId) {
  e.preventDefault()
  e.dataTransfer.dropEffect = 'move'
  dragOverCol.value = colId
}

function onDrop(e, colId) {
  e.preventDefault()
  const id = draggingId.value || e.dataTransfer.getData('text/plain')
  const task = tasks.value.find((t) => t.id === id)
  if (task && task.status !== colId) {
    task.status = colId
    save()
  }
  onDragEnd()
}
</script>

<template>
  <div class="page">
    <header class="hero">
      <div class="hero-text">
        <p class="eyebrow">Vue3 · Vite · localStorage</p>
        <h1>任务管理看板</h1>
        <p class="sub">新建任务卡片、拖拽切换状态、按分类筛选，数据自动保存在浏览器本地。</p>
      </div>

      <div class="hero-side">
        <div class="progress-card">
          <div class="progress-top">
            <span>整体完成度</span>
            <strong>{{ stats.percent }}%</strong>
          </div>
          <div class="bar"><i :style="{ width: stats.percent + '%' }"></i></div>
          <div class="progress-meta">
            <span>共 {{ stats.total }} 个任务</span>
            <span>进行中 {{ stats.doing }}</span>
            <span>已完成 {{ stats.done }}</span>
          </div>
        </div>
        <button class="primary" @click="openForm()">＋ 新建任务</button>
      </div>
    </header>

    <section class="toolbar">
      <div class="chips">
        <button
          class="chip"
          :class="{ active: activeTag === 'all' }"
          @click="activeTag = 'all'"
        >
          全部 <em>{{ tasks.length }}</em>
        </button>
        <button
          v-for="tag in allTags"
          :key="tag.name"
          class="chip"
          :class="{ active: activeTag === tag.name }"
          @click="activeTag = tag.name"
        >
          <i class="dot" :style="{ background: tagColor(tag.name) }"></i>
          {{ tag.name }} <em>{{ tag.count }}</em>
        </button>
      </div>

      <div class="toolbar-right">
        <input v-model="keyword" class="search" type="search" placeholder="搜索标题或描述…" />
        <button class="ghost" @click="clearDone">清空已完成</button>
      </div>
    </section>

    <main class="board">
      <section
        v-for="col in COLUMNS"
        :key="col.id"
        class="column"
        :class="{ 'drag-over': dragOverCol === col.id }"
        @dragover="onDragOver($event, col.id)"
        @dragleave="dragOverCol = null"
        @drop="onDrop($event, col.id)"
      >
        <div class="column-head">
          <div class="column-title">
            <i class="bar-accent" :style="{ background: col.accent }"></i>
            <h2>{{ col.title }}</h2>
            <span class="count">{{ columnTasks(col.id).length }}</span>
          </div>
          <button class="mini" @click="openForm(col.id)">＋</button>
        </div>
        <p class="column-hint">{{ col.hint }}</p>

        <div class="cards">
          <article
            v-for="task in columnTasks(col.id)"
            :key="task.id"
            class="card"
            :class="{ dragging: draggingId === task.id }"
            draggable="true"
            @dragstart="onDragStart($event, task)"
            @dragend="onDragEnd"
          >
            <div class="card-head">
              <span v-if="task.tag" class="tag" :style="{ background: tagColor(task.tag) + '1a', color: tagColor(task.tag) }">
                {{ task.tag }}
              </span>
              <span v-else class="tag tag-empty">未分类</span>
              <button class="del" title="删除任务" @click.stop="removeTask(task.id)">✕</button>
            </div>
            <h3>{{ task.title }}</h3>
            <p v-if="task.desc" class="desc">{{ task.desc }}</p>
            <footer>
              <span>{{ formatDate(task.createdAt) }}</span>
              <span class="grip">⠿ 拖动</span>
            </footer>
          </article>

          <p v-if="columnTasks(col.id).length === 0" class="empty">把卡片拖到这里，或点右上角 ＋ 新建</p>
        </div>
      </section>
    </main>

    <div v-if="showForm" class="overlay" @click.self="showForm = false">
      <div class="modal">
        <h2>新建任务</h2>
        <label>
          <span>标题</span>
          <input v-model="form.title" type="text" placeholder="例如：完成首页视觉走查" maxlength="60" />
        </label>
        <label>
          <span>描述</span>
          <textarea v-model="form.desc" rows="3" placeholder="补充说明（可选）" maxlength="200"></textarea>
        </label>
        <div class="row">
          <label>
            <span>分类标签</span>
            <input v-model="form.tag" type="text" placeholder="例如：设计 / 开发" maxlength="12" />
          </label>
          <label>
            <span>所在列</span>
            <select v-model="form.status">
              <option v-for="col in COLUMNS" :key="col.id" :value="col.id">{{ col.title }}</option>
            </select>
          </label>
        </div>
        <div class="modal-actions">
          <button class="ghost" @click="showForm = false">取消</button>
          <button class="primary" :disabled="!form.title.trim()" @click="submitForm">创建任务</button>
        </div>
      </div>
    </div>

    <footer class="page-foot">数据保存在浏览器 localStorage 中，刷新页面不会丢失 · 支持拖拽排序与分类筛选</footer>
  </div>
</template>

<style scoped>
.page {
  max-width: 1240px;
  margin: 0 auto;
  padding: 32px 24px 48px;
}

.hero {
  display: flex;
  gap: 28px;
  align-items: stretch;
  justify-content: space-between;
  flex-wrap: wrap;
  margin-bottom: 22px;
}

.eyebrow {
  margin: 0 0 8px;
  font-size: 12px;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: #8b93a7;
}

.hero h1 {
  margin: 0 0 8px;
  font-size: 32px;
  letter-spacing: -0.02em;
}

.sub {
  margin: 0;
  max-width: 520px;
  color: var(--muted);
  line-height: 1.6;
  font-size: 14px;
}

.hero-side {
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-width: 300px;
}

.progress-card {
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 16px 18px;
  box-shadow: var(--shadow-sm);
}

.progress-top {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  color: var(--muted);
  margin-bottom: 10px;
}

.progress-top strong {
  color: var(--text);
  font-size: 16px;
}

.bar {
  height: 8px;
  border-radius: 99px;
  background: #eef1f8;
  overflow: hidden;
}

.bar i {
  display: block;
  height: 100%;
  border-radius: 99px;
  background: linear-gradient(90deg, #6366f1, #8b5cf6 55%, #ec4899);
  transition: width 0.45s ease;
}

.progress-meta {
  display: flex;
  gap: 14px;
  margin-top: 10px;
  font-size: 12px;
  color: var(--muted);
}

.primary {
  border: none;
  border-radius: 12px;
  padding: 12px 18px;
  font-size: 14px;
  font-weight: 600;
  color: #fff;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  box-shadow: 0 10px 22px rgba(99, 102, 241, 0.32);
  transition: transform 0.16s ease, box-shadow 0.16s ease, opacity 0.16s ease;
}

.primary:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 14px 26px rgba(99, 102, 241, 0.38);
}

.primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  box-shadow: none;
}

.toolbar {
  display: flex;
  gap: 14px;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  margin-bottom: 18px;
}

.chips {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  border: 1px solid var(--border);
  background: rgba(255, 255, 255, 0.75);
  color: var(--muted);
  border-radius: 99px;
  padding: 7px 13px;
  font-size: 13px;
  transition: all 0.16s ease;
}

.chip em {
  font-style: normal;
  font-size: 11px;
  background: #eef1f8;
  border-radius: 99px;
  padding: 1px 7px;
}

.chip:hover {
  color: var(--text);
  border-color: #c9d0e3;
}

.chip.active {
  color: #fff;
  border-color: transparent;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  box-shadow: 0 8px 18px rgba(99, 102, 241, 0.28);
}

.chip.active em {
  background: rgba(255, 255, 255, 0.24);
  color: #fff;
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  display: inline-block;
}

.toolbar-right {
  display: flex;
  gap: 10px;
  align-items: center;
}

.search {
  width: 230px;
  border: 1px solid var(--border);
  background: #fff;
  border-radius: 11px;
  padding: 9px 12px;
  outline: none;
  transition: border-color 0.16s ease, box-shadow 0.16s ease;
}

.search:focus {
  border-color: #a5b4fc;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.14);
}

.ghost {
  border: 1px solid var(--border);
  background: #fff;
  color: var(--muted);
  border-radius: 11px;
  padding: 9px 14px;
  font-size: 13px;
  transition: all 0.16s ease;
}

.ghost:hover {
  color: #ef4444;
  border-color: #fecaca;
  background: #fff7f7;
}

.board {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 18px;
  align-items: start;
}

.column {
  background: rgba(255, 255, 255, 0.66);
  border: 1px solid var(--border);
  border-radius: 18px;
  padding: 16px 14px 10px;
  backdrop-filter: blur(6px);
  transition: background 0.18s ease, border-color 0.18s ease, transform 0.18s ease;
}

.column.drag-over {
  background: rgba(238, 242, 255, 0.95);
  border-color: #a5b4fc;
  box-shadow: inset 0 0 0 2px rgba(99, 102, 241, 0.18);
}

.column-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.column-title {
  display: flex;
  align-items: center;
  gap: 8px;
}

.bar-accent {
  width: 4px;
  height: 16px;
  border-radius: 99px;
}

.column-title h2 {
  margin: 0;
  font-size: 15px;
  font-weight: 600;
}

.count {
  font-size: 11px;
  color: var(--muted);
  background: #eff2f9;
  border-radius: 99px;
  padding: 1px 8px;
}

.mini {
  border: 1px solid var(--border);
  background: #fff;
  color: var(--muted);
  width: 26px;
  height: 26px;
  border-radius: 9px;
  line-height: 1;
  transition: all 0.16s ease;
}

.mini:hover {
  color: #6366f1;
  border-color: #c7d2fe;
  background: #f5f7ff;
}

.column-hint {
  margin: 6px 0 12px 12px;
  font-size: 12px;
  color: #9aa3b8;
}

.cards {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-height: 60px;
  padding-bottom: 6px;
}

.card {
  background: #fff;
  border: 1px solid var(--border);
  border-radius: 13px;
  padding: 12px 13px 10px;
  box-shadow: var(--shadow-sm);
  cursor: grab;
  animation: pop 0.22s ease both;
  transition: transform 0.16s ease, box-shadow 0.16s ease, border-color 0.16s ease;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
  border-color: #dfe4f1;
}

.card.dragging {
  opacity: 0.45;
  transform: rotate(1.2deg) scale(0.99);
}

@keyframes pop {
  from {
    opacity: 0;
    transform: translateY(6px) scale(0.985);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

.card-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.tag {
  font-size: 11px;
  padding: 2px 9px;
  border-radius: 99px;
  font-weight: 600;
}

.tag-empty {
  background: #f1f4fa;
  color: #9aa3b8;
}

.del {
  border: none;
  background: transparent;
  color: #c3cad9;
  font-size: 12px;
  padding: 2px 4px;
  border-radius: 6px;
  transition: all 0.16s ease;
}

.del:hover {
  color: #ef4444;
  background: #fff1f1;
}

.card h3 {
  margin: 9px 0 4px;
  font-size: 14px;
  line-height: 1.45;
  font-weight: 600;
}

.desc {
  margin: 0 0 8px;
  font-size: 12.5px;
  line-height: 1.55;
  color: var(--muted);
  word-break: break-word;
}

.card footer {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #a9b1c4;
  border-top: 1px dashed #eef1f8;
  padding-top: 7px;
}

.grip {
  opacity: 0;
  transition: opacity 0.16s ease;
}

.card:hover .grip {
  opacity: 1;
}

.empty {
  margin: 6px 0 10px;
  padding: 18px 10px;
  text-align: center;
  font-size: 12px;
  color: #a9b1c4;
  border: 1px dashed #dfe4f1;
  border-radius: 12px;
}

.overlay {
  position: fixed;
  inset: 0;
  background: rgba(17, 24, 39, 0.42);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 30;
  animation: fade 0.18s ease both;
}

@keyframes fade {
  from {
    opacity: 0;
  }
}

.modal {
  width: 100%;
  max-width: 470px;
  background: #fff;
  border-radius: 18px;
  padding: 22px;
  box-shadow: var(--shadow-lg);
  animation: rise 0.2s ease both;
}

@keyframes rise {
  from {
    opacity: 0;
    transform: translateY(10px) scale(0.985);
  }
}

.modal h2 {
  margin: 0 0 16px;
  font-size: 18px;
}

.modal label {
  display: block;
  margin-bottom: 13px;
}

.modal label span {
  display: block;
  font-size: 12px;
  color: var(--muted);
  margin-bottom: 6px;
}

.modal input,
.modal textarea,
.modal select {
  width: 100%;
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 10px 12px;
  outline: none;
  resize: vertical;
  transition: border-color 0.16s ease, box-shadow 0.16s ease;
}

.modal input:focus,
.modal textarea:focus,
.modal select:focus {
  border-color: #a5b4fc;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.14);
}

.row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 18px;
}

.page-foot {
  margin-top: 26px;
  text-align: center;
  font-size: 12px;
  color: #a9b1c4;
}

@media (max-width: 900px) {
  .board {
    grid-template-columns: 1fr;
  }
  .search {
    width: 100%;
  }
  .toolbar-right {
    width: 100%;
  }
  .hero h1 {
    font-size: 26px;
  }
}
</style>
