<script setup lang="ts">
import type { SortType, TableType } from 'nocodb-sdk'

definePageMeta({
  hideHeader: true,
  hasSidebar: true,
})

type LagoonLevel = 'survey' | 'quadrat' | 'observation'
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

const surveyMeta = ref<TableType | null>(null)
const quadratMeta = ref<TableType | null>(null)
const observationMeta = ref<TableType | null>(null)
const samplerMeta = ref<TableType | null>(null)
const categoryMeta = ref<TableType | null>(null)
const observationTypeMeta = ref<TableType | null>(null)

const level = ref<LagoonLevel>('survey')
const rows = ref<RowRecord[]>([])
const page = ref(1)
const pageSize = ref(100)
const pageInfo = ref<{ totalRows?: number; page?: number; pageSize?: number }>({})
const sortState = ref<{ column: string; direction: 'asc' | 'desc' } | null>(null)
const showColumnsPanel = ref(false)
const lastRequestId = ref(0)
const lastAppliedQueryFingerprint = ref('')

const drillSurveyDate = ref('')
const drillQuadrat = ref<{ date: string; x: string; y: string } | null>(null)

const selectedYears = ref<string[]>([])
const selectedXs = ref<string[]>([])
const selectedYs = ref<string[]>([])
const selectedSizes = ref<string[]>([])
const selectedSampler = ref('')
const selectedCategory = ref('')
const selectedGenus = ref('')
const fromDate = ref('')
const toDate = ref('')

const years = ref<string[]>([])
const xValues = ref<string[]>([])
const yValues = ref<string[]>([])
const samplers = ref<string[]>([])
const categories = ref<string[]>([])
const genera = ref<string[]>([])

const hiddenColumns = reactive<Record<LagoonLevel, string[]>>({
  survey: [],
  quadrat: [],
  observation: [],
})

const tableScrollRef = ref<HTMLElement | null>(null)
const topBarRef = ref<HTMLElement | null>(null)
const topThumbRef = ref<HTMLElement | null>(null)
const scrollHintRef = ref<HTMLElement | null>(null)

let isTopThumbDragging = false
let dragStartX = 0
let dragStartLeft = 0

const levelColumns: Record<LagoonLevel, string[]> = {
  survey: ['Location', 'Site', 'Depth', 'Year', 'Date', 'Samplers', 'Quadrats', 'Observations', 'Colonies counted'],
  quadrat: ['Location', 'Site', 'Depth', 'Year', 'Date', 'X along shore (m)', 'Y seaward (m)', 'Sampler', 'Sand %', 'Rock %', 'GRV %', 'BR %', 'Substrate total %', 'Observations', 'Colonies counted'],
  observation: ['Location', 'Site', 'Depth', 'Year', 'Date', 'X along shore (m)', 'Y seaward (m)', 'Sampler', 'Observation #', 'Category', 'Observation', 'Species', 'Count', 'Size', '% Living', 'Bleached', 'Form', 'Color', 'Algae type', 'Condition'],
}

const defaultSortTitles: Record<LagoonLevel, string[]> = {
  survey: ['Location', 'Depth', 'Date'],
  quadrat: ['Location', 'Depth', 'Date', 'X along shore (m)', 'Y seaward (m)'],
  observation: ['Location', 'Depth', 'Date', 'X along shore (m)', 'Y seaward (m)', 'Observation #'],
}

const visibleColumns = computed(() => levelColumns[level.value].filter((column) => !hiddenColumns[level.value].includes(column)))
const totalRows = computed(() => Number(pageInfo.value.totalRows ?? rows.value.length ?? 0))
const totalPages = computed(() => Math.max(1, Math.ceil(totalRows.value / pageSize.value)))
const activeViewMeta = computed(() => {
  if (level.value === 'survey') return surveyMeta.value
  if (level.value === 'quadrat') return quadratMeta.value
  return observationMeta.value
})
const drillLabel = computed(() => {
  if (drillQuadrat.value) return `${drillQuadrat.value.date} / X ${drillQuadrat.value.x} / Y ${drillQuadrat.value.y}`
  if (drillSurveyDate.value) return `survey ${drillSurveyDate.value}`
  return ''
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

const buildGroup = (field: string, values: string[], op: 'eq' | 'like' = 'eq') => {
  if (!values.length) return ''
  if (op === 'eq' && values.length > 1) {
    return `(${field},in,${values.join(',')})`
  }

  return values.map((value) => `(${field},${op},${value})`).join('~or')
}

const buildDateExact = (field: string, value: string) => {
  if (!value) return ''
  // Legacy parser expects a date sub-operator for Date/DateTime fields.
  return `(${field},eq,exactDate,${value})`
}

const buildDateBound = (field: string, op: 'gte' | 'lte', value: string) => {
  if (!value) return ''
  return `(${field},${op},exactDate,${value})`
}

const buildWhere = () => {
  const parts: string[] = ['(is_test,eq,0)']

  if (drillSurveyDate.value) {
    const dateTitle = findColumnTitle(activeViewMeta.value, 'Date')
    parts.push(buildDateExact(dateTitle, drillSurveyDate.value))
  }

  if (drillQuadrat.value) {
    const xTitle = findColumnTitle(activeViewMeta.value, 'X along shore (m)')
    const yTitle = findColumnTitle(activeViewMeta.value, 'Y seaward (m)')
    parts.push(`(${xTitle},eq,${drillQuadrat.value.x})`)
    parts.push(`(${yTitle},eq,${drillQuadrat.value.y})`)
  }

  if (selectedYears.value.length) parts.push(buildGroup('Year', selectedYears.value))
  if (selectedSampler.value) {
    if (level.value === 'survey') {
      const surveySamplersTitle = findColumnTitle(activeViewMeta.value, 'Samplers')
      parts.push(`(${surveySamplersTitle},like,${selectedSampler.value})`)
    } else {
      const samplerTitle = findColumnTitle(activeViewMeta.value, 'Sampler')
      parts.push(`(${samplerTitle},eq,${selectedSampler.value})`)
    }
  }

  if (level.value !== 'survey') {
    if (selectedXs.value.length) parts.push(buildGroup('X along shore (m)', selectedXs.value))
    if (selectedYs.value.length) parts.push(buildGroup('Y seaward (m)', selectedYs.value))
  }

  if (level.value === 'observation') {
    if (selectedCategory.value) parts.push(`(Category,eq,${selectedCategory.value})`)
    if (selectedGenus.value) parts.push(`(Observation,eq,${selectedGenus.value})`)
    if (selectedSizes.value.length) parts.push(buildGroup('Size', selectedSizes.value))
  }

  if (fromDate.value) parts.push(buildDateBound('Date', 'gte', fromDate.value))
  if (toDate.value) parts.push(buildDateBound('Date', 'lte', toDate.value))

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
  if (!surveyMeta.value || !quadratMeta.value || !categoryMeta.value || !observationTypeMeta.value) return

  const surveyRows = await listAllRows(surveyMeta.value, { where: '(is_test,eq,0)' }, 1000)
  years.value = [...new Set(surveyRows.map((row) => String(row['Year'] ?? '').trim()).filter(Boolean))].sort((a, b) => Number(a) - Number(b))

  const quadratRows = await listAllRows(quadratMeta.value, { where: '(is_test,eq,0)' }, 1000)
  xValues.value = [...new Set(quadratRows.map((row) => String(row['X along shore (m)'] ?? '').trim()).filter(Boolean))].sort((a, b) => Number(a) - Number(b))
  yValues.value = [...new Set(quadratRows.map((row) => String(row['Y seaward (m)'] ?? '').trim()).filter(Boolean))].sort((a, b) => Number(a) - Number(b))

  if (samplerMeta.value) {
    const samplerRows = await listAllRows(samplerMeta.value, {}, 1000)
    samplers.value = [...new Set(samplerRows.map((row) => String(row.sampler_name ?? row.name ?? '').trim()).filter(Boolean))].sort()
  } else {
    samplers.value = [...new Set(quadratRows.map((row) => String(row['Sampler'] ?? '').trim()).filter(Boolean))].sort()
  }

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
  selectedXs.value = []
  selectedYs.value = []
  selectedSizes.value = []
  selectedSampler.value = ''
  selectedCategory.value = ''
  selectedGenus.value = ''
  fromDate.value = ''
  toDate.value = ''
  drillSurveyDate.value = ''
  drillQuadrat.value = null
  level.value = 'survey'
  resetSort()
  page.value = 1
  await loadRows()
}

const openDrillDown = async (row: RowRecord) => {
  if (level.value === 'survey') {
    const date = String(getRowValueByTitle(row, 'Date') ?? '').trim()
    if (!date) return

    drillSurveyDate.value = date
    drillQuadrat.value = null
    level.value = 'quadrat'
    selectedXs.value = []
    selectedYs.value = []
    selectedCategory.value = ''
    selectedGenus.value = ''
    selectedSizes.value = []
    page.value = 1
    resetSort()
    await loadRows()
    return
  }

  if (level.value === 'quadrat') {
    const date = String(getRowValueByTitle(row, 'Date') ?? '').trim()
    const x = String(getRowValueByTitle(row, 'X along shore (m)') ?? '').trim()
    const y = String(getRowValueByTitle(row, 'Y seaward (m)') ?? '').trim()
    if (!date || !x || !y) return

    drillSurveyDate.value = date
    drillQuadrat.value = { date, x, y }
    level.value = 'observation'
    selectedCategory.value = ''
    selectedGenus.value = ''
    selectedSizes.value = []
    page.value = 1
    resetSort()
    await loadRows()
  }
}

const clearDrillDown = async () => {
  drillSurveyDate.value = ''
  drillQuadrat.value = null
  level.value = 'survey'
  page.value = 1
  resetSort()
  await loadRows()
}

const changeLevel = async (nextLevel: LagoonLevel) => {
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
    downloadCsv(`nmp-lagoon-${level.value}-${fileDate}.csv`, columns, responseRows)
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

    const surveyTable = tableByNames(tables, ['v_lagoon_surveys'])
    const quadratTable = tableByNames(tables, ['v_lagoon_quadrats'])
    const observationTable = tableByNames(tables, ['v_lagoon_observations'])
    const samplersTable = tableByNames(tables, ['samplers'])
    const categoriesTable = tableByNames(tables, ['observation_categories'])
    const observationTypesTable = tableByNames(tables, ['observation_types'])

    if (!surveyTable || !quadratTable || !observationTable) {
      errorMsg.value = 'Required lagoon report views are missing in this base.'
      return
    }

    surveyMeta.value = (await getMeta(base.value.id, surveyTable.id as string)) as TableType
    quadratMeta.value = (await getMeta(base.value.id, quadratTable.id as string)) as TableType
    observationMeta.value = (await getMeta(base.value.id, observationTable.id as string)) as TableType
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
    <header class="bg-slate-800 text-white px-6 py-3 flex items-center justify-between border-b-4 border-teal-500">
      <div>
        <div class="font-semibold text-lg">NMP - Lagoon Query & Browse</div>
        <div class="text-xs opacity-80">Read-only lagoon reports with server-side filtering, pagination, and CSV download</div>
      </div>
    </header>

    <main class="p-4 space-y-4">
      <section v-if="errorMsg" class="rounded-lg border border-rose-300 bg-rose-50 text-rose-700 p-3 text-sm">
        {{ errorMsg }}
      </section>

      <section v-if="isBusy" class="rounded-lg border border-slate-200 bg-white p-4 text-sm text-slate-600">Loading lagoon query interface...</section>

      <template v-else>
        <section class="bg-white border border-slate-200 rounded-lg shadow-sm p-3">
          <div class="mt-4 inline-flex rounded-lg border border-slate-400 overflow-hidden bg-white shadow-sm" id="level-toggle">
            <button
              type="button"
              class="px-4 py-2 text-sm font-medium transition-colors duration-150"
              :class="
                level === 'survey'
                  ? 'bg-teal-600 text-white'
                  : 'bg-slate-100 text-slate-700 hover:bg-slate-200'
              "
              @click="changeLevel('survey')"
            >
              By survey
            </button>
            <button
              type="button"
              class="px-4 py-2 text-sm font-medium border-l border-slate-300 transition-colors duration-150"
              :class="
                level === 'quadrat'
                  ? 'bg-teal-600 text-white'
                  : 'bg-slate-100 text-slate-700 hover:bg-slate-200'
              "
              @click="changeLevel('quadrat')"
            >
              By quadrat
            </button>
            <button
              type="button"
              class="px-4 py-2 text-sm font-medium border-l border-slate-300 transition-colors duration-150"
              :class="
                level === 'observation'
                  ? 'bg-teal-600 text-white'
                  : 'bg-slate-100 text-slate-700 hover:bg-slate-200'
              "
              @click="changeLevel('observation')"
            >
              By observation
            </button>
          </div>

          <div class="mt-3 bg-white rounded-lg border border-slate-200 p-3 space-y-3">
            <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">Year</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="year in years" :key="year" type="button" class="chip" :class="{ active: selectedYears.includes(year) }" @click="toggleString(selectedYears, year)">{{ year }}</button>
                </div>
              </div>
              <label class="text-xs">Sampler
                <select v-model="selectedSampler" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any sampler</option>
                  <option v-for="sampler in samplers" :key="sampler">{{ sampler }}</option>
                </select>
              </label>
            </div>

            <div v-if="level !== 'survey'" class="space-y-2">
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">X along shore (m)</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="x in xValues" :key="x" type="button" class="chip" :class="{ active: selectedXs.includes(x) }" @click="toggleString(selectedXs, x)">{{ x }}</button>
                </div>
              </div>
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">Y seaward (m)</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="y in yValues" :key="y" type="button" class="chip" :class="{ active: selectedYs.includes(y) }" @click="toggleString(selectedYs, y)">{{ y }}</button>
                </div>
              </div>
            </div>

            <div v-if="level === 'observation'" class="grid grid-cols-2 md:grid-cols-3 gap-3 items-end">
              <label class="text-xs">Category
                <select v-model="selectedCategory" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any category</option>
                  <option v-for="category in categories" :key="category">{{ category }}</option>
                </select>
              </label>
              <label class="text-xs">Observation
                <select v-model="selectedGenus" class="mt-1 w-full border rounded px-2 py-1 text-sm">
                  <option value="">Any</option>
                  <option v-for="genus in genera" :key="genus">{{ genus }}</option>
                </select>
              </label>
              <div>
                <div class="text-xs font-medium text-slate-500 mb-1">Size class</div>
                <div class="flex flex-wrap gap-1.5">
                  <button v-for="size in ['S', 'M', 'L', 'H']" :key="size" type="button" class="chip" :class="{ active: selectedSizes.includes(size) }" @click="toggleString(selectedSizes, size)">{{ size }}</button>
                </div>
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
              :class="canRunQuery ? 'bg-teal-600 text-white' : 'bg-slate-300 text-slate-600'"
              :disabled="!canRunQuery"
              @click="runQuery()"
            >
              Run query
            </button>
            <span v-if="isLoadingRows" class="inline-flex items-center gap-1.5 px-2 py-1 rounded bg-teal-50 text-teal-700 text-xs font-medium">
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
                <button type="button" class="mt-2 text-xs text-teal-700 underline" @click="hiddenColumns[level] = []">show all</button>
              </div>
            </div>
            <span class="ml-auto text-sm text-slate-500">{{ totalRows }} record{{ totalRows === 1 ? '' : 's' }}</span>
          </div>

          <div v-if="drillLabel" class="mt-3 bg-teal-100 border border-teal-300 rounded px-3 py-2 text-sm flex items-center gap-2">
            <span>Showing only <b>{{ drillLabel }}</b></span>
            <button type="button" class="ml-auto px-2 py-0.5 bg-white border border-teal-300 rounded text-xs" @click="clearDrillDown()">Show everything again</button>
          </div>
        </section>

        <div class="flex items-center justify-between text-xs text-slate-500 px-1">
          <span>Click a column heading to sort. In survey/quadrat levels, click a row to drill deeper.</span>
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
                  <th v-for="column in visibleColumns" :key="column" class="text-left cursor-pointer" @click="applySort(column)">
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
                <tr
                  v-for="(row, index) in rows"
                  :key="index"
                  :class="[index % 2 ? 'odd bg-slate-50' : 'even', level !== 'observation' ? 'drill' : '']"
                  @click="level !== 'observation' ? openDrillDown(row) : null"
                >
                  <td v-for="column in visibleColumns" :key="column" :title="formatDisplayValue(row[column])">{{ formatDisplayValue(row[column]) }}</td>
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
  background: #ccfbf1;
  border-color: #2dd4bf;
}

#results td,
#results th {
  padding: 0.2rem 0.5rem;
  white-space: nowrap;
  font-size: 0.8rem;
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
  border: 2px solid #99f6e4;
  border-top-color: #0f766e;
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
  background: #ccfbf1;
}
</style>
