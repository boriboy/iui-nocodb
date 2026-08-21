<script setup lang="ts">
import type { SortType, TableType } from 'nocodb-sdk'

definePageMeta({
  hideHeader: true,
  hasSidebar: true,
})

type CoralLevel = 'transect' | 'observation' | 'colony'
type RowRecord = Record<string, any>

const { $api } = useNuxtApp()
const { base } = storeToRefs(useBase())
const { baseTables } = storeToRefs(useTablesStore())
const { loadProjectTables } = useTablesStore()
const { getMeta } = useMetas()

const isBusy = ref(true)
const isLoadingRows = ref(false)
const isExporting = ref(false)
const errorMsg = ref('')

const transectMeta = ref<TableType | null>(null)
const observationMeta = ref<TableType | null>(null)
const colonyMeta = ref<TableType | null>(null)
const siteMeta = ref<TableType | null>(null)
const samplerMeta = ref<TableType | null>(null)
const categoryMeta = ref<TableType | null>(null)
const observationTypeMeta = ref<TableType | null>(null)

const level = ref<CoralLevel>('transect')
const rows = ref<RowRecord[]>([])
const page = ref(1)
const pageSize = ref(100)
const pageInfo = ref<{ totalRows?: number; page?: number; pageSize?: number }>({})
const sortState = ref<{ column: string; direction: 'asc' | 'desc' } | null>(null)
const showColumnsPanel = ref(false)
const drillTransectId = ref('')
const lastRequestId = ref(0)
const lastAppliedQueryFingerprint = ref('')

const selectedYears = ref<string[]>([])
const selectedLocations = ref<string[]>([])
const selectedDepths = ref<string[]>([])
const selectedSizes = ref<string[]>([])
const selectedSampler = ref('')
const selectedStatus = ref('')
const selectedCategory = ref('')
const selectedGenus = ref('')
const fromDate = ref('')
const toDate = ref('')

const years = ref<string[]>([])
const locations = ref<string[]>([])
const depths = ref<string[]>([])
const samplers = ref<string[]>([])
const categories = ref<string[]>([])
const genera = ref<string[]>([])

const hiddenColumns = reactive<Record<CoralLevel, string[]>>({
  transect: [],
  observation: [],
  colony: [],
})

const tableScrollRef = ref<HTMLElement | null>(null)
const topBarRef = ref<HTMLElement | null>(null)
const topThumbRef = ref<HTMLElement | null>(null)
const scrollHintRef = ref<HTMLElement | null>(null)

let isTopThumbDragging = false
let dragStartX = 0
let dragStartLeft = 0

const levelColumns: Record<CoralLevel, string[]> = {
  transect: ['Location', 'Site', 'Depth', 'Year', 'Date', 'Transect #', 'Transect ID', 'Sampler', 'Status', 'Transect length (m)', 'Observations', 'Colonies', 'Live stony cover %', 'Notes'],
  observation: ['Location', 'Site', 'Depth', 'Year', 'Date', 'Transect #', 'Transect ID', 'Sampler', 'Transect status', 'Transect length (m)', 'Observation #', 'From (m)', 'To (m)', 'Length (cm)', 'Category', 'Observation', 'Species', 'Size', '% Living', 'Bleached', 'Form', 'Color', 'Algae type', 'Condition', 'Joined with', 'Remark'],
  colony: ['Location', 'Site', 'Depth', 'Year', 'Date', 'Transect #', 'Transect ID', 'Sampler', 'Observation #', 'Category', 'Observation', 'Species', 'Size', '% Living', 'Joined?', 'Joined with'],
}

const defaultSortTitles: Record<CoralLevel, string[]> = {
  transect: ['Location', 'Depth', 'Date', 'Transect #'],
  observation: ['Location', 'Depth', 'Date', 'Transect #', 'Observation #'],
  colony: ['Location', 'Depth', 'Date', 'Transect #', 'Observation #'],
}

const visibleColumns = computed(() => levelColumns[level.value].filter((column) => !hiddenColumns[level.value].includes(column)))
const totalRows = computed(() => Number(pageInfo.value.totalRows ?? rows.value.length ?? 0))
const totalPages = computed(() => Math.max(1, Math.ceil(totalRows.value / pageSize.value)))
const activeViewMeta = computed(() => {
  if (level.value === 'transect') return transectMeta.value
  if (level.value === 'observation') return observationMeta.value
  return colonyMeta.value
})
const currentQueryFingerprint = computed(() => {
  return JSON.stringify({
    level: level.value,
    page: page.value,
    pageSize: pageSize.value,
    where: buildWhere(),
    sorts: buildSorts(),
    metaId: activeViewMeta.value?.id || '',
  })
})
const canRunQuery = computed(() => !isLoadingRows.value && currentQueryFingerprint.value !== lastAppliedQueryFingerprint.value)

const getColumnMeta = (meta: TableType | null, title: string) => meta?.columns?.find((column) => column.title === title) || null

const getColumnId = (meta: TableType | null, title: string) => getColumnMeta(meta, title)?.id as string | undefined

const normalizeLabel = (value: string) => value.toLowerCase().replace(/[^a-z0-9]/g, '')

const findColumnTitle = (meta: TableType | null, preferredTitle: string) => {
  const exact = getColumnMeta(meta, preferredTitle)
  if (exact?.title) return exact.title

  const wanted = normalizeLabel(preferredTitle)
  const fuzzy = meta?.columns?.find((column) => normalizeLabel(String(column.title || '')) === wanted)
  return (fuzzy?.title as string) || preferredTitle
}

const getRowValueByTitle = (row: RowRecord, title: string) => {
  if (row?.[title] != null && row?.[title] !== '') return row[title]

  const wanted = normalizeLabel(title)
  const foundKey = Object.keys(row || {}).find((key) => normalizeLabel(key) === wanted)
  if (foundKey) return row[foundKey]

  return ''
}

const tableByNames = (tables: TableType[], names: string[]) => {
  return (
    tables.find((table) => names.includes((table.table_name || '').toLowerCase())) ||
    tables.find((table) => names.includes((table.title || '').toLowerCase())) ||
    null
  )
}

const listAllRows = async (meta: TableType, params: Record<string, any> = {}, pageLimit = 1000) => {
  const baseId = base.value?.id
  if (!baseId) return []

  const collected: RowRecord[] = []
  let offset = 0

  while (true) {
    const response = await $api.dbTableRow.list('noco', baseId, meta.id as string, {
      ...params,
      limit: pageLimit,
      offset,
    })

    const list = (response?.list ?? []) as RowRecord[]
    collected.push(...list)

    if (list.length < pageLimit) break
    offset += pageLimit
  }

  return collected
}

const toggleString = (collection: string[], value: string) => {
  const index = collection.indexOf(value)
  if (index >= 0) collection.splice(index, 1)
  else collection.push(value)
}

const buildGroup = (field: string, values: string[], op: 'eq' | 'like' = 'eq') => values.map((value) => `(${field},${op},${value})`).join('~or')

const buildWhere = () => {
  const parts: string[] = ['(is_test,eq,0)']

  if (drillTransectId.value) {
    const transectIdTitle = findColumnTitle(activeViewMeta.value, 'Transect ID')
    parts.push(`(${transectIdTitle},eq,${drillTransectId.value})`)
  }

  if (selectedYears.value.length) parts.push(buildGroup('Year', selectedYears.value))
  if (selectedLocations.value.length) parts.push(buildGroup('Location', selectedLocations.value))
  if (selectedDepths.value.length) parts.push(buildGroup('Depth', selectedDepths.value))
  if (selectedSampler.value) parts.push(`(Sampler,eq,${selectedSampler.value})`)

  if (level.value === 'transect') {
    if (selectedStatus.value) parts.push(`(Status,eq,${selectedStatus.value})`)
  } else {
    if (selectedCategory.value) parts.push(`(Category,eq,${selectedCategory.value})`)
    if (selectedGenus.value) parts.push(`(Observation,eq,${selectedGenus.value})`)
    if (selectedSizes.value.length) parts.push(buildGroup('Size', selectedSizes.value))
    if (level.value === 'colony' && !selectedCategory.value) parts.push('(Category,neq,Substrate)')
  }

  if (fromDate.value) parts.push(`(Date,gte,${fromDate.value})`)
  if (toDate.value) parts.push(`(Date,lte,${toDate.value})`)

  return parts.filter(Boolean).join('~and')
}

const buildSorts = () => {
  const meta = activeViewMeta.value
  if (!meta) return [] as SortType[]

  if (sortState.value) {
    const column = getColumnId(meta, sortState.value.column)
    return column ? [{ fk_column_id: column, direction: sortState.value.direction }] : []
  }

  return defaultSortTitles[level.value]
    .map((title) => {
      const column = getColumnId(meta, title)
      return column ? { fk_column_id: column, direction: 'asc' as const } : null
    })
    .filter(Boolean) as SortType[]
}

const applySort = (column: string) => {
  if (sortState.value?.column === column) {
    sortState.value.direction = sortState.value.direction === 'asc' ? 'desc' : 'asc'
  } else {
    sortState.value = { column, direction: 'asc' }
  }
  page.value = 1
  loadRows()
}

const changePage = async (delta: number) => {
  const nextPage = Math.max(1, Math.min(totalPages.value, page.value + delta))
  if (nextPage === page.value) return

  page.value = nextPage
  await loadRows()
}

const resetSort = () => {
  sortState.value = null
}

const updateScrollMetrics = () => {
  const scroller = tableScrollRef.value
  const topBar = topBarRef.value
  const thumb = topThumbRef.value
  const scrollHint = scrollHintRef.value
  if (!scroller || !topBar || !thumb) return

  const scrollable = scroller.scrollWidth > scroller.clientWidth + 4
  topBar.style.display = scrollable ? '' : 'none'
  if (scrollHint) scrollHint.style.display = scrollable ? '' : 'none'

  if (!scrollable) return

  const barWidth = topBar.clientWidth
  const thumbWidth = Math.max(40, (barWidth * scroller.clientWidth) / scroller.scrollWidth)
  const maxScroll = scroller.scrollWidth - scroller.clientWidth
  const thumbLeft = maxScroll > 0 ? (scroller.scrollLeft / maxScroll) * (barWidth - thumbWidth) : 0

  thumb.style.width = `${thumbWidth}px`
  thumb.style.left = `${thumbLeft}px`
}

const syncTableScrollFromThumbLeft = (left: number) => {
  const scroller = tableScrollRef.value
  const topBar = topBarRef.value
  const thumb = topThumbRef.value
  if (!scroller || !topBar || !thumb) return

  const barWidth = topBar.clientWidth
  const thumbWidth = thumb.clientWidth
  const maxLeft = Math.max(0, barWidth - thumbWidth)
  const clampedLeft = Math.max(0, Math.min(left, maxLeft))
  const maxScroll = Math.max(0, scroller.scrollWidth - scroller.clientWidth)

  if (maxLeft === 0 || maxScroll === 0) {
    scroller.scrollLeft = 0
    return
  }

  const ratio = clampedLeft / maxLeft
  scroller.scrollLeft = ratio * maxScroll
}

const onTopBarMouseDown = (event: MouseEvent) => {
  const topBar = topBarRef.value
  const thumb = topThumbRef.value
  if (!topBar || !thumb) return

  const rect = topBar.getBoundingClientRect()
  const thumbHalf = thumb.clientWidth / 2
  const targetLeft = event.clientX - rect.left - thumbHalf

  syncTableScrollFromThumbLeft(targetLeft)
  updateScrollMetrics()
}

const onTopThumbMouseDown = (event: MouseEvent) => {
  const thumb = topThumbRef.value
  if (!thumb) return

  isTopThumbDragging = true
  dragStartX = event.clientX
  dragStartLeft = parseFloat(thumb.style.left || '0') || 0
}

const onGlobalMouseMove = (event: MouseEvent) => {
  if (!isTopThumbDragging) return
  const delta = event.clientX - dragStartX
  syncTableScrollFromThumbLeft(dragStartLeft + delta)
  updateScrollMetrics()
}

const onGlobalMouseUp = () => {
  isTopThumbDragging = false
}

const formatCsvCell = (value: any) => {
  const text = value == null ? '' : String(value)
  return `"${text.replace(/"/g, '""')}"`
}

const downloadCsv = (filename: string, columns: string[], data: RowRecord[]) => {
  const content = [columns.join(','), ...data.map((row) => columns.map((column) => formatCsvCell(row[column])).join(','))].join('\r\n')
  const blob = new Blob([content], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = filename
  anchor.style.display = 'none'
  document.body.appendChild(anchor)
  anchor.click()
  document.body.removeChild(anchor)
  URL.revokeObjectURL(url)
}

const loadFilterOptions = async () => {
  if (!transectMeta.value || !siteMeta.value || !samplerMeta.value || !categoryMeta.value || !observationTypeMeta.value) return

  const transects = await listAllRows(transectMeta.value, { where: '(is_test,eq,0)' }, 1000)
  years.value = [...new Set(transects.map((row) => String(row['Year'] ?? row.survey_year ?? '').trim()).filter(Boolean))].sort((a, b) => Number(a) - Number(b))
  locations.value = [...new Set(transects.map((row) => String(row['Location'] ?? row.location ?? '').trim()).filter(Boolean))].sort()

  const depthRows = await listAllRows(siteMeta.value, { where: '(is_test,eq,0)' }, 1000)
  depths.value = [...new Set(depthRows.map((row) => String(row.depth_m ?? row['Depth'] ?? '').trim()).filter(Boolean))].sort((a, b) => Number(a) - Number(b))

  const samplerRows = await listAllRows(samplerMeta.value, {}, 1000)
  samplers.value = [...new Set(samplerRows.map((row) => String(row.sampler_name ?? row.name ?? '').trim()).filter(Boolean))].sort()

  const categoryRows = await listAllRows(categoryMeta.value, {}, 1000)
  categories.value = [...new Set(categoryRows.map((row) => String(row.category_name ?? row.name ?? '').trim()).filter(Boolean))].sort()

  const genusRows = await listAllRows(observationTypeMeta.value, {}, 1000)
  genera.value = [...new Set(genusRows.map((row) => String(row.observation_name ?? row.name ?? '').trim()).filter(Boolean))].sort()
}

const loadRows = async () => {
  const meta = activeViewMeta.value
  const baseId = base.value?.id
  if (!meta || !baseId) return

  const requestId = ++lastRequestId.value
  const requestFingerprint = currentQueryFingerprint.value
  isLoadingRows.value = true

  try {
    const response = await $api.dbTableRow.list('noco', baseId, meta.id as string, {
      limit: pageSize.value,
      offset: (page.value - 1) * pageSize.value,
      where: buildWhere(),
      sortArrJson: stringifyFilterOrSortArr(buildSorts()),
    } as any)

    if (requestId !== lastRequestId.value) return

    rows.value = (response?.list ?? []) as RowRecord[]
    pageInfo.value = response?.pageInfo ?? {}
    lastAppliedQueryFingerprint.value = requestFingerprint

    const resolvedTotal = Number(response?.pageInfo?.totalRows ?? 0)
    const resolvedPages = Math.max(1, Math.ceil(resolvedTotal / pageSize.value))
    if (resolvedTotal && page.value > resolvedPages) {
      page.value = resolvedPages
      await loadRows()
      return
    }

    await nextTick()
    updateScrollMetrics()
  } catch (error: any) {
    if (requestId === lastRequestId.value) {
      errorMsg.value = await extractSdkResponseErrorMsg(error)
    }
  } finally {
    if (requestId === lastRequestId.value) {
      isLoadingRows.value = false
    }
  }
}

const runQuery = async () => {
  page.value = 1
  await loadRows()
}

const resetFilters = async () => {
  selectedYears.value = []
  selectedLocations.value = []
  selectedDepths.value = []
  selectedSizes.value = []
  selectedSampler.value = ''
  selectedStatus.value = ''
  selectedCategory.value = ''
  selectedGenus.value = ''
  fromDate.value = ''
  toDate.value = ''
  drillTransectId.value = ''
  level.value = 'transect'
  resetSort()
  page.value = 1
  await loadRows()
}

const openDrillDown = async (row: RowRecord) => {
  const transectId = String(getRowValueByTitle(row, 'Transect ID') ?? '').trim()
  if (!transectId) return

  drillTransectId.value = transectId
  // Clicking a transect row should always open the full observation list for that transect.
  level.value = 'observation'
  selectedCategory.value = ''
  selectedGenus.value = ''
  selectedSizes.value = []
  page.value = 1
  resetSort()
  await loadRows()
}

const clearDrillDown = async () => {
  drillTransectId.value = ''
  level.value = 'transect'
  page.value = 1
  resetSort()
  await loadRows()
}

const changeLevel = async (nextLevel: CoralLevel) => {
  level.value = nextLevel
  page.value = 1
  resetSort()
  await loadRows()
  await nextTick()
  updateScrollMetrics()
}

const exportCurrentCsv = async () => {
  const meta = activeViewMeta.value
  const baseId = base.value?.id
  if (!meta || !baseId || isExporting.value) return

  isExporting.value = true
  try {
    const columns = visibleColumns.value
    const responseRows: RowRecord[] = []
    let offset = 0

    while (true) {
      const response = await $api.dbTableRow.list('noco', baseId, meta.id as string, {
        limit: 1000,
        offset,
        where: buildWhere(),
        sortArrJson: stringifyFilterOrSortArr(buildSorts()),
      } as any)

      const list = (response?.list ?? []) as RowRecord[]
      responseRows.push(...list)
      if (list.length < 1000) break
      offset += 1000
    }

    const fileDate = new Date().toISOString().slice(0, 10)
    downloadCsv(`nmp-coral-${level.value}-${fileDate}.csv`, columns, responseRows)
  } catch (error: any) {
    message.error(await extractSdkResponseErrorMsg(error))
  } finally {
    isExporting.value = false
  }
}

const init = async () => {
  try {
    isBusy.value = true
    errorMsg.value = ''

    if (!base.value?.id) return

    await loadProjectTables(base.value.id)
    const tables = (baseTables.value.get(base.value.id) || []) as TableType[]

    const transectTable = tableByNames(tables, ['v_coral_transects'])
    const observationTable = tableByNames(tables, ['v_coral_observations'])
    const colonyTable = tableByNames(tables, ['v_coral_colonies'])
    const sitesTable = tableByNames(tables, ['sites'])
    const samplersTable = tableByNames(tables, ['samplers'])
    const categoriesTable = tableByNames(tables, ['observation_categories'])
    const observationTypesTable = tableByNames(tables, ['observation_types'])

    if (!transectTable || !observationTable || !colonyTable) {
      errorMsg.value = 'Required coral report views are missing in this base.'
      return
    }

    transectMeta.value = (await getMeta(base.value.id, transectTable.id as string)) as TableType
    observationMeta.value = (await getMeta(base.value.id, observationTable.id as string)) as TableType
    colonyMeta.value = (await getMeta(base.value.id, colonyTable.id as string)) as TableType
    siteMeta.value = sitesTable ? ((await getMeta(base.value.id, sitesTable.id as string)) as TableType) : null
    samplerMeta.value = samplersTable ? ((await getMeta(base.value.id, samplersTable.id as string)) as TableType) : null
    categoryMeta.value = categoriesTable ? ((await getMeta(base.value.id, categoriesTable.id as string)) as TableType) : null
    observationTypeMeta.value = observationTypesTable ? ((await getMeta(base.value.id, observationTypesTable.id as string)) as TableType) : null

    await loadFilterOptions()
    await loadRows()
  } catch (error: any) {
    errorMsg.value = await extractSdkResponseErrorMsg(error)
  } finally {
    isBusy.value = false
  }
}

onMounted(async () => {
  await init()
  await nextTick()
  updateScrollMetrics()
  window.addEventListener('resize', updateScrollMetrics)
  window.addEventListener('mousemove', onGlobalMouseMove)
  window.addEventListener('mouseup', onGlobalMouseUp)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', updateScrollMetrics)
  window.removeEventListener('mousemove', onGlobalMouseMove)
  window.removeEventListener('mouseup', onGlobalMouseUp)
})

const formatDisplayValue = (value: any) => (value == null || value === '' ? '—' : value)
</script>

<template>
  <div class="h-full overflow-auto bg-slate-200 text-slate-800">
    <header class="bg-slate-800 text-white px-6 py-3 flex items-center justify-between border-b-4 border-cyan-500">
      <div>
        <div class="font-semibold text-lg">NMP - Coral Query & Browse</div>
        <div class="text-xs opacity-80">Read-only coral reports with server-side filtering, pagination, and CSV download</div>
      </div>
    </header>

    <main class="p-4 space-y-4">
      <section v-if="errorMsg" class="rounded-lg border border-rose-300 bg-rose-50 text-rose-700 p-3 text-sm">
        {{ errorMsg }}
      </section>

      <section v-if="isBusy" class="rounded-lg border border-slate-200 bg-white p-4 text-sm text-slate-600">Loading coral query interface...</section>

      <template v-else>
        <section class="bg-white border border-slate-200 rounded-lg shadow-sm p-3">
          <div class="mt-4 inline-flex rounded-lg border border-slate-400 overflow-hidden bg-white shadow-sm" id="level-toggle">
            <button
              type="button"
              class="px-4 py-2 text-sm font-medium transition-colors duration-150"
              :class="
                level === 'transect'
                  ? 'bg-sky-600 text-white'
                  : 'bg-slate-100 text-slate-700 hover:bg-slate-200'
              "
              @click="changeLevel('transect')"
            >
              By transect
            </button>
            <button
              type="button"
              class="px-4 py-2 text-sm font-medium border-l border-slate-300 transition-colors duration-150"
              :class="
                level === 'observation'
                  ? 'bg-sky-600 text-white'
                  : 'bg-slate-100 text-slate-700 hover:bg-slate-200'
              "
              @click="changeLevel('observation')"
            >
              By observation
            </button>
            <button
              type="button"
              class="px-4 py-2 text-sm font-medium border-l border-slate-300 transition-colors duration-150"
              :class="
                level === 'colony'
                  ? 'bg-sky-600 text-white'
                  : 'bg-slate-100 text-slate-700 hover:bg-slate-200'
              "
              @click="changeLevel('colony')"
            >
              By colony
            </button>
          </div>

          <div v-if="drillTransectId" class="mt-2 bg-sky-100 border border-sky-300 rounded px-3 py-2 text-sm flex items-center gap-2">
            <span>Showing only inside transect <b>{{ drillTransectId }}</b></span>
            <button type="button" class="ml-auto px-2 py-0.5 bg-white border border-sky-300 rounded text-xs" @click="clearDrillDown()">Back to all transects</button>
          </div>

          <div class="mt-3 bg-white rounded-lg border border-slate-200 p-3 space-y-3">
            <div class="grid grid-cols-1 md:grid-cols-3 gap-3">
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">Year</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="year in years" :key="year" type="button" class="chip" :class="{ active: selectedYears.includes(year) }" @click="toggleString(selectedYears, year)">{{ year }}</button>
                </div>
              </div>
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">Location</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="location in locations" :key="location" type="button" class="chip" :class="{ active: selectedLocations.includes(location) }" @click="toggleString(selectedLocations, location)">{{ location }}</button>
                </div>
              </div>
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">Depth (m)</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="depth in depths" :key="depth" type="button" class="chip" :class="{ active: selectedDepths.includes(depth) }" @click="toggleString(selectedDepths, depth)">{{ depth }}</button>
                </div>
              </div>
            </div>

            <div class="grid grid-cols-2 md:grid-cols-4 gap-3 items-end">
              <label class="text-xs">Sampler
                <select v-model="selectedSampler" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any sampler</option>
                  <option v-for="sampler in samplers" :key="sampler">{{ sampler }}</option>
                </select>
              </label>
              <label v-if="level === 'transect'" class="text-xs">Status
                <select v-model="selectedStatus" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any</option>
                  <option>Still entering</option>
                  <option>Done</option>
                  <option>Done - needs checking</option>
                </select>
              </label>
              <label v-if="level !== 'transect'" class="text-xs">Category
                <select v-model="selectedCategory" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any category</option>
                  <option v-for="category in categories" :key="category">{{ category }}</option>
                </select>
              </label>
              <label v-if="level !== 'transect'" class="text-xs">Genus
                <select v-model="selectedGenus" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any genus</option>
                  <option v-for="genus in genera" :key="genus">{{ genus }}</option>
                </select>
              </label>
            </div>

            <div v-if="level !== 'transect'" id="wrap-size">
              <div class="text-xs font-medium text-slate-500 mb-1">Size class</div>
              <div class="flex flex-wrap gap-1.5">
                <button v-for="size in ['S', 'M', 'L', 'H']" :key="size" type="button" class="chip" :class="{ active: selectedSizes.includes(size) }" @click="toggleString(selectedSizes, size)">{{ size }}</button>
              </div>
            </div>

            <details class="text-xs text-slate-500">
              <summary class="cursor-pointer">Advanced: exact date range</summary>
              <div class="flex gap-3 mt-2 flex-wrap">
                <label>From<input v-model="fromDate" type="date" class="ml-1 border rounded px-2 py-1"></label>
                <label>To<input v-model="toDate" type="date" class="ml-1 border rounded px-2 py-1"></label>
              </div>
            </details>
          </div>

          <div class="mt-2 flex gap-2 flex-wrap items-center">
            <button
              type="button"
              class="px-4 py-2 rounded text-sm font-medium disabled:cursor-not-allowed"
              :class="canRunQuery ? 'bg-sky-600 text-white' : 'bg-slate-300 text-slate-600'"
              :disabled="!canRunQuery"
              @click="runQuery()"
            >
              Run query
            </button>
            <span v-if="isLoadingRows" class="inline-flex items-center gap-1.5 px-2 py-1 rounded bg-sky-50 text-sky-700 text-xs font-medium">
              <span class="mini-spinner"></span>
              Loading...
            </span>
            <button type="button" class="px-4 py-2 bg-slate-200 rounded text-sm" :disabled="isExporting" @click="exportCurrentCsv()">{{ isExporting ? 'Exporting…' : 'Export CSV' }}</button>
            <button type="button" class="px-4 py-2 bg-slate-100 rounded text-sm text-slate-600" @click="resetFilters()">Reset filters</button>
            <div class="relative">
              <button type="button" class="px-4 py-2 bg-white border border-slate-300 rounded text-sm" @click="showColumnsPanel = !showColumnsPanel">Columns &#9662;</button>
              <div v-if="showColumnsPanel" class="absolute z-30 mt-1 bg-white border border-slate-300 rounded shadow-lg p-2 w-72 max-h-80 overflow-y-auto">
                <div class="text-xs text-slate-500 mb-1">Tick what you want to see in this view</div>
                <div class="space-y-1">
                  <label v-for="column in levelColumns[level]" :key="column" class="flex items-center gap-2 text-xs">
                    <input v-model="hiddenColumns[level]" type="checkbox" :value="column"> {{ column }}
                  </label>
                </div>
                <button type="button" class="mt-2 text-xs text-sky-700 underline" @click="hiddenColumns[level] = []">show all</button>
              </div>
            </div>
            <span class="ml-auto text-sm text-slate-500">{{ totalRows }} record{{ totalRows === 1 ? '' : 's' }}</span>
          </div>
        </section>

        <div class="flex items-center justify-between text-xs text-slate-500 px-1">
          <span>Click a column heading to sort. In "By transect", click a row to see what is inside that transect.</span>
          <span>Page {{ page }} / {{ totalPages }}</span>
        </div>

        <section id="table-wrap" class="bg-white rounded-lg border border-slate-200">
          <div ref="topBarRef" id="top-bar" style="display:none" @mousedown="onTopBarMouseDown">
            <div ref="topThumbRef" id="top-thumb" @mousedown.stop.prevent="onTopThumbMouseDown"></div>
          </div>
          <div ref="tableScrollRef" id="table-scroll" class="overflow-x-auto rounded-lg" @scroll="updateScrollMetrics()">
            <table id="results" class="min-w-full">
              <thead class="bg-slate-50 text-slate-600">
                <tr id="thead-row">
                  <th v-for="column in visibleColumns" :key="column" class="text-left cursor-pointer" :class="{ long: column === 'Joined with' }" @click="applySort(column)">
                    {{ column }}<span v-if="sortState?.column === column">{{ sortState.direction === 'asc' ? ' ↑' : ' ↓' }}</span>
                  </th>
                </tr>
              </thead>
              <tbody id="tbody">
                <tr v-if="!rows.length && isLoadingRows">
                  <td class="px-3 py-4 text-slate-400" :colspan="visibleColumns.length">Loading records...</td>
                </tr>
                <tr v-else-if="!rows.length">
                  <td class="px-3 py-4 text-slate-400" :colspan="visibleColumns.length">No records match the filters.</td>
                </tr>
                <tr v-for="(row, index) in rows" :key="index" :class="[index % 2 ? 'odd bg-slate-50' : 'even', level === 'transect' ? 'drill' : '']" @click="level === 'transect' ? openDrillDown(row) : null">
                  <td v-for="column in visibleColumns" :key="column" :class="{ long: column === 'Joined with' }" :title="formatDisplayValue(row[column])">{{ formatDisplayValue(row[column]) }}</td>
                </tr>
              </tbody>
            </table>
          </div>
          <div ref="scrollHintRef" id="scroll-hint"></div>
        </section>

        <div class="flex items-center justify-between gap-3 flex-wrap text-xs text-slate-500 px-1">
          <div class="flex items-center gap-2">
            <button type="button" class="px-2 py-1 border rounded bg-white disabled:opacity-50" :disabled="page <= 1 || isLoadingRows" @click="changePage(-1)">prev</button>
            <span class="px-3 py-1 border rounded bg-white">{{ page }} / {{ totalPages }}</span>
            <button type="button" class="px-2 py-1 border rounded bg-white disabled:opacity-50" :disabled="page >= totalPages || isLoadingRows" @click="changePage(1)">next</button>
          </div>
        </div>
      </template>
    </main>
  </div>
</template>

<style scoped>
.chip {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  padding: 0.1rem 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 9999px;
  font-size: 0.75rem;
  cursor: pointer;
  background: #fff;
}

.chip.active {
  background: #e0f2fe;
  border-color: #38bdf8;
}

#results td,
#results th {
  padding: 0.2rem 0.5rem;
  white-space: nowrap;
  font-size: 0.8rem;
}

#results td.long,
#results th.long {
  max-width: 15rem;
  overflow: hidden;
  text-overflow: ellipsis;
}

#top-bar {
  height: 10px;
  margin: 6px 8px;
  background: #e2e8f0;
  border-radius: 5px;
  position: relative;
}

#top-thumb {
  position: absolute;
  top: 0;
  bottom: 0;
  background: #94a3b8;
  border-radius: 5px;
}

#table-scroll::-webkit-scrollbar {
  height: 11px;
}

#table-scroll::-webkit-scrollbar-track {
  background: #f1f5f9;
  border-radius: 6px;
}

#table-scroll::-webkit-scrollbar-thumb {
  background: #94a3b8;
  border-radius: 6px;
}

#table-scroll {
  scrollbar-width: thin;
  scrollbar-color: #94a3b8 #f1f5f9;
}

#table-wrap {
  position: relative;
}

.mini-spinner {
  width: 0.8rem;
  height: 0.8rem;
  border: 2px solid #bae6fd;
  border-top-color: #0284c7;
  border-radius: 9999px;
  animation: spin 0.75s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

#scroll-hint {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  width: 3rem;
  pointer-events: none;
  background: linear-gradient(to right, rgba(255, 255, 255, 0), rgba(255, 255, 255, 0.95));
  display: none;
}

#results tbody tr.drill {
  cursor: pointer;
}

#results tbody tr.drill:hover {
  background: #e0f2fe;
}
</style>