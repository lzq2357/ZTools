<template>
  <div class="content-panel">
    <!-- 可滚动内容区 -->
    <Transition name="list-slide">
      <div v-show="!showEditor" class="scrollable-content">
        <!-- Tab 切换 -->
        <div class="tabs-container">
          <div class="tab-group">
            <button
              :class="['tab-btn', { active: activeTab === 'global' }]"
              @click="activeTab = 'global'"
            >
              全局快捷键
            </button>
            <button
              :class="['tab-btn', { active: activeTab === 'app' }]"
              @click="activeTab = 'app'"
            >
              应用快捷键
            </button>
          </div>
        </div>

        <!-- 顶部添加按钮 -->
        <div class="panel-header">
          <button class="btn" @click="showAddEditor">添加快捷键</button>
        </div>

        <!-- 快捷键列表 -->
        <div class="shortcut-list">
          <div v-for="shortcut in filteredShortcuts" :key="shortcut.id" class="card shortcut-item">
            <div class="shortcut-info">
              <div class="shortcut-key-display">{{ shortcut.shortcut }}</div>
              <div class="shortcut-desc">{{ shortcut.target }}</div>
            </div>

            <div class="shortcut-meta">
              <button
                class="icon-btn edit-btn"
                title="编辑"
                :disabled="isDeleting"
                @click="handleEdit(shortcut)"
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  width="14"
                  height="14"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                >
                  <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"></path>
                  <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"></path>
                </svg>
              </button>
              <button
                class="icon-btn delete-btn"
                title="删除"
                :disabled="isDeleting"
                @click="handleDelete(shortcut.id)"
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  width="14"
                  height="14"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                >
                  <polyline points="3 6 5 6 21 6"></polyline>
                  <path
                    d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"
                  ></path>
                  <line x1="10" y1="11" x2="10" y2="17"></line>
                  <line x1="14" y1="11" x2="14" y2="17"></line>
                </svg>
              </button>
            </div>
          </div>

          <!-- 空状态 -->
          <div v-if="!loading && currentShortcuts.length === 0" class="empty-state">
            <Icon name="keyboard" :size="64" class="empty-icon" />
            <div class="empty-text">暂无{{ activeTab === 'global' ? '全局' : '应用' }}快捷键</div>
            <div class="empty-hint">
              点击"添加快捷键"来创建你的第一个{{ activeTab === 'global' ? '全局' : '应用' }}快捷键
            </div>
          </div>
        </div>
      </div>
    </Transition>

    <!-- 快捷键编辑器覆盖面板组件 -->
    <Transition name="slide">
      <ShortcutEditor
        v-if="showEditor"
        :editing-shortcut="editingShortcut"
        :prefill-target="prefillTarget"
        :is-app-shortcut="activeTab === 'app'"
        @back="closeEditor"
        @save="handleSave"
      />
    </Transition>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { useToast } from '../../composables/useToast'
import { weightedSearch } from '../../utils/weightedSearch'
import Icon from '../common/Icon.vue'
import ShortcutEditor from './ShortcutEditor.vue'

const props = defineProps<{
  searchQuery?: string
  autoAddTarget?: string
}>()

const emit = defineEmits<{
  (e: 'autoAddConsumed'): void
}>()

const { success, error, warning, confirm } = useToast()

interface GlobalShortcut {
  id: string
  shortcut: string
  target: string
  enabled: boolean
}

// Tab 切换
const activeTab = ref<'global' | 'app'>('global')

// 全局快捷键列表
const globalShortcuts = ref<GlobalShortcut[]>([])
// 应用快捷键列表
const appShortcuts = ref<GlobalShortcut[]>([])
const isDeleting = ref(false)
const loading = ref(true)

// 当前显示的快捷键列表
const currentShortcuts = computed(() => {
  return activeTab.value === 'global' ? globalShortcuts.value : appShortcuts.value
})

const filteredShortcuts = computed(() =>
  weightedSearch(currentShortcuts.value, props.searchQuery || '', [
    { value: (s) => s.shortcut || '', weight: 10 },
    { value: (s) => s.target || '', weight: 5 }
  ])
)

// 编辑器状态
const showEditor = ref(false)
const editingShortcut = ref<GlobalShortcut | null>(null)
const prefillTarget = ref('')

// 监听 autoAddTarget prop
watch(
  () => props.autoAddTarget,
  (target) => {
    if (target) {
      prefillTarget.value = target
      editingShortcut.value = null
      showEditor.value = true
      emit('autoAddConsumed')
    }
  },
  { immediate: true }
)

// 加载全局快捷键列表
async function loadGlobalShortcuts(): Promise<void> {
  try {
    const data = await window.ztools.internal.dbGet('global-shortcuts')
    globalShortcuts.value = data || []
    console.log('加载全局快捷键:', globalShortcuts.value)
  } catch (error) {
    console.error('加载全局快捷键失败:', error)
  }
}

// 加载应用快捷键列表
async function loadAppShortcuts(): Promise<void> {
  try {
    const data = await window.ztools.internal.dbGet('app-shortcuts')
    appShortcuts.value = data || []
    console.log('加载应用快捷键:', appShortcuts.value)
  } catch (error) {
    console.error('加载应用快捷键失败:', error)
  }
}

// 加载所有快捷键
async function loadShortcuts(): Promise<void> {
  loading.value = true
  try {
    await Promise.all([loadGlobalShortcuts(), loadAppShortcuts()])
  } finally {
    loading.value = false
  }
}

// 保存全局快捷键列表
async function saveGlobalShortcuts(): Promise<void> {
  try {
    await window.ztools.internal.dbPut(
      'global-shortcuts',
      JSON.parse(JSON.stringify(globalShortcuts.value))
    )
    console.log('保存全局快捷键成功')
  } catch (error) {
    console.error('保存全局快捷键失败:', error)
  }
}

// 保存应用快捷键列表
async function saveAppShortcuts(): Promise<void> {
  try {
    await window.ztools.internal.dbPut(
      'app-shortcuts',
      JSON.parse(JSON.stringify(appShortcuts.value))
    )
    console.log('保存应用快捷键成功')
  } catch (error) {
    console.error('保存应用快捷键失败:', error)
  }
}

// 显示添加编辑器
function showAddEditor(): void {
  editingShortcut.value = null
  prefillTarget.value = ''
  showEditor.value = true
}

// 显示编辑编辑器
function handleEdit(shortcut: GlobalShortcut): void {
  editingShortcut.value = shortcut
  prefillTarget.value = ''
  showEditor.value = true
}

// 关闭编辑器
function closeEditor(): void {
  showEditor.value = false
  editingShortcut.value = null
  prefillTarget.value = ''
}

// 保存快捷键（添加或编辑）
async function handleSave(recordedShortcut: string, targetCommand: string): Promise<void> {
  if (!recordedShortcut || !targetCommand) {
    return
  }

  // 根据当前 tab 选择对应的逻辑
  if (activeTab.value === 'global') {
    await handleSaveGlobalShortcut(recordedShortcut, targetCommand)
  } else {
    await handleSaveAppShortcut(recordedShortcut, targetCommand)
  }
}

// 保存全局快捷键（添加或编辑）
async function handleSaveGlobalShortcut(
  recordedShortcut: string,
  targetCommand: string
): Promise<void> {
  // 如果是编辑模式
  if (editingShortcut.value) {
    const exists = globalShortcuts.value.some(
      (s) => s.id !== editingShortcut.value!.id && s.shortcut === recordedShortcut
    )
    if (exists) {
      warning('该快捷键已被其他指令占用，请使用其他快捷键')
      return
    }

    const oldShortcut = editingShortcut.value.shortcut

    try {
      if (oldShortcut !== recordedShortcut) {
        await window.ztools.internal.unregisterGlobalShortcut(oldShortcut)
      }

      const result = await window.ztools.internal.registerGlobalShortcut(
        recordedShortcut,
        targetCommand
      )

      if (result.success) {
        const index = globalShortcuts.value.findIndex((s) => s.id === editingShortcut.value!.id)
        if (index >= 0) {
          globalShortcuts.value[index].shortcut = recordedShortcut
          globalShortcuts.value[index].target = targetCommand
        }

        await saveGlobalShortcuts()
        success('快捷键更新成功!')
        closeEditor()
      } else {
        if (oldShortcut !== recordedShortcut) {
          await window.ztools.internal.registerGlobalShortcut(
            oldShortcut,
            editingShortcut.value.target
          )
        }
        error(`快捷键注册失败: ${result.error}`)
      }
    } catch (err: any) {
      if (oldShortcut !== recordedShortcut) {
        await window.ztools.internal.registerGlobalShortcut(
          oldShortcut,
          editingShortcut.value.target
        )
      }
      console.error('更新快捷键失败:', err)
      error(`更新快捷键失败: ${err.message || '未知错误'}`)
    }
    return
  }

  // 添加模式：检查快捷键是否已存在
  const exists = globalShortcuts.value.some((s) => s.shortcut === recordedShortcut)
  if (exists) {
    warning('该快捷键已存在，请使用其他快捷键')
    return
  }

  const newShortcut: GlobalShortcut = {
    id: Date.now().toString(),
    shortcut: recordedShortcut,
    target: targetCommand,
    enabled: true
  }

  globalShortcuts.value.push(newShortcut)
  await saveGlobalShortcuts()

  try {
    const result = await window.ztools.internal.registerGlobalShortcut(
      recordedShortcut,
      targetCommand
    )
    if (result.success) {
      success('快捷键添加成功!')
      closeEditor()
    } else {
      globalShortcuts.value = globalShortcuts.value.filter((s) => s.id !== newShortcut.id)
      await saveGlobalShortcuts()
      error(`快捷键注册失败: ${result.error}`)
    }
  } catch (err: any) {
    globalShortcuts.value = globalShortcuts.value.filter((s) => s.id !== newShortcut.id)
    await saveGlobalShortcuts()
    console.error('注册快捷键失败:', err)
    error(`注册快捷键失败: ${err.message || '未知错误'}`)
  }
}

// 保存应用快捷键（添加或编辑）
async function handleSaveAppShortcut(
  recordedShortcut: string,
  targetCommand: string
): Promise<void> {
  // 如果是编辑模式
  if (editingShortcut.value) {
    const exists = appShortcuts.value.some(
      (s) => s.id !== editingShortcut.value!.id && s.shortcut === recordedShortcut
    )
    if (exists) {
      warning('该快捷键已被其他指令占用，请使用其他快捷键')
      return
    }

    const oldShortcut = editingShortcut.value.shortcut

    try {
      if (oldShortcut !== recordedShortcut) {
        await window.ztools.internal.unregisterAppShortcut(oldShortcut)
      }

      const result = await window.ztools.internal.registerAppShortcut(
        recordedShortcut,
        targetCommand
      )

      if (result.success) {
        const index = appShortcuts.value.findIndex((s) => s.id === editingShortcut.value!.id)
        if (index >= 0) {
          appShortcuts.value[index].shortcut = recordedShortcut
          appShortcuts.value[index].target = targetCommand
        }

        await saveAppShortcuts()
        success('应用快捷键更新成功!')
        closeEditor()
      } else {
        if (oldShortcut !== recordedShortcut) {
          await window.ztools.internal.registerAppShortcut(oldShortcut, editingShortcut.value.target)
        }
        error(`应用快捷键注册失败: ${result.error}`)
      }
    } catch (err: any) {
      if (oldShortcut !== recordedShortcut) {
        await window.ztools.internal.registerAppShortcut(oldShortcut, editingShortcut.value.target)
      }
      console.error('更新应用快捷键失败:', err)
      error(`更新应用快捷键失败: ${err.message || '未知错误'}`)
    }
    return
  }

  // 添加模式：检查快捷键是否已存在
  const exists = appShortcuts.value.some((s) => s.shortcut === recordedShortcut)
  if (exists) {
    warning('该快捷键已存在，请使用其他快捷键')
    return
  }

  const newShortcut: GlobalShortcut = {
    id: Date.now().toString(),
    shortcut: recordedShortcut,
    target: targetCommand,
    enabled: true
  }

  appShortcuts.value.push(newShortcut)
  await saveAppShortcuts()

  try {
    const result = await window.ztools.internal.registerAppShortcut(recordedShortcut, targetCommand)
    if (result.success) {
      success('应用快捷键添加成功!')
      closeEditor()
    } else {
      appShortcuts.value = appShortcuts.value.filter((s) => s.id !== newShortcut.id)
      await saveAppShortcuts()
      error(`应用快捷键注册失败: ${result.error}`)
    }
  } catch (err: any) {
    appShortcuts.value = appShortcuts.value.filter((s) => s.id !== newShortcut.id)
    await saveAppShortcuts()
    console.error('注册应用快捷键失败:', err)
    error(`注册应用快捷键失败: ${err.message || '未知错误'}`)
  }
}

// 删除快捷键
async function handleDelete(id: string): Promise<void> {
  const isGlobal = activeTab.value === 'global'
  const shortcutList = isGlobal ? globalShortcuts.value : appShortcuts.value
  const shortcut = shortcutList.find((s) => s.id === id)
  if (!shortcut) return

  const confirmed = await confirm({
    title: '删除快捷键',
    message: `确定要删除快捷键"${shortcut.shortcut}"吗？`,
    type: 'danger',
    confirmText: '删除',
    cancelText: '取消'
  })
  if (!confirmed) return

  isDeleting.value = true
  try {
    if (isGlobal) {
      const result = await window.ztools.internal.unregisterGlobalShortcut(shortcut.shortcut)
      if (result.success) {
        globalShortcuts.value = globalShortcuts.value.filter((s) => s.id !== id)
        await saveGlobalShortcuts()
      } else {
        error(`快捷键删除失败: ${result.error}`)
      }
    } else {
      const result = await window.ztools.internal.unregisterAppShortcut(shortcut.shortcut)
      if (result.success) {
        appShortcuts.value = appShortcuts.value.filter((s) => s.id !== id)
        await saveAppShortcuts()
      } else {
        error(`应用快捷键删除失败: ${result.error}`)
      }
    }
  } catch (err: any) {
    console.error('删除快捷键失败:', err)
    error(`删除快捷键失败: ${err.message || '未知错误'}`)
  } finally {
    isDeleting.value = false
  }
}

// 初始化
onMounted(() => {
  loadShortcuts()
})
</script>

<style scoped>
.content-panel {
  position: relative;
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg-color);
}

/* 可滚动内容区 */
.scrollable-content {
  position: absolute;
  inset: 0;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 20px;
}

/* Tab 切换 */
.tabs-container {
  position: sticky;
  top: 0;
  background: var(--bg-color);
  z-index: 10;
  padding-bottom: 16px;
}

.tab-group {
  display: flex;
  gap: 6px;
  background: var(--control-bg);
  padding: 3px;
  border-radius: 8px;
}

.tab-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  flex: 1;
  justify-content: center;
  padding: 6px 14px;
  font-size: 13px;
  border: none;
  background: transparent;
  color: var(--text-secondary);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  font-weight: 500;
}

.tab-btn:hover {
  background: var(--hover-bg);
  color: var(--text-color);
}

.tab-btn.active {
  background: var(--active-bg);
  color: var(--primary-color);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

/* 列表滑动动画 */
.list-slide-enter-active {
  transition:
    transform 0.2s ease-out,
    opacity 0.15s ease;
}

.list-slide-leave-active {
  transition:
    transform 0.2s ease-in,
    opacity 0.15s ease;
}

.list-slide-enter-from {
  transform: translateX(-100%);
  opacity: 0;
}

.list-slide-enter-to {
  transform: translateX(0);
  opacity: 1;
}

.list-slide-leave-from {
  transform: translateX(0);
  opacity: 1;
}

.list-slide-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}

/* 顶部按钮 */
.panel-header {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 20px;
}

/* 快捷键列表 */
.shortcut-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.shortcut-item {
  display: flex;
  align-items: center;
  padding: 12px 14px;
  cursor: pointer;
  transition: all 0.2s;
}

.shortcut-item:hover {
  background: var(--hover-bg);
  transform: translateX(2px);
}

.shortcut-info {
  flex: 1;
  min-width: 0;
}

.shortcut-key-display {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 4px;
  font-family: monospace;
}

.shortcut-desc {
  font-size: 13px;
  color: var(--text-secondary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.shortcut-meta {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* 图标按钮颜色样式 */
.edit-btn {
  color: var(--primary-color);
}

.edit-btn:hover:not(:disabled) {
  background: var(--primary-light-bg);
}

.delete-btn {
  color: var(--danger-color);
}

.delete-btn:hover:not(:disabled) {
  background: var(--danger-light-bg);
}

/* 空状态 */
.empty-state {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px;
  text-align: center;
  pointer-events: none;
}

.empty-icon {
  margin-bottom: 16px;
  opacity: 0.3;
  color: var(--text-secondary);
}

.empty-text {
  font-size: 18px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 8px;
}

.empty-hint {
  font-size: 14px;
  color: var(--text-secondary);
}
</style>
