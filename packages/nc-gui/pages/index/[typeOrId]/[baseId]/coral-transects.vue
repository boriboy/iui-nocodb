<script setup lang="ts">
import type { ColumnType, TableType } from 'nocodb-sdk'

definePageMeta({
  hideHeader: true,
  hasSidebar: true,
})

type RowRecord = Record<string, any>

const route = useRoute()
const { $api } = useNuxtApp()
const { base } = storeToRefs(useBase())
const { baseTables } = storeToRefs(useTablesStore())
const { loadProjectTables } = useTablesStore()
const { getMeta } = useMetas()

const isBusy = ref(true)
const errorMsg = ref('')

const transectMeta = ref<TableType | null>(null)
const observationMeta = ref<TableType | null>(null)
const siteMeta = ref<TableType | null>(null)
const samplerMeta = ref<TableType | null>(null)
const categoryMeta = ref<TableType | null>(null)
const observationTypeMeta = ref<TableType | null>(null)

const tab = computed(() => ((route.query.tab as string) === 'wip' ? 'wip' : 'entry'))

const currentTransect = ref<RowRecord | null>(null)
const isEditorOpen = ref(false)
const isHydratingEditor = ref(false)
const lastPersistedSignature = ref('')
const wipTransects = ref<RowRecord[]>([])
const observations = ref<RowRecord[]>([])

const sites = ref<RowRecord[]>([])
const samplers = ref<RowRecord[]>([])
const categories = ref<RowRecord[]>([])
const observationTypes = ref<RowRecord[]>([])

const autosaveTimer = ref<ReturnType<typeof setTimeout> | null>(null)

const headerModel = reactive({
  survey_date: '',
  site_id: '',
  depth_m: '',
  sampler_id: '',
  notes: '',
})

const rowsModel = ref<
  Array<{
    observation_id?: string
    distance_from_m: number
    distance_to_m: string
    category_id: string
    observation_type_id: string
    species: string
    size_class: string
    pct_living: string
    form_id: string
    color_id: string
    algae_type_id: string
    condition_id: string
    bleached: boolean
    remark: string
  }>
>([])

const getColumn = (meta: TableType | null, title: string) => meta?.columns?.find((c) => c.title === title)

const getPkTitle = (meta: TableType | null) => (meta?.columns?.find((c) => c.pk)?.title as string | undefined) ?? 'id'

const tableByNames = (tables: TableType[], names: string[]) => {
  return (
    tables.find((t) => names.includes((t.table_name || '').toLowerCase())) ||
    tables.find((t) => names.includes((t.title || '').toLowerCase())) ||
    null
  )
}

const getStatusOptions = () => {
  const statusCol = transectMeta.value?.columns?.find((c: any) => c?.title === 'status' || c?.column_name === 'status') as any
  const raw = statusCol?.colOptions?.options
  if (!raw) return [] as string[]

  if (Array.isArray(raw)) {
    return raw
      .map((opt: any) => String(opt?.title ?? opt?.label ?? opt?.value ?? ''))
      .map((v: string) => v.trim())
      .filter(Boolean)
  }

  if (typeof raw === 'string') {
    try {
      const parsed = JSON.parse(raw)
      if (Array.isArray(parsed)) {
        return parsed
          .map((opt: any) => String(opt?.title ?? opt?.label ?? opt?.value ?? ''))
          .map((v: string) => v.trim())
          .filter(Boolean)
      }
    } catch {
      return []
    }
  }

  return [] as string[]
}

const canUseStatusValue = (value: string) => {
  const options = getStatusOptions().map((v) => v.toUpperCase())
  return options.includes(value.toUpperCase())
}

const draftStatusValue = () => {
  if (canUseStatusValue('WIP')) return 'WIP'
  if (canUseStatusValue('INCOMPLETE')) return 'INCOMPLETE'
  return null
}

const isReportCompleted = (row: RowRecord) => {
  if (row?.submitted_at) return true
  return String(row?.status || '').toUpperCase() === 'COMPLETE'
}

const listTableRows = async (meta: TableType, where = '', limit = 500) => {
  const baseId = base.value?.id
  if (!baseId) return []
  const listRes = await $api.dbTableRow.list('noco', baseId, meta.id as string, {
    where,
    limit,
    offset: 0,
  })
  return listRes?.list ?? []
}

const computeTotalLength = () => {
  if (!rowsModel.value.length) return 0
  const first = rowsModel.value[0]
  const maxTo = rowsModel.value.reduce((acc, row) => {
    const toVal = Number(row.distance_to_m)
    return Number.isFinite(toVal) ? Math.max(acc, toVal) : acc
  }, Number(first.distance_from_m || 0))
  return Math.max(0, maxTo - Number(first.distance_from_m || 0))
}

const syncHeaderFromCurrent = () => {
  if (!currentTransect.value) return
  headerModel.survey_date = (currentTransect.value.survey_date || '').toString().slice(0, 10)
  headerModel.site_id = currentTransect.value.site_id ? String(currentTransect.value.site_id) : ''
  headerModel.depth_m = currentTransect.value.depth_m != null ? String(currentTransect.value.depth_m) : ''
  headerModel.sampler_id = currentTransect.value.sampler_id != null ? String(currentTransect.value.sampler_id) : ''
  headerModel.notes = currentTransect.value.notes || ''
}

const toObservationUi = (row: RowRecord) => ({
  observation_id: row.observation_id,
  distance_from_m: Number(row.distance_from_m || 0),
  distance_to_m: row.distance_to_m != null ? String(row.distance_to_m) : '',
  category_id: row.category_id != null ? String(row.category_id) : '',
  observation_type_id: row.observation_type_id != null ? String(row.observation_type_id) : '',
  species: row.species || '',
  size_class: row.size_class || '',
  pct_living: row.pct_living != null ? String(row.pct_living) : '',
  form_id: row.form_id != null ? String(row.form_id) : '',
  color_id: row.color_id != null ? String(row.color_id) : '',
  algae_type_id: row.algae_type_id != null ? String(row.algae_type_id) : '',
  condition_id: row.condition_id != null ? String(row.condition_id) : '',
  bleached: !!row.bleached,
  remark: row.remark || '',
})

const currentDraftSignature = () => {
  if (!currentTransect.value) return ''

  return JSON.stringify({
    header: {
      survey_date: headerModel.survey_date || '',
      site_id: headerModel.site_id || '',
      depth_m: headerModel.depth_m || '',
      sampler_id: headerModel.sampler_id || '',
      notes: headerModel.notes || '',
    },
    rows: rowsModel.value.map((row) => ({
      observation_id: row.observation_id || '',
      distance_from_m: row.distance_from_m || 0,
      distance_to_m: row.distance_to_m || '',
      category_id: row.category_id || '',
      observation_type_id: row.observation_type_id || '',
      species: row.species || '',
      size_class: row.size_class || '',
      pct_living: row.pct_living || '',
      form_id: row.form_id || '',
      color_id: row.color_id || '',
      algae_type_id: row.algae_type_id || '',
      condition_id: row.condition_id || '',
      bleached: !!row.bleached,
      remark: row.remark || '',
    })),
  })
}

const hydrateRowsForCurrent = async () => {
  if (!observationMeta.value || !currentTransect.value) return
  const where = `(transect_id,eq,${currentTransect.value.transect_id})`
  const list = await listTableRows(observationMeta.value, where, 2000)
  observations.value = list
  rowsModel.value = list
    .sort((a, b) => Number(a.distance_from_m || 0) - Number(b.distance_from_m || 0))
    .map(toObservationUi)
}

const upsertTransectHeader = async () => {
  if (!transectMeta.value || !currentTransect.value || !base.value?.id) return

  const transectPk = getPkTitle(transectMeta.value)
  const transectId = currentTransect.value[transectPk] ?? currentTransect.value.transect_id

  const payload: RowRecord = {
    survey_date: headerModel.survey_date || null,
    site_id: headerModel.site_id || null,
    depth_m: headerModel.depth_m ? Number(headerModel.depth_m) : null,
    sampler_id: headerModel.sampler_id ? Number(headerModel.sampler_id) : null,
    notes: headerModel.notes || null,
    total_length_m: computeTotalLength(),
    last_saved_at: new Date().toISOString(),
  }

  if (!isReportCompleted(currentTransect.value)) {
    const nextDraftStatus = draftStatusValue()
    if (nextDraftStatus) {
      payload.status = nextDraftStatus
    }
  }

  const updated = await $api.dbTableRow.update('noco', base.value.id, transectMeta.value.id as string, encodeURIComponent(String(transectId)), payload)
  currentTransect.value = updated
}

const persistObservationRow = async (row: (typeof rowsModel.value)[number], index: number) => {
  if (!observationMeta.value || !currentTransect.value || !base.value?.id) return

  const transectId = currentTransect.value.transect_id
  const payload: RowRecord = {
    transect_id: transectId,
    distance_from_m: Number(row.distance_from_m || 0),
    distance_to_m: row.distance_to_m ? Number(row.distance_to_m) : null,
    length_cm: row.distance_to_m ? Number(((Number(row.distance_to_m) - Number(row.distance_from_m || 0)) * 100).toFixed(1)) : null,
    category_id: row.category_id ? Number(row.category_id) : null,
    observation_type_id: row.observation_type_id ? Number(row.observation_type_id) : null,
    species: row.species || null,
    size_class: row.size_class || null,
    pct_living: row.pct_living ? Number(row.pct_living) : null,
    form_id: row.form_id ? Number(row.form_id) : null,
    color_id: row.color_id ? Number(row.color_id) : null,
    algae_type_id: row.algae_type_id ? Number(row.algae_type_id) : null,
    condition_id: row.condition_id ? Number(row.condition_id) : null,
    bleached: !!row.bleached,
    remark: row.remark || null,
    data_source: 'ui_entry',
    qc_flag: 0,
  }

  if (row.observation_id) {
    const updated = await $api.dbTableRow.update('noco', base.value.id, observationMeta.value.id as string, encodeURIComponent(String(row.observation_id)), payload)
    row.observation_id = String(updated.observation_id)
  } else {
    const created = await $api.dbTableRow.create('noco', base.value.id, observationMeta.value.id as string, payload)
    row.observation_id = String(created.observation_id)
  }
}

const scheduleAutosave = () => {
  if (autosaveTimer.value) clearTimeout(autosaveTimer.value)
  autosaveTimer.value = setTimeout(async () => {
    try {
      if (!isEditorOpen.value || !currentTransect.value || isHydratingEditor.value) return

      const signatureBeforeSave = currentDraftSignature()
      if (!signatureBeforeSave || signatureBeforeSave === lastPersistedSignature.value) return

      await upsertTransectHeader()
      for (let i = 0; i < rowsModel.value.length; i++) {
        await persistObservationRow(rowsModel.value[i], i)
      }

      lastPersistedSignature.value = currentDraftSignature()
    } catch (e: any) {
      message.error(`Autosave failed: ${await extractSdkResponseErrorMsg(e)}`)
    }
  }, 350)
}

const addRow = () => {
  const last = rowsModel.value[rowsModel.value.length - 1]
  const from = last?.distance_to_m ? Number(last.distance_to_m) : Number(last?.distance_from_m || 0)
  rowsModel.value.push({
    distance_from_m: Number(from.toFixed(2)),
    distance_to_m: '',
    category_id: '',
    observation_type_id: '',
    species: '',
    size_class: '',
    pct_living: '',
    form_id: '',
    color_id: '',
    algae_type_id: '',
    condition_id: '',
    bleached: false,
    remark: '',
  })
  scheduleAutosave()
}

const removeRow = async (idx: number) => {
  const row = rowsModel.value[idx]
  if (!row) return
  if (row.observation_id && observationMeta.value && base.value?.id) {
    await $api.dbTableRow.delete('noco', base.value.id, observationMeta.value.id as string, encodeURIComponent(String(row.observation_id)))
  }
  rowsModel.value.splice(idx, 1)
  scheduleAutosave()
}

const loadWips = async () => {
  if (!transectMeta.value) return
  const list = await listTableRows(transectMeta.value, '', 1000)
  wipTransects.value = list
    .filter((row) => !isReportCompleted(row as RowRecord))
    .sort((a, b) => new Date((b as any).last_saved_at || (b as any).created_at || 0).getTime() - new Date((a as any).last_saved_at || (a as any).created_at || 0).getTime())
}

const startNewTransect = async () => {
  if (!transectMeta.value || !base.value?.id) return

  const today = new Date().toISOString().slice(0, 10)
  const payload: RowRecord = {
    survey_date: today,
    survey_method: 'LINE_TRANSECT',
    total_length_m: 0,
    last_saved_at: new Date().toISOString(),
  }

  const nextDraftStatus = draftStatusValue()
  if (nextDraftStatus) {
    payload.status = nextDraftStatus
  }

  const created = await $api.dbTableRow.create('noco', base.value.id, transectMeta.value.id as string, payload)

  isHydratingEditor.value = true
  currentTransect.value = created
  isEditorOpen.value = true
  syncHeaderFromCurrent()
  rowsModel.value = []
  addRow()
  lastPersistedSignature.value = currentDraftSignature()
  isHydratingEditor.value = false
  await loadWips()
}

const resumeWip = async (transect: RowRecord) => {
  isHydratingEditor.value = true
  currentTransect.value = transect
  isEditorOpen.value = true
  syncHeaderFromCurrent()
  await hydrateRowsForCurrent()
  if (!rowsModel.value.length) addRow()
  lastPersistedSignature.value = currentDraftSignature()
  isHydratingEditor.value = false
}

const closeEditor = () => {
  isEditorOpen.value = false
  currentTransect.value = null
  rowsModel.value = []
  lastPersistedSignature.value = ''
}

const deleteWip = async (transect: RowRecord) => {
  if (!transectMeta.value || !observationMeta.value || !base.value?.id) return
  if (isReportCompleted(transect)) {
    message.warning('Only WIP reports can be deleted.')
    return
  }

  Modal.confirm({
    title: `Delete WIP transect #${transect.transect_id}?`,
    content: 'This removes only data belonging to this report.',
    okType: 'danger',
    async onOk() {
      const childRows = await listTableRows(observationMeta.value as TableType, `(transect_id,eq,${transect.transect_id})`, 2000)
      for (const row of childRows) {
        await $api.dbTableRow.delete('noco', base.value!.id as string, observationMeta.value!.id as string, encodeURIComponent(String(row.observation_id)))
      }

      await $api.dbTableRow.delete('noco', base.value!.id as string, transectMeta.value!.id as string, encodeURIComponent(String(transect.transect_id)))

      if (currentTransect.value?.transect_id === transect.transect_id) {
        closeEditor()
      }

      await loadWips()
    },
  })
}

const markComplete = async () => {
  if (!currentTransect.value || !transectMeta.value || !base.value?.id) return
  await upsertTransectHeader()
  const total = computeTotalLength()
  const payload: RowRecord = {
    submitted_at: new Date().toISOString(),
    last_saved_at: new Date().toISOString(),
  }

  if (canUseStatusValue('COMPLETE')) {
    payload.status = 'COMPLETE'
  }

  const updated = await $api.dbTableRow.update('noco', base.value.id, transectMeta.value.id as string, encodeURIComponent(String(currentTransect.value.transect_id)), payload)
  currentTransect.value = updated
  message.success('Transect status updated')
  await loadWips()
}

const init = async () => {
  try {
    isBusy.value = true
    errorMsg.value = ''

    if (!base.value?.id) return

    await loadProjectTables(base.value.id)
    const tables = (baseTables.value.get(base.value.id) || []) as TableType[]

    const transectTable = tableByNames(tables, ['coral_transects'])
    const observationTable = tableByNames(tables, ['coral_transect_observations'])
    const sitesTable = tableByNames(tables, ['sites'])
    const samplersTable = tableByNames(tables, ['samplers'])
    const categoriesTable = tableByNames(tables, ['observation_categories'])
    const observationTypesTable = tableByNames(tables, ['observation_types'])

    if (!transectTable || !observationTable) {
      errorMsg.value = 'Required coral transect tables are missing in this base.'
      return
    }

    transectMeta.value = (await getMeta(base.value.id, transectTable.id as string)) as TableType
    observationMeta.value = (await getMeta(base.value.id, observationTable.id as string)) as TableType
    siteMeta.value = sitesTable ? ((await getMeta(base.value.id, sitesTable.id as string)) as TableType) : null
    samplerMeta.value = samplersTable ? ((await getMeta(base.value.id, samplersTable.id as string)) as TableType) : null
    categoryMeta.value = categoriesTable ? ((await getMeta(base.value.id, categoriesTable.id as string)) as TableType) : null
    observationTypeMeta.value = observationTypesTable ? ((await getMeta(base.value.id, observationTypesTable.id as string)) as TableType) : null

    sites.value = siteMeta.value ? await listTableRows(siteMeta.value) : []
    samplers.value = samplerMeta.value ? await listTableRows(samplerMeta.value) : []
    categories.value = categoryMeta.value ? await listTableRows(categoryMeta.value) : []
    observationTypes.value = observationTypeMeta.value ? await listTableRows(observationTypeMeta.value, '', 4000) : []

    await loadWips()

    if (tab.value === 'wip') {
      return
    }

    if (wipTransects.value.length) {
      await resumeWip(wipTransects.value[0])
    } else {
      await startNewTransect()
    }
  } catch (e: any) {
    errorMsg.value = await extractSdkResponseErrorMsg(e)
  } finally {
    isBusy.value = false
  }
}

watch(
  () => ({ ...headerModel }),
  () => {
    if (!currentTransect.value || !isEditorOpen.value || isHydratingEditor.value) return
    scheduleAutosave()
  },
  { deep: true },
)

watch(
  rowsModel,
  () => {
    if (!currentTransect.value || !isEditorOpen.value || isHydratingEditor.value) return
    scheduleAutosave()
  },
  { deep: true },
)

onBeforeUnmount(() => {
  if (autosaveTimer.value) clearTimeout(autosaveTimer.value)
})

onMounted(init)
</script>

<template>
  <div class="h-full overflow-auto bg-slate-100 text-slate-800">
    <header class="bg-blue-900 text-white px-6 py-3 flex items-center justify-between border-b-4 border-blue-500">
      <div>
        <div class="font-semibold text-lg">NMP - Coral Transect Data Entry</div>
        <div class="text-xs opacity-80">Line transect method - 10 m tape, 1 cm precision</div>
      </div>
      <div class="text-xs opacity-80">Live DB mode</div>
    </header>

    <main class="p-4 space-y-4">
      <section v-if="errorMsg" class="rounded-lg border border-rose-300 bg-rose-50 text-rose-700 p-3 text-sm">
        {{ errorMsg }}
      </section>

      <section v-if="isBusy" class="rounded-lg border border-slate-200 bg-white p-4 text-sm text-slate-600">Loading coral transect interface...</section>

      <template v-else>
        <section class="bg-white border border-slate-200 rounded-lg shadow-sm p-4">
          <div class="flex items-center justify-between mb-3">
            <div class="font-semibold text-blue-900">Coral transect reports</div>
            <div class="flex items-center gap-2">
              <NcButton type="secondary" size="small" @click="loadWips()">Refresh WIP list</NcButton>
              <NcButton type="primary" size="small" @click="startNewTransect()">+ New report</NcButton>
            </div>
          </div>

          <div class="overflow-auto rounded border border-slate-200">
            <table class="w-full text-xs">
              <thead class="bg-slate-50 text-slate-600">
                <tr>
                  <th class="text-left px-3 py-2">Transect #</th>
                  <th class="text-left px-3 py-2">Date</th>
                  <th class="text-left px-3 py-2">Site</th>
                  <th class="text-left px-3 py-2">Length</th>
                  <th class="text-left px-3 py-2">Status</th>
                  <th class="text-left px-3 py-2">Last saved</th>
                  <th class="text-right px-3 py-2">Actions</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="t in wipTransects" :key="t.transect_id" class="border-t border-slate-100">
                  <td class="px-3 py-2">{{ t.transect_id }}</td>
                  <td class="px-3 py-2">{{ (t.survey_date || '').toString().slice(0, 10) }}</td>
                  <td class="px-3 py-2">{{ t.site_id || '-' }}</td>
                  <td class="px-3 py-2">{{ t.total_length_m ?? 0 }} m</td>
                  <td class="px-3 py-2">{{ t.status }}</td>
                  <td class="px-3 py-2">{{ t.last_saved_at ? new Date(t.last_saved_at).toLocaleString() : '-' }}</td>
                  <td class="px-3 py-2 text-right">
                    <div class="inline-flex gap-2">
                      <NcButton size="xsmall" type="primary" @click="resumeWip(t)">Resume</NcButton>
                      <NcButton size="xsmall" type="secondary" @click="deleteWip(t)">Delete</NcButton>
                    </div>
                  </td>
                </tr>
                <tr v-if="!wipTransects.length">
                  <td colspan="7" class="px-3 py-4 text-slate-500">No WIP transects found.</td>
                </tr>
              </tbody>
            </table>
          </div>
        </section>

        <Teleport to="body">
          <div v-if="isEditorOpen && currentTransect" class="fixed inset-0 flex items-center justify-center bg-slate-900/50 px-4" style="z-index: 2000">
            <div class="w-full max-w-7xl max-h-[92vh] overflow-auto rounded-lg bg-white shadow-2xl">
            <div class="sticky top-0 z-10 border-b border-slate-200 bg-white px-4 py-3 flex items-center justify-between">
              <div>
                <div class="text-sm font-semibold text-blue-900">WIP Transect Report #{{ currentTransect.transect_id }}</div>
                <div class="text-xs text-slate-500">All edits are autosaved while this modal is open.</div>
              </div>
              <NcButton type="secondary" size="small" @click="closeEditor()">Pause / close</NcButton>
            </div>

            <div class="p-4 grid grid-cols-12 gap-3 items-end border-b border-slate-200">
              <div class="col-span-2">
                <div class="text-[11px] text-slate-600 mb-1">Date</div>
                <input v-model="headerModel.survey_date" type="date" class="w-full border rounded px-2 py-1 text-xs" />
              </div>
              <div class="col-span-3">
                <div class="text-[11px] text-slate-600 mb-1">Site</div>
                <select v-model="headerModel.site_id" class="w-full border rounded px-2 py-1 text-xs">
                  <option value="">- pick -</option>
                  <option v-for="s in sites" :key="s.site_id" :value="String(s.site_id)">{{ s.site_id }}</option>
                </select>
              </div>
              <div class="col-span-2">
                <div class="text-[11px] text-slate-600 mb-1">Depth (m)</div>
                <input v-model="headerModel.depth_m" type="number" class="w-full border rounded px-2 py-1 text-xs" />
              </div>
              <div class="col-span-3">
                <div class="text-[11px] text-slate-600 mb-1">Sampler</div>
                <select v-model="headerModel.sampler_id" class="w-full border rounded px-2 py-1 text-xs">
                  <option value="">- pick -</option>
                  <option v-for="s in samplers" :key="s.sampler_id" :value="String(s.sampler_id)">{{ s.sampler_name }}</option>
                </select>
              </div>
              <div class="col-span-2">
                <div class="text-[11px] text-slate-600 mb-1">Total length</div>
                <div class="w-full border rounded px-2 py-1 text-xs bg-slate-50">{{ computeTotalLength().toFixed(2) }} m</div>
              </div>
              <div class="col-span-12">
                <div class="text-[11px] text-slate-600 mb-1">Transect notes</div>
                <input v-model="headerModel.notes" class="w-full border rounded px-2 py-1 text-xs" placeholder="Optional notes" />
              </div>
            </div>

            <section class="p-4">
              <div class="flex items-center justify-between mb-3">
                <div class="font-semibold text-blue-900">Observations along the tape</div>
                <div class="text-xs text-slate-500">{{ rowsModel.length }} rows</div>
              </div>

              <div class="space-y-2">
                <div v-for="(row, idx) in rowsModel" :key="row.observation_id || `new-${idx}`" class="rounded-lg border border-amber-300 p-3 bg-white">
                  <div class="grid grid-cols-12 gap-2 items-end">
                    <div class="col-span-1 text-xs font-semibold">#{{ idx + 1 }}</div>
                    <div class="col-span-2">
                      <div class="text-[10px] text-slate-500 mb-1">From (m)</div>
                      <input v-model.number="row.distance_from_m" type="number" step="0.01" class="w-full border rounded px-2 py-1 text-xs" />
                    </div>
                    <div class="col-span-2">
                      <div class="text-[10px] text-slate-500 mb-1">To (m)</div>
                      <input v-model="row.distance_to_m" type="number" step="0.01" class="w-full border rounded px-2 py-1 text-xs" />
                    </div>
                    <div class="col-span-2">
                      <div class="text-[10px] text-slate-500 mb-1">Category</div>
                      <select v-model="row.category_id" class="w-full border rounded px-2 py-1 text-xs">
                        <option value="">- pick -</option>
                        <option v-for="c in categories" :key="c.category_id" :value="String(c.category_id)">{{ c.category_name }}</option>
                      </select>
                    </div>
                    <div class="col-span-3">
                      <div class="text-[10px] text-slate-500 mb-1">Observation</div>
                      <select v-model="row.observation_type_id" class="w-full border rounded px-2 py-1 text-xs">
                        <option value="">- pick -</option>
                        <option
                          v-for="o in observationTypes.filter((ot) => !row.category_id || String(ot.category_id) === row.category_id)"
                          :key="o.type_id"
                          :value="String(o.type_id)"
                        >
                          {{ o.observation_name }}
                        </option>
                      </select>
                    </div>
                    <div class="col-span-2 flex justify-end gap-2">
                      <NcButton size="xsmall" type="secondary" @click="removeRow(idx)">Delete</NcButton>
                    </div>

                    <div class="col-span-2">
                      <div class="text-[10px] text-slate-500 mb-1">Species</div>
                      <input v-model="row.species" class="w-full border rounded px-2 py-1 text-xs" />
                    </div>
                    <div class="col-span-1">
                      <div class="text-[10px] text-slate-500 mb-1">Size</div>
                      <select v-model="row.size_class" class="w-full border rounded px-1 py-1 text-xs">
                        <option value=""></option>
                        <option value="S">S</option>
                        <option value="M">M</option>
                        <option value="L">L</option>
                        <option value="H">H</option>
                      </select>
                    </div>
                    <div class="col-span-1">
                      <div class="text-[10px] text-slate-500 mb-1">% Living</div>
                      <input v-model="row.pct_living" type="number" class="w-full border rounded px-2 py-1 text-xs" />
                    </div>
                    <div class="col-span-2">
                      <div class="text-[10px] text-slate-500 mb-1">Remark</div>
                      <input v-model="row.remark" class="w-full border rounded px-2 py-1 text-xs" />
                    </div>
                    <div class="col-span-2 flex items-center gap-2">
                      <input v-model="row.bleached" type="checkbox" />
                      <span class="text-xs text-slate-600">Bleached</span>
                    </div>
                  </div>
                </div>
              </div>

              <div class="mt-3 flex items-center gap-2">
                <NcButton type="primary" size="small" @click="addRow">+ Add Row</NcButton>
                <NcButton type="secondary" size="small" @click="markComplete">Submit / update status</NcButton>
                <span class="text-xs text-slate-500">Autosave is on. Pausing/exiting keeps current WIP data.</span>
              </div>
            </section>
            </div>
          </div>
        </Teleport>
      </template>
    </main>
  </div>
</template>
