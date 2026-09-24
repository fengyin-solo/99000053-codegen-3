<template>
  <div class="board-page">
    <div class="board-header">
      <div class="board-title">
        <el-button text :icon="ArrowLeft" @click="$router.push('/')">Back</el-button>
        <h2 v-if="boardStore.currentBoard">{{ boardStore.currentBoard.name }}</h2>
      </div>
      <div class="board-actions">
        <!-- Quick column navigator -->
        <div v-if="!boardStore.loading" class="column-navigator" role="group" aria-label="Column navigator">
          <el-button
            :icon="ArrowLeftBold"
            :disabled="columns.length === 0 || atStart"
            aria-label="Previous page of columns"
            @click="pagePrev"
          />
          <el-select
            v-model="activeColumnId"
            class="column-nav-select"
            :placeholder="columns.length === 0 ? 'No columns' : 'Go to column'"
            filterable
            :disabled="columns.length === 0"
            @change="onSelectColumn"
          >
            <el-option
              v-for="(col, idx) in columns"
              :key="col.id"
              :label="col.name"
              :value="col.id"
            >
              <span class="nav-option-index">{{ idx + 1 }}.</span>
              <span class="nav-option-name">{{ col.name }}</span>
            </el-option>
          </el-select>
          <span class="column-nav-position">{{ positionText }}</span>
          <el-button
            :icon="ArrowRightBold"
            :disabled="columns.length === 0 || atEnd"
            aria-label="Next page of columns"
            @click="pageNext"
          />
        </div>
        <el-button type="primary" :icon="Plus" @click="showAddColumn = true">
          Add Column
        </el-button>
      </div>
    </div>

    <div v-if="boardStore.loading" class="loading-state">
      <el-icon class="is-loading" :size="32"><Loading /></el-icon>
      <p>Loading board...</p>
    </div>

    <div v-else ref="scrollerRef" class="columns-container" @scroll="onScrollerScroll">
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
            :data-column-id="column.id"
            :cards="boardStore.cards[column.id] || []"
            :all-columns="boardStore.columns"
            @add-card="handleAddCard"
            @edit-card="openCardDetail"
            @delete-card="confirmDeleteCard"
            @move-card="handleMoveCard"
            @rename-column="handleRenameColumn"
            @delete-column="confirmDeleteColumn"
          />
        </template>
      </draggable>
      <el-empty
        v-if="columns.length === 0"
        class="empty-columns"
        description="No columns yet. Add your first column!"
      />
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
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Plus, ArrowLeft, ArrowLeftBold, ArrowRightBold, Loading } from '@element-plus/icons-vue'
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

// --- Column quick navigation / paging state --------------------------------
const columns = computed(() => boardStore.columns)

const scrollerRef = ref(null)
const activeColumnId = ref(null)   // column the navigator points at
const atStart = ref(true)          // pager: already at the first column
const atEnd = ref(true)            // pager: already at the last column

let boardLoadingToken = 0
let scrollRafId = null
let resizeRafId = null

const positionText = computed(() => {
  const idx = columns.value.findIndex(c => c.id === activeColumnId.value)
  if (idx === -1) return `0 / ${columns.value.length}`
  return `${idx + 1} / ${columns.value.length}`
})

function storageKey(boardId) {
  return `board:${boardId}:activeColumnId`
}

function findColumnEl(columnId) {
  if (!scrollerRef.value || columnId == null) return null
  return scrollerRef.value.querySelector(`.column[data-column-id="${columnId}"]`)
}

// Scroll so the given column's left edge aligns with the container's left edge.
function scrollToColumn(columnId, behavior = 'smooth') {
  const el = findColumnEl(columnId)
  const scroller = scrollerRef.value
  if (!el || !scroller) return
  const target = scroller.scrollLeft + el.getBoundingClientRect().left
    - scroller.getBoundingClientRect().left
  scroller.scrollTo({ left: Math.max(0, target), behavior })
}

// Snapshot each rendered column's offset relative to the scroll container.
function columnLayouts() {
  const scroller = scrollerRef.value
  if (!scroller) return []
  const scrollerRect = scroller.getBoundingClientRect()
  return columns.value
    .map(col => {
      const el = findColumnEl(col.id)
      if (!el) return null
      const rect = el.getBoundingClientRect()
      return {
        id: col.id,
        left: rect.left - scrollerRect.left + scroller.scrollLeft,
        width: rect.width
      }
    })
    .filter(Boolean)
}

// First column not scrolled past the container's left edge.
function leftmostColumnId() {
  const scroller = scrollerRef.value
  if (!scroller) return null
  const sl = scroller.scrollLeft
  let pick = null
  for (const item of columnLayouts()) {
    if (item.left <= sl + 1) pick = item.id
    else break
  }
  return pick
}

function updateEdges() {
  const scroller = scrollerRef.value
  if (!scroller || columns.value.length === 0) {
    atStart.value = true
    atEnd.value = true
    return
  }
  const maxScroll = scroller.scrollWidth - scroller.clientWidth
  atStart.value = scroller.scrollLeft <= 1
  atEnd.value = scroller.scrollLeft >= maxScroll - 1
}

function syncFromScroll() {
  updateEdges()
  // While a programmatic smooth scroll is animating, keep the selection locked
  // to its target unless the target has already reached the left edge.
  if (scrollingProgrammatically) {
    const layouts = columnLayouts()
    const target = layouts.find(i => i.id === programmaticTargetId)
    if (target && target.left > scrollerRef.value.scrollLeft + 1) return
    clearTimeout(programmaticTimer)
    scrollingProgrammatically = false
    programmaticTargetId = null
  }
  // Keep the locate-by-name select in sync with what is actually in view.
  const id = leftmostColumnId()
  if (id != null && id !== activeColumnId.value) {
    activeColumnId.value = id
    persistActiveColumn(id)
  }
}

function onScrollerScroll() {
  if (scrollRafId != null) return
  scrollRafId = requestAnimationFrame(() => {
    scrollRafId = null
    syncFromScroll()
  })
}

function onWindowResize() {
  if (resizeRafId != null) return
  resizeRafId = requestAnimationFrame(() => {
    resizeRafId = null
    updateEdges()
  })
}

let scrollingProgrammatically = false
let programmaticTargetId = null
let programmaticTimer = null
function scrollAndTrack(columnId, behavior = 'smooth') {
  if (columnId == null) return
  activeColumnId.value = columnId
  persistActiveColumn(columnId)
  scrollingProgrammatically = true
  programmaticTargetId = columnId
  clearTimeout(programmaticTimer)
  scrollToColumn(columnId, behavior)
  // Safety net in case no scroll event settles (already in position).
  programmaticTimer = setTimeout(() => {
    scrollingProgrammatically = false
    programmaticTargetId = null
    syncFromScroll()
  }, 500)
  nextTick(updateEdges)
}

function persistActiveColumn(columnId) {
  if (!boardStore.currentBoard || columnId == null) return
  try {
    localStorage.setItem(storageKey(boardStore.currentBoard.id), String(columnId))
  } catch {
    // storage unavailable; navigation still works for this session
  }
}

function clearPersistedActiveColumn() {
  if (!boardStore.currentBoard) return
  try {
    localStorage.removeItem(storageKey(boardStore.currentBoard.id))
  } catch {
    // ignore
  }
}

// Locate by name (select): jump straight to the chosen column.
function onSelectColumn(columnId) {
  if (columnId == null) return
  scrollAndTrack(columnId, 'smooth')
}

// Paging: advance by roughly one viewport worth of columns.
function pageNext() {
  if (columns.value.length === 0 || atEnd.value) return
  const scroller = scrollerRef.value
  if (!scroller) return
  const sl = scroller.scrollLeft
  const viewport = scroller.clientWidth
  const layouts = columnLayouts()
  // First column starting at/after the current viewport's right edge.
  const next = layouts.find(item => item.left >= sl + viewport - 1)
  scrollAndTrack(next ? next.id : layouts[layouts.length - 1].id)
}

function pagePrev() {
  if (columns.value.length === 0 || atStart.value) return
  const scroller = scrollerRef.value
  if (!scroller) return
  const sl = scroller.scrollLeft
  const viewport = scroller.clientWidth
  const layouts = columnLayouts()
  // First column whose right edge passes one viewport to the left.
  const prev = layouts.find(item => item.left + item.width > sl - viewport)
  scrollAndTrack(prev ? prev.id : layouts[0].id)
}

// Column set changed (add / delete / drag reorder / board switch).
watch(columns, async (newCols, oldCols) => {
  await nextTick()
  if (!scrollerRef.value) return

  // Keep the navigator valid: current selection still exists?
  const exists = newCols.some(c => c.id === activeColumnId.value)
  if (exists) {
    // Order may have changed after a drag; re-align with the moved column.
    scrollToColumn(activeColumnId.value, 'auto')
  } else if (oldCols && oldCols.length > 0 && newCols.length > 0) {
    // The active column was deleted: fall back to the leftmost column in view
    // rather than jumping across the board.
    const fallback = leftmostColumnId() ?? newCols[0].id
    activeColumnId.value = fallback
    persistActiveColumn(fallback)
    scrollToColumn(fallback, 'auto')
  } else if (newCols.length > 0) {
    activeColumnId.value = newCols[0].id
  } else {
    // Board now has no columns; don't leave a stale selection behind.
    activeColumnId.value = null
    clearPersistedActiveColumn()
  }
  // The browser clamps scrollLeft asynchronously when content shrinks; let the
  // layout settle, then re-clamp the scroll and pager edges to the final state.
  requestAnimationFrame(() => {
    if (!scrollerRef.value) return
    if (activeColumnId.value != null) {
      scrollToColumn(activeColumnId.value, 'auto')
    } else {
      scrollerRef.value.scrollLeft = 0
    }
    updateEdges()
  })
  updateEdges()
}, { deep: false })

// --- Board loading ----------------------------------------------------------
async function loadBoard(boardId) {
  const token = ++boardLoadingToken
  boardStore.currentBoard = { id: boardId, name: 'Loading...' }
  // Reset navigator up front so a previous board can never show through.
  activeColumnId.value = null
  atStart.value = true
  atEnd.value = true
  scrollerRef.value?.scrollTo({ left: 0 })

  try {
    await boardStore.fetchColumns(boardId)
    if (token !== boardLoadingToken) return
    await boardStore.fetchAllCards(boardId)
    if (token !== boardLoadingToken) return

    // Get board name from boards list or set from URL
    let found = boardStore.boards.find(b => b.id === boardId)
    if (!found) {
      await boardStore.fetchBoards()
      if (token !== boardLoadingToken) return
      found = boardStore.boards.find(b => b.id === boardId)
    }
    if (found) boardStore.currentBoard = found

    await nextTick()
    if (token !== boardLoadingToken) return
    restoreNavigation(boardId)
  } catch (err) {
    if (token !== boardLoadingToken) return
    ElMessage.error('Failed to load board')
    router.push('/')
  }
}

function restoreNavigation(boardId) {
  const cols = boardStore.columns
  if (cols.length === 0 || !scrollerRef.value) {
    activeColumnId.value = null
    updateEdges()
    return
  }
  // Reopen the board at the column last viewed; fall back to the first one
  // when the stored column is gone (deleted, moved board, etc.).
  let storedId = null
  try {
    storedId = localStorage.getItem(storageKey(boardId))
  } catch {
    storedId = null
  }
  if (storedId != null) storedId = Number(storedId)
  const target = cols.some(c => c.id === storedId) ? storedId : cols[0].id
  if (target !== storedId) clearPersistedActiveColumn()
  activeColumnId.value = target
  persistActiveColumn(target)
  scrollToColumn(target, 'auto')
  updateEdges()
}

// Navigating directly from one board to another reuses this component,
// so the :id param must trigger a full (reset) reload.
watch(() => route.params.id, (newId, oldId) => {
  if (newId == null || Number(newId) === Number(oldId)) return
  loadBoard(Number(newId))
})

onMounted(async () => {
  window.addEventListener('resize', onWindowResize)
  await loadBoard(parseInt(route.params.id))
})

onUnmounted(() => {
  boardLoadingToken++ // drop any in-flight load for this view
  window.removeEventListener('resize', onWindowResize)
  clearTimeout(programmaticTimer)
  if (scrollRafId != null) cancelAnimationFrame(scrollRafId)
  if (resizeRafId != null) cancelAnimationFrame(resizeRafId)
  boardStore.clearBoard()
})

async function handleAddColumn() {
  if (!newColumnName.value.trim()) return
  try {
    const created = await boardStore.addColumn(boardStore.currentBoard.id, newColumnName.value.trim())
    newColumnName.value = ''
    showAddColumn.value = false
    ElMessage.success('Column added')
    // Bring the freshly added column into view.
    await nextTick()
    scrollAndTrack(created.id, 'smooth')
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
  margin-bottom: 20px;
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

.board-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.column-navigator {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #fff;
  border: 1px solid #dcdfe6;
  border-radius: 8px;
  padding: 4px 8px;
}

.column-nav-select {
  width: 200px;
}

.column-nav-position {
  font-size: 12px;
  color: #909399;
  min-width: 42px;
  text-align: center;
  user-select: none;
}

.nav-option-index {
  color: #909399;
  margin-right: 8px;
}

.nav-option-name {
  color: #303133;
}

.columns-container {
  flex: 1;
  overflow-x: auto;
  overflow-y: hidden;
  position: relative;
}

.columns-wrapper {
  display: flex;
  gap: 16px;
  height: 100%;
  min-height: 400px;
}

.empty-columns {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding-top: 80px;
  background: transparent;
}

.column-ghost {
  opacity: 0.5;
  background: #e8f4ff;
  border-radius: 8px;
}

.loading-state {
  text-align: center;
  padding: 60px;
  color: #909399;
}

.loading-state p {
  margin: 12px;
}
</style>
