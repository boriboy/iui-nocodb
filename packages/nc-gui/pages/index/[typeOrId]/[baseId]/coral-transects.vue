<script setup lang="ts">
import type { TableType } from 'nocodb-sdk'

definePageMeta({
  hideHeader: true,
  hasSidebar: true,
})

const { base } = storeToRefs(useBase())
const { baseTables } = storeToRefs(useTablesStore())
const { loadProjectTables, openTable } = useTablesStore()
const { getMeta } = useMetas()

const transectTable = ref<TableType | null>(null)
const isLoading = ref(true)

const resolveTransectTable = async () => {
  if (!base.value?.id) return null

  await loadProjectTables(base.value.id)

  const tables = baseTables.value.get(base.value.id) ?? []
  const found =
    tables.find((table) => table.id === 'coral_transects') ||
    tables.find((table) => table.title === 'coral_transects') ||
    tables.find((table) => table.title === 'Coral Transects') ||
    tables.find((table) => table.table_name === 'coral_transects') ||
    tables.find((table) => table.table_name === 'Coral Transects')

  if (!found) return null

  const meta = (await getMeta(base.value.id, found.id as string)) as TableType | null
  return meta ?? found
}

const launchLiveEditor = async () => {
  if (!transectTable.value) return

  await openTable(transectTable.value, true)
}

onMounted(async () => {
  try {
    transectTable.value = await resolveTransectTable()
  } finally {
    isLoading.value = false
  }
})
</script>

<template>
  <div class="h-full overflow-auto bg-slate-50">
    <div class="min-h-full px-6 py-6 lg:px-10">
      <div class="mx-auto max-w-5xl space-y-6">
        <div class="rounded-2xl border border-slate-200 bg-white shadow-sm overflow-hidden">
          <div class="bg-gradient-to-r from-blue-950 via-slate-900 to-cyan-900 px-6 py-5 text-white">
            <div class="text-xs uppercase tracking-[0.24em] text-cyan-200/80">Coral data entry</div>
            <h1 class="mt-2 text-2xl font-semibold">Coral transect interface</h1>
            <p class="mt-2 max-w-2xl text-sm text-cyan-50/85">
              Launch the live transect table from here. All edits use the existing NocoDB row APIs, so changes persist
              immediately to the database.
            </p>
          </div>

          <div class="grid gap-4 p-6 lg:grid-cols-[1.5fr_1fr]">
            <div class="space-y-4">
              <div class="rounded-xl border border-slate-200 bg-slate-50 p-4">
                <div class="text-sm font-semibold text-slate-900">What this opens</div>
                <div class="mt-2 text-sm text-slate-600">
                  The live coral transect table in NocoDB. Use it to add and edit transect rows with immediate
                  database writes.
                </div>
              </div>

              <div class="rounded-xl border border-slate-200 bg-white p-4">
                <div class="text-xs uppercase tracking-wide text-slate-500">Table status</div>
                <div class="mt-2 text-sm text-slate-700" v-if="isLoading">Loading transect table...</div>
                <div class="mt-2 text-sm text-slate-700" v-else-if="transectTable">
                  Ready: {{ transectTable.title || transectTable.table_name || 'coral_transects' }}
                </div>
                <div class="mt-2 text-sm text-rose-600" v-else>Could not find a coral transect table in this base.</div>
              </div>
            </div>

            <div class="rounded-2xl border border-slate-200 bg-slate-900 p-5 text-white">
              <div class="text-xs uppercase tracking-[0.22em] text-cyan-300/80">Live editor</div>
              <div class="mt-3 text-lg font-semibold">Open the working table</div>
              <p class="mt-2 text-sm text-slate-300">
                This button opens the real coral transect grid, not a mock copy.
              </p>
              <NcButton
                class="mt-5 w-full"
                type="primary"
                :disabled="!transectTable || isLoading"
                @click="launchLiveEditor()"
              >
                Open coral transect table
              </NcButton>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>