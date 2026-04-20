<template>
  <div class="workbench">
    <StatsTopBar :slug="slug" @open-settings="showLLMSettings = true" />

    <n-spin :show="pageLoading" class="workbench-spin" description="加载工作台…">
      <div class="workbench-inner">

        <!-- 左侧面板 -->
        <div
          class="side-pane left-pane"
          :style="{ width: leftCollapsed ? '0' : leftWidth + 'px' }"
        >
          <ChapterList
            ref="chapterListRef"
            :slug="slug"
            :chapters="chapters"
            :current-chapter-id="currentChapterId"
            @select="onSidebarChapterSelect"
            @back="goHome"
            @refresh="handleChapterUpdated"
            @plan-act="handlePlanAct"
          />
        </div>

        <!-- 左侧分割线 + 折叠按钮 -->
        <div
          class="divider left-divider"
          @mousedown="startDragLeft"
        >
          <button
            class="collapse-btn"
            @mousedown.stop
            @click="leftCollapsed = !leftCollapsed"
            :title="leftCollapsed ? '展开左栏' : '收起左栏'"
          >{{ leftCollapsed ? '›' : '‹' }}</button>
        </div>

        <!-- 中间主区域 -->
        <div class="main-pane">
          <WorkArea
            ref="workAreaRef"
            :slug="slug"
            :book-title="bookTitle"
            :chapters="chapters"
            :target-words-per-chapter="targetWordsPerChapter"
            :current-chapter-id="currentChapterId"
            :chapter-content="chapterContent"
            :chapter-loading="chapterLoading"
            @set-right-panel="setRightPanel"
            @chapter-updated="handleChapterUpdated"
          />
        </div>

        <!-- 右侧分割线 + 折叠按钮 -->
        <div
          class="divider right-divider"
          @mousedown="startDragRight"
        >
          <button
            class="collapse-btn"
            @mousedown.stop
            @click="rightCollapsed = !rightCollapsed"
            :title="rightCollapsed ? '展开右栏' : '收起右栏'"
          >{{ rightCollapsed ? '‹' : '›' }}</button>
        </div>

        <!-- 右侧面板 -->
        <div
          class="side-pane right-pane"
          :style="{ width: rightCollapsed ? '0' : rightWidth + 'px' }"
        >
          <SettingsPanel
            :slug="slug"
            :current-panel="rightPanel"
            :bible-key="biblePanelKey"
            :current-chapter="currentChapter"
            @update:current-panel="onSettingsPanelChange"
          />
        </div>

      </div>
    </n-spin>

    <!-- 幕→章 AI 规划弹层 -->
    <ActPlanningModal
      v-model:show="showActPlanning"
      :act-id="actPlanningId"
      :act-title="actPlanningTitle"
      @confirmed="handleChapterUpdated"
    />

    <!-- LLM Settings Modal -->
    <LLMSettingsModal v-model:show="showLLMSettings" />

    <!-- 全局浮动按钮 -->
    <GlobalLLMFloatingButton />
    <PromptPlazaFAB />
  </div>
</template>

<script setup lang="ts">
import { defineAsyncComponent, onMounted, onUnmounted, computed, ref, watch, type ComponentPublicInstance } from 'vue'
import { useRoute } from 'vue-router'
import { useMessage } from 'naive-ui'
import { useWorkbench } from '../composables/useWorkbench'
import { useStatsStore } from '../stores/statsStore'
import { useWorkbenchRefreshStore } from '../stores/workbenchRefreshStore'

const StatsTopBar = defineAsyncComponent(() => import('../components/stats/StatsTopBar.vue'))
const ChapterList = defineAsyncComponent(() => import('../components/workbench/ChapterList.vue'))
const WorkArea = defineAsyncComponent(() => import('../components/workbench/WorkArea.vue'))
const SettingsPanel = defineAsyncComponent(() => import('../components/workbench/SettingsPanel.vue'))
const ActPlanningModal = defineAsyncComponent(() => import('../components/workbench/ActPlanningModal.vue'))
const LLMSettingsModal = defineAsyncComponent(() => import('../components/LLMSettingsModal.vue'))
const GlobalLLMFloatingButton = defineAsyncComponent(() => import('../components/global/GlobalLLMFloatingButton.vue'))
const PromptPlazaFAB = defineAsyncComponent(() => import('../components/global/PromptPlazaFAB.vue'))

const route = useRoute()
const message = useMessage()
const statsStore = useStatsStore()
const workbenchRefresh = useWorkbenchRefreshStore()

const slug = route.params.slug as string

const chapterListRef = ref<ComponentPublicInstance<{ refreshStoryTree: () => void }> | null>(null)
const workAreaRef = ref<ComponentPublicInstance<{ ensureAssistedMode: () => void }> | null>(null)

// ━━━ 侧栏宽度 & 折叠（基于 1920px 视口的相对比例）━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 左侧面板：最小 12.4% (238px/1920)，最大 31.3% (600px/1920)，默认 20.2% (388px/1920)
const LEFT_MIN_PCT = 12.4
const LEFT_MAX_PCT = 31.3
const LEFT_DEFAULT_PCT = 20.2

// 右侧面板：最小 13.5% (260px/1920)，最大 27.1% (520px/1920)，默认 26.7% (512px/1920)
const RIGHT_MIN_PCT = 13.5
const RIGHT_MAX_PCT = 27.1
const RIGHT_DEFAULT_PCT = 26.7

// 根据当前视口宽度计算像素值
function vwToPx(vw: number): number {
  return Math.round((vw / 100) * window.innerWidth)
}

const leftWidth = ref(vwToPx(LEFT_DEFAULT_PCT))
const rightWidth = ref(vwToPx(RIGHT_DEFAULT_PCT))
const leftCollapsed = ref(false)
const rightCollapsed = ref(false)

// 拖拽左侧分割线
let dragging: 'left' | 'right' | null = null
let dragStartX = 0
let dragStartWidth = 0

function startDragLeft(e: MouseEvent) {
  dragging = 'left'
  dragStartX = e.clientX
  dragStartWidth = leftWidth.value
  // 缓存边界值
  updateCacheBoundaries()
  // 拖拽时禁止文本选中
  document.body.style.userSelect = 'none'
  // 添加拖拽类禁用过渡动画
  document.querySelector('.left-pane')?.classList.add('dragging')
  document.addEventListener('mousemove', onDrag)
  document.addEventListener('mouseup', stopDrag)
}

function startDragRight(e: MouseEvent) {
  dragging = 'right'
  dragStartX = e.clientX
  dragStartWidth = rightWidth.value
  // 缓存边界值
  updateCacheBoundaries()
  // 拖拽时禁止文本选中
  document.body.style.userSelect = 'none'
  // 添加拖拽类禁用过渡动画
  document.querySelector('.right-pane')?.classList.add('dragging')
  document.addEventListener('mousemove', onDrag)
  document.addEventListener('mouseup', stopDrag)
}

// 使用 requestAnimationFrame 节流拖拽更新
let rafId: number | null = null
let pendingWidth: number | null = null
let pendingSide: 'left' | 'right' | null = null

// 缓存边界值，避免拖拽时重复计算
let cachedLeftMin = 0, cachedLeftMax = 0
let cachedRightMin = 0, cachedRightMax = 0

function updateCacheBoundaries() {
  cachedLeftMin = vwToPx(LEFT_MIN_PCT)
  cachedLeftMax = vwToPx(LEFT_MAX_PCT)
  cachedRightMin = vwToPx(RIGHT_MIN_PCT)
  cachedRightMax = vwToPx(RIGHT_MAX_PCT)
}

function updateWidth() {
  if (pendingWidth === null || pendingSide === null) return
  if (pendingSide === 'left') {
    leftWidth.value = pendingWidth
    if (leftCollapsed.value) leftCollapsed.value = false
  } else {
    rightWidth.value = pendingWidth
    if (rightCollapsed.value) rightCollapsed.value = false
  }
  pendingWidth = null
  pendingSide = null
  rafId = null
}

function onDrag(e: MouseEvent) {
  // 防止拖拽时选中文本
  e.preventDefault()
  if (dragging === 'left') {
    const delta = e.clientX - dragStartX
    pendingWidth = Math.min(cachedLeftMax, Math.max(cachedLeftMin, dragStartWidth + delta))
    pendingSide = 'left'
  } else if (dragging === 'right') {
    const delta = dragStartX - e.clientX   // 右侧：向左拖变大
    pendingWidth = Math.min(cachedRightMax, Math.max(cachedRightMin, dragStartWidth + delta))
    pendingSide = 'right'
  }
  if (rafId === null) {
    rafId = requestAnimationFrame(updateWidth)
  }
}

function stopDrag() {
  dragging = null
  // 取消未执行的动画帧
  if (rafId !== null) {
    cancelAnimationFrame(rafId)
    rafId = null
    // 应用最后待处理的宽度
    if (pendingWidth !== null && pendingSide !== null) {
      updateWidth()
    }
  }
  // 移除拖拽类恢复过渡动画
  document.querySelector('.left-pane')?.classList.remove('dragging')
  document.querySelector('.right-pane')?.classList.remove('dragging')
  // 恢复文本选中
  document.body.style.userSelect = ''
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
}

// ━━━ Workbench 逻辑 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
async function onSidebarChapterSelect(chapterId: number, title = '') {
  await handleChapterSelect(chapterId, title)
  workAreaRef.value?.ensureAssistedMode?.()
}

const handleChapterUpdated = async () => {
  await loadDesk()
  void statsStore.loadBookStats(slug, true).catch(() => {})
  biblePanelKey.value += 1
  chapterListRef.value?.refreshStoryTree?.()
  workbenchRefresh.bumpAfterChapterDeskChange()
}

// 幕→章 规划弹层
const showActPlanning = ref(false)
const showLLMSettings = ref(false)
const actPlanningId = ref('')
const actPlanningTitle = ref('')

const handlePlanAct = (actId: string, actTitle: string) => {
  actPlanningId.value = actId
  actPlanningTitle.value = actTitle
  showActPlanning.value = true
}

const {
  bookTitle,
  chapters,
  rightPanel,
  biblePanelKey,
  pageLoading,
  bookMeta,
  currentJobId,
  currentChapterId,
  chapterContent,
  chapterLoading,
  targetWordsPerChapter,
  setRightPanel,
  loadDesk,
  goHome,
  goToChapter,
  handleChapterSelect,
} = useWorkbench({ slug })

const currentChapter = computed(() => {
  if (!currentChapterId.value) return null
  return chapters.value.find(ch => ch.id === currentChapterId.value) || null
})

function onSettingsPanelChange(panel: string) {
  rightPanel.value = panel
}

function parseChapterQuery(q: unknown): number | null {
  if (q == null || q === '') return null
  const raw = Array.isArray(q) ? q[0] : q
  const n = Number(raw)
  return !Number.isNaN(n) && n >= 1 ? n : null
}

async function syncChapterFromRoute() {
  const n = parseChapterQuery(route.query.chapter)
  if (n != null) {
    await goToChapter(n)
  }
}

// 窗口大小变化时重新计算面板宽度（保持比例）
let resizeRafId: number | null = null
function handleResize() {
  if (resizeRafId !== null) return
  resizeRafId = requestAnimationFrame(() => {
    resizeRafId = null
    if (!leftCollapsed.value) {
      const currentPct = (leftWidth.value / window.innerWidth) * 100
      // 保持在有效范围内
      leftWidth.value = vwToPx(Math.min(LEFT_MAX_PCT, Math.max(LEFT_MIN_PCT, currentPct)))
    }
    if (!rightCollapsed.value) {
      const currentPct = (rightWidth.value / window.innerWidth) * 100
      rightWidth.value = vwToPx(Math.min(RIGHT_MAX_PCT, Math.max(RIGHT_MIN_PCT, currentPct)))
    }
  })
}

onMounted(async () => {
  try {
    await loadDesk()
    await syncChapterFromRoute()
  } catch {
    message.error('加载失败，请检查网络与后端是否已启动')
    bookTitle.value = slug
  } finally {
    pageLoading.value = false
  }
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  stopDrag()
  window.removeEventListener('resize', handleResize)
  // 取消未执行的 resize RAF
  if (resizeRafId !== null) {
    cancelAnimationFrame(resizeRafId)
    resizeRafId = null
  }
})

watch(
  () => route.query.chapter,
  () => {
    void syncChapterFromRoute()
  }
)
</script>

<style scoped>
.workbench {
  height: 100vh;
  min-height: 0;
  background: var(--app-page-bg, #f0f2f8);
  display: flex;
  flex-direction: column;
}

.workbench-spin {
  flex: 1;
  min-height: 0;
}

.workbench-spin :deep(.n-spin-content) {
  height: 100%;
  min-height: 100%;
}

.workbench-inner {
  height: 100%;
  min-height: 0;
  display: flex;
  flex-direction: row;
  overflow: hidden;
}

/* ━━━ 侧面板 ━━━ */
.side-pane {
  flex-shrink: 0;
  overflow: hidden;
  transition: width 0.2s ease;
  min-width: 0;
  /* 拖拽时优化渲染性能 */
  will-change: width;
}

/* 拖拽时禁用过渡动画，避免卡顿 */
.side-pane.dragging {
  transition: none;
}

.left-pane  { border-right: 1px solid var(--aitext-split-border, #e4e4e4); }
.right-pane { border-left:  1px solid var(--aitext-split-border, #e4e4e4); }

/* 折叠时去掉边框避免 1px 残留 */
.left-pane[style*="width: 0"],
.left-pane[style*="width:0"]  { border-right: none; }
.right-pane[style*="width: 0"],
.right-pane[style*="width:0"] { border-left: none; }

/* ━━━ 中间区 ━━━ */
.main-pane {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  /* 避免面板宽度变化时重排 */
  contain: layout;
}

/* ━━━ 分割线 ━━━ */
.divider {
  position: relative;
  flex-shrink: 0;
  width: 5px;
  background: transparent;
  cursor: col-resize;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  /* 拖拽时防止选中文本 */
  user-select: none;
  -webkit-user-select: none;
}

.divider::after {
  content: '';
  position: absolute;
  top: 0; bottom: 0;
  left: 2px;
  width: 1px;
  background: transparent;
  transition: background 0.15s;
}

.divider:hover::after {
  background: var(--n-primary-color, #18a058);
}

/* ━━━ 折叠按钮 ━━━ */
.collapse-btn {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 20;
  width: 16px;
  height: 44px;
  padding: 0;
  border: 1px solid var(--aitext-split-border, #ddd);
  border-radius: 4px;
  background: var(--app-surface, #fff);
  cursor: pointer;
  font-size: 12px;
  color: #aaa;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 1px 6px rgba(0,0,0,0.10);
  transition: color 0.15s, background 0.15s, border-color 0.15s;
  user-select: none;
  line-height: 1;
}

.collapse-btn:hover {
  color: var(--n-primary-color, #18a058);
  background: #f0faf5;
  border-color: var(--n-primary-color, #18a058);
}
</style>
