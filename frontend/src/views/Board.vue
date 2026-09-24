<template>
  <div class="board-page">
    <div class="board-header">
      <div class="board-title">
        <el-button text :icon="ArrowLeft" @click="$router.push('/')">Back</el-button>
        <h2 v-if="boardStore.currentBoard">{{ boardStore.currentBoard.name }}</h2>
      </div>
      <div class="board-actions">
        <el-button type="primary" :icon="Plus" @click="showAddColumn = true">
          Add Column
        </el-button>
      </div>
    </div>

    <!-- Quick column locating & paging -->
    <div v-if="!boardStore.loading && boardStore.columns.length" class="board-toolbar">
      <el-select
        v-model="selectedColumnId"
        class="column-locator"
        filterable
        :placeholder="`Locate a column (${boardStore.columns.length})`"
        @change="onLocatorChange"
      >
        <el-option
          v-for="(col, idx) in boardStore.columns"
          :key="col.id"
          :label="col.name"
          :value="col.id"
        >
          <span class="locator-index">{{ idx + 1 }}.</span>
          <span class="locator-name">{{ col.name }}</span>
        </el-option>
      </el-select>

      <div class="pager">
        <el-tooltip content="First page" placement="top">
          <el-button text :icon="DArrowLeft" :disabled="atStart" @click="scrollPages(-Infinity)" />
        </el-tooltip>
        <el-tooltip content="Previous page" placement="top">
          <el-button text :icon="ArrowLeft" :disabled="atStart" @click="scrollPages(-1)" />
        </el-tooltip>
        <span class="pager-range">{{ visibleRange }}</span>
        <el-tooltip content="Next page" placement="top">
          <el-button text :icon="ArrowRight" :disabled="atEnd" @click="scrollPages(1)" />
        </el-tooltip>
        <el-tooltip content="Last page" placement="top">
          <el-button text :icon="DArrowRight" :disabled="atEnd" @click="scrollPages(Infinity)" />
        </el-tooltip>
      </div>
    </div>

    <div v-if="boardStore.loading" class="loading-state">
      <el-icon class="is-loading" :size="32"><Loading /></el-icon>
      <p>Loading board...</p>
    </div>

    <div v-else-if="!boardStore.columns.length" class="empty-state">
      <el-empty description="No columns yet. Add your first column to get started!" />
    </div>

    <div v-else ref="scrollerRef" class="columns-container" @scroll="updateScrollState">
      <draggable
        v-model="boardStore.columns"
        item-key="id"
        class="columns-wrapper"
        ghost-class="column-ghost"
        animation="200"
        @end="onColumnDragEnd"
      >
        <template #item="{ element: column }">
          <Column
            :column="column"
            :cards="boardStore.cards[column.id] || []"
            :all-columns="boardStore.columns"
            :data-column-id="column.id"
            :class="{ 'column-located': flashColumnId === column.id }"
            @add-card="handleAddCard"
            @edit-card="openCardDetail"
            @delete-card="confirmDeleteCard"
            @move-card="handleMoveCard"
            @rename-column="handleRenameColumn"
            @delete-column="confirmDeleteColumn"
          />
        </template>
      </draggable>
    </div>

    <!-- Add Column Dialog -->
    <el-dialog v-model="showAddColumn" title="Add Column" width="400px" :close-on-click-modal="false">
      <el-form @submit.prevent="handleAddColumn">
        <el-form-item label="Column Name">
          <el-input v-model="newColumnName" placeholder="Enter column name" @keyup.enter="handleAddColumn" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showAddColumn = false">Cancel</el-button>
        <el-button type="primary" :disabled="!newColumnName.trim()" @click="handleAddColumn">Add</el-button>
      </template>
    </el-dialog>

    <!-- Add Card Dialog -->
    <AddCardForm
      v-model:visible="showAddCard"
      :column-id="addingToColumnId"
      @added="onCardAdded"
    />

    <!-- Card Detail Dialog -->
    <CardDetail
      v-model:visible="showCardDetail"
      :card="selectedCard"
      :all-columns="boardStore.columns"
      @updated="onCardUpdated"
      @move="handleMoveCard"
    />
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  Plus, ArrowLeft, ArrowRight, DArrowLeft, DArrowRight, Loading
} from '@element-plus/icons-vue'
import draggable from 'vuedraggable'
import { useBoardStore } from '../stores/board.js'
import { columnApi } from '../api/index.js'
import Column from '../components/Column.vue'
import AddCardForm from '../components/AddCardForm.vue'
import CardDetail from '../components/CardDetail.vue'

const route = useRoute()
const router = useRouter()
const boardStore = useBoardStore()

const showAddColumn = ref(false)
const newColumnName = ref('')
const showAddCard = ref(false)
const addingToColumnId = ref(null)
const showCardDetail = ref(false)
const selectedCard = ref(null)

// --- Column locating & paging ---------------------------------------------
const scrollerRef = ref(null)
const selectedColumnId = ref(null)
const flashColumnId = ref(null)
const atStart = ref(true)
const atEnd = ref(true)
const visibleStartIndex = ref(0)
const visibleEndIndex = ref(0)
// Suppresses persistence while the saved column is being restored on reopen.
let restoringLocator = false

const storageKey = () => `task-board:${route.params.id}:selectedColumn`

function saveSelectedColumn(id) {
  try {
    if (id !== null && id !== undefined) {
      localStorage.setItem(storageKey(), String(id))
    } else {
      localStorage.removeItem(storageKey())
    }
  } catch {
    // localStorage may be unavailable; locating still works for this session
  }
}

function getColumnElement(columnId) {
  return scrollerRef.value?.querySelector(`[data-column-id="${columnId}"]`)
}

function scrollColumnIntoView(columnId, flash = true) {
  const el = getColumnElement(columnId)
  const scroller = scrollerRef.value
  if (!el || !scroller) return
  const relativeLeft =
    el.getBoundingClientRect().left - scroller.getBoundingClientRect().left + scroller.scrollLeft
  const targetLeft = relativeLeft - 8
  const maxScroll = scroller.scrollWidth - scroller.clientWidth
  scroller.scrollTo({
    left: Math.max(0, Math.min(targetLeft, maxScroll)),
    behavior: 'smooth'
  })
  if (flash) {
    flashColumnId.value = null
    // Restart the highlight animation even when the same column is picked again
    requestAnimationFrame(() => {
      flashColumnId.value = columnId
      setTimeout(() => {
        if (flashColumnId.value === columnId) flashColumnId.value = null
      }, 1600)
    })
  }
}

function onLocatorChange(columnId) {
  if (restoringLocator) return
  saveSelectedColumn(columnId)
  nextTick(() => scrollColumnIntoView(columnId))
}

function scrollPages(pages) {
  const scroller = scrollerRef.value
  if (!scroller) return
  const maxScroll = scroller.scrollWidth - scroller.clientWidth
  let target
  if (pages === -Infinity) {
    target = 0
  } else if (pages === Infinity) {
    target = maxScroll
  } else {
    target = scroller.scrollLeft + pages * (scroller.clientWidth - 48)
  }
  scroller.scrollTo({
    left: Math.max(0, Math.min(target, maxScroll)),
    behavior: 'smooth'
  })
}

function updateScrollState() {
  const scroller = scrollerRef.value
  if (!scroller) return
  const { scrollLeft, clientWidth } = scroller
  atStart.value = scrollLeft <= 1
  atEnd.value = scrollLeft + clientWidth >= scroller.scrollWidth - 1

  // Determine which columns intersect the visible viewport (in DOM order).
  const scrollerRect = scroller.getBoundingClientRect()
  const viewLeft = scrollerRect.left
  const viewRight = scrollerRect.right
  let first = -1
  let last = -1
  boardStore.columns.forEach((col, i) => {
    const el = getColumnElement(col.id)
    if (!el) return
    const rect = el.getBoundingClientRect()
    if (rect.right > viewLeft && rect.left < viewRight) {
      if (first === -1) first = i
      last = i
    }
  })
  if (first !== -1) {
    visibleStartIndex.value = first
    visibleEndIndex.value = last
  }
}

const visibleRange = computed(() => {
  const total = boardStore.columns.length
  if (!total) return ''
  return `${visibleStartIndex.value + 1}–${visibleEndIndex.value + 1} / ${total}`
})

// Keep pager state in sync with column add/remove/reorder and viewport resize
watch(
  () => boardStore.columns.map(c => c.id).join(','),
  () => {
    nextTick(() => {
      updateScrollState()
      if (selectedColumnId.value &&
          !boardStore.columns.some(c => c.id === selectedColumnId.value)) {
        selectedColumnId.value = null
        saveSelectedColumn(null)
      }
    })
  }
)

function onWindowResize() {
  updateScrollState()
}

// --- Board loading ---------------------------------------------------------
async function loadBoard(boardId) {
  // Reset per-board locating state up front so a previous board's selection
  // or scroll position can never carry into this one.
  restoringLocator = true
  selectedColumnId.value = null
  flashColumnId.value = null
  if (scrollerRef.value) scrollerRef.value.scrollLeft = 0

  boardStore.currentBoard = { id: boardId, name: 'Loading...' }
  try {
    await boardStore.fetchColumns(boardId)
    await boardStore.fetchAllCards(boardId)
    // Get board name from boards list or fetch it
    let found = boardStore.boards.find(b => b.id === boardId)
    if (!found) {
      await boardStore.fetchBoards()
      found = boardStore.boards.find(b => b.id === boardId)
    }
    if (found) boardStore.currentBoard = found

    // Restore the previously located column only if it still exists
    let savedId = null
    try {
      savedId = parseInt(localStorage.getItem(storageKey()), 10)
    } catch {
      savedId = null
    }
    await nextTick()
    if (savedId && boardStore.columns.some(c => c.id === savedId)) {
      selectedColumnId.value = savedId
      scrollColumnIntoView(savedId, false)
    } else {
      saveSelectedColumn(null)
      selectedColumnId.value = null
    }
    updateScrollState()
  } catch (err) {
    ElMessage.error('Failed to load board')
    router.push('/')
  } finally {
    restoringLocator = false
  }
}

onMounted(() => {
  loadBoard(parseInt(route.params.id))
  window.addEventListener('resize', onWindowResize)
})

// Same component is reused when navigating directly between boards
watch(() => route.params.id, (newId, oldId) => {
  if (newId && parseInt(newId) !== parseInt(oldId)) {
    loadBoard(parseInt(newId))
  }
})

onUnmounted(() => {
  window.removeEventListener('resize', onWindowResize)
  boardStore.clearBoard()
})

async function handleAddColumn() {
  if (!newColumnName.value.trim()) return
  try {
    const col = await boardStore.addColumn(boardStore.currentBoard.id, newColumnName.value.trim())
    newColumnName.value = ''
    showAddColumn.value = false
    ElMessage.success('Column added')
    await nextTick()
    updateScrollState()
    // Locate the freshly added column so it is visible even far off-screen
    restoringLocator = false
    selectedColumnId.value = col.id
    saveSelectedColumn(col.id)
    scrollColumnIntoView(col.id)
  } catch (err) {
    ElMessage.error('Failed to add column')
  }
}

function handleAddCard(columnId) {
  addingToColumnId.value = columnId
  showAddCard.value = true
}

function onCardAdded() {
  showAddCard.value = false
}

function openCardDetail(card) {
  selectedCard.value = { ...card }
  showCardDetail.value = true
}

function onCardUpdated(updatedCard) {
  selectedCard.value = { ...updatedCard }
}

async function confirmDeleteCard(card) {
  try {
    await ElMessageBox.confirm(
      `Delete "${card.title}"?`,
      'Delete Card',
      { type: 'warning', confirmButtonText: 'Delete', cancelButtonText: 'Cancel' }
    )
    await boardStore.deleteCard(card.id)
    ElMessage.success('Card deleted')
  } catch (err) {
    // cancelled
  }
}

async function handleMoveCard(cardId, targetColumnId, position) {
  try {
    await boardStore.moveCard(cardId, targetColumnId, position)
    ElMessage.success('Card moved')
  } catch (err) {
    ElMessage.error('Failed to move card')
  }
}

async function handleRenameColumn(columnId, newName) {
  try {
    await boardStore.renameColumn(columnId, newName)
    ElMessage.success('Column renamed')
  } catch (err) {
    ElMessage.error('Failed to rename column')
  }
}

async function confirmDeleteColumn(column) {
  const cardCount = (boardStore.cards[column.id] || []).length
  const msg = cardCount > 0
    ? `Delete "${column.name}" and its ${cardCount} card(s)?`
    : `Delete "${column.name}"?`
  try {
    await ElMessageBox.confirm(msg, 'Delete Column', {
      type: 'warning',
      confirmButtonText: 'Delete',
      cancelButtonText: 'Cancel'
    })
    await boardStore.deleteColumn(column.id)
    ElMessage.success('Column deleted')
    await nextTick()
    updateScrollState()
  } catch (err) {
    // cancelled
  }
}

async function onColumnDragEnd(evt) {
  // Update column positions after drag
  const columns = boardStore.columns
  for (let i = 0; i < columns.length; i++) {
    if (columns[i].position !== i) {
      try {
        await columnApi.update(columns[i].id, { position: i })
        columns[i].position = i
      } catch (err) {
        // Refresh to get correct state
        await boardStore.fetchColumns(boardStore.currentBoard.id)
        break
      }
    }
  }
  nextTick(updateScrollState)
}
</script>

<style scoped>
.board-page {
  padding: 20px;
  height: calc(100vh - 60px);
  display: flex;
  flex-direction: column;
}

.board-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  flex-shrink: 0;
}

.board-title {
  display: flex;
  align-items: center;
  gap: 12px;
}

.board-title h2 {
  font-size: 22px;
  color: #303133;
}

.board-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
  flex-shrink: 0;
}

.column-locator {
  width: 260px;
}

.locator-index {
  color: #909399;
  margin-right: 6px;
  font-variant-numeric: tabular-nums;
}

.locator-name {
  color: #303133;
}

.pager {
  display: flex;
  align-items: center;
  gap: 2px;
}

.pager-range {
  font-size: 13px;
  color: #909399;
  margin: 0 6px;
  min-width: 64px;
  text-align: center;
  font-variant-numeric: tabular-nums;
}

.columns-container {
  flex: 1;
  overflow-x: auto;
  overflow-y: hidden;
}

.columns-wrapper {
  display: flex;
  gap: 16px;
  height: 100%;
  min-height: 400px;
}

.column-ghost {
  opacity: 0.5;
  background: #e8f4ff;
  border-radius: 8px;
}

.columns-container :deep(.column-located) {
  animation: column-flash 1.4s ease;
}

@keyframes column-flash {
  0% {
    box-shadow: 0 0 0 0 rgba(64, 158, 255, 0);
  }
  25% {
    box-shadow: 0 0 0 3px rgba(64, 158, 255, 0.9);
  }
  100% {
    box-shadow: 0 0 0 3px rgba(64, 158, 255, 0);
  }
}

.empty-state,
.loading-state {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: 60px;
  color: #909399;
}

.loading-state {
  flex-direction: column;
}

.loading-state p {
  margin-top: 12px;
}
</style>
