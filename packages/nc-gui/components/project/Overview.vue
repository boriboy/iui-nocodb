<script lang="ts" setup>
import type { TableType } from 'nocodb-sdk'

const { openedProject, isDataSourceLimitReached } = storeToRefs(useBases())

const baseStore = useBase()
const { base } = storeToRefs(baseStore)

const isNewBaseModalOpen = ref(false)

const { isMobileMode } = useGlobal()

const { isUIAllowed } = useRoles()

const router = useRouter()

const { $e } = useNuxtApp()

const { t } = useI18n()

const { showEEFeatures, showExternalSourcePlanLimitExceededModal } = useEeConfig()

const { activeSidebarTab } = storeToRefs(useSidebarStore())

const tabActionLabel = computed(() => {
  const labels: Record<string, string> = {
    workflows: t('objects.workflow'),
    docs: t('objects.document'),
  }
  return labels[activeSidebarTab.value] ?? t('general.data')
})

const overviewHeading = computed(() => {
  if (activeSidebarTab.value === 'data') return 'IUI Biologists Workspace'
  return `${tabActionLabel.value} ${t('labels.actions')}`
})

const isImportModalOpen = ref(false)

const defaultBase = computed(() => {
  return openedProject.value?.sources?.[0]
})

function openTableCreateDialog(baseIndex?: number | undefined) {
  $e('c:table:create:navdraw')

  const isOpen = ref(true)
  let sourceId = openedProject.value!.sources?.[0].id
  if (typeof baseIndex === 'number') {
    sourceId = openedProject.value!.sources?.[baseIndex].id
  }

  if (!sourceId || !openedProject.value?.id) return

  const { close } = useDialog(resolveComponent('DlgTableCreate'), {
    'modelValue': isOpen,
    sourceId,
    'baseId': openedProject.value.id,
    'onCreate': closeDialog,
    'onUpdate:modelValue': () => closeDialog(),
  })

  function closeDialog(table?: TableType) {
    isOpen.value = false

    if (!table) return

    // TODO: Better way to know when the table node dom is available
    setTimeout(() => {
      const newTableDom = document.querySelector(`[data-table-id="${table.id}"]`)
      if (!newTableDom) return

      newTableDom?.scrollIntoView({ behavior: 'smooth', block: 'nearest' })
    }, 1000)

    close(1000)
  }
}

const onCreateBaseClick = () => {
  if (showExternalSourcePlanLimitExceededModal() || isDataSourceLimitReached.value) return

  isNewBaseModalOpen.value = true
}

const openCoralReport = () => {
  if (!base.value?.id) return
  router.push(`/nc/${base.value.id}/coral-query`)
}

const openLagoonReport = () => {
  if (!base.value?.id) return
  router.push(`/nc/${base.value.id}/lagoon-query`)
}
</script>

<template>
  <div class="nc-all-tables-view py-4 px-6 nc-scrollbar-thin h-full overflow-y-auto">
    <div class="text-subHeading2 text-nc-content-gray mb-5 -mt-1.5">{{ overviewHeading }}</div>

    <div
      class="nc-overview-actions flex flex-row gap-6 flex-wrap max-w-[1000px]"
      :class="{
        'pointer-events-none': base?.isLoading,
      }"
    >
      <template v-if="base?.isLoading">
        <ProjectActionItem v-for="item in 7" :key="item" is-loading label="loading" />
      </template>
      <template v-else>
        <!-- Data actions (shown on Data tab) -->
        <template v-if="activeSidebarTab === 'data'">
          <div class="iui-home w-full">
            <div class="iui-card-grid">
              <article class="iui-card iui-coral">
                <div>
                  <h3>Coral Transects</h3>
                  <p>Review coral transect outcomes and drill into observation-level details.</p>
                </div>
                <div class="iui-card-actions">
                  <button type="button" class="iui-btn iui-btn-primary" @click="openCoralReport">Open coral report</button>
                  <button type="button" class="iui-btn iui-btn-disabled" disabled title="Data entry interface coming soon">Data entry (coming soon)</button>
                </div>
              </article>

              <article class="iui-card iui-lagoon">
                <div>
                  <h3>Lagoon Quadrats</h3>
                  <p>Browse lagoon quadrat summaries and move from survey to quadrat and observation detail.</p>
                </div>
                <div class="iui-card-actions">
                  <button type="button" class="iui-btn iui-btn-primary" @click="openLagoonReport">Open lagoon report</button>
                  <button type="button" class="iui-btn iui-btn-disabled" disabled title="Data entry interface coming soon">Data entry (coming soon)</button>
                </div>
              </article>
            </div>
          </div>
        </template>

        <!-- Docs tab actions -->
        <template v-if="activeSidebarTab === 'docs' && showEEFeatures">
          <ProjectActionCreateNewDocument :base-id="base?.id" />
        </template>

        <!-- Automation actions (shown on Automation tab) -->
        <template v-if="activeSidebarTab === 'workflows' && !isMobileMode && showEEFeatures">
          <ProjectActionCreateEmptyWorkflow />
          <ProjectActionCreateEmptyScript />
          <ProjectActionScriptsByNocoDB />
        </template>
      </template>
    </div>

    <div v-if="!base.isLoading" class="nc-overview-empty-placeholder">
      <NcEmptyPlaceholder :title="$t('msg.noActionsAvailable')" />
    </div>

  </div>
</template>

<style lang="scss" scoped>
.nc-overview-empty-placeholder {
  @apply mt-10;
  display: none;
}

.nc-overview-actions:empty ~ .nc-overview-empty-placeholder {
  display: block;
}

.iui-home {
  width: 100%;
  max-width: 1100px;
}

.iui-card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(290px, 1fr));
  gap: 1rem;
}

.iui-card {
  border-radius: 16px;
  border: 1px solid #dbe4ef;
  background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
  padding: 1.15rem;
  min-height: 230px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.06);
}

.iui-coral {
  background: linear-gradient(180deg, #fffef7 0%, #fff8ea 100%);
}

.iui-lagoon {
  background: linear-gradient(180deg, #f6fffc 0%, #ecfdfa 100%);
}

.iui-card h3 {
  margin: 0 0 0.45rem;
  color: #0f172a;
  font-size: 1.2rem;
  font-weight: 700;
}

.iui-card p {
  margin: 0;
  color: #475569;
  font-size: 0.92rem;
  line-height: 1.45;
}

.iui-card-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6rem;
  margin-top: 1rem;
}

.iui-btn {
  border-radius: 10px;
  border: 1px solid #cbd5e1;
  padding: 0.5rem 0.85rem;
  font-size: 0.84rem;
  font-weight: 600;
  transition: all 0.15s ease;
}

.iui-btn-primary {
  background: #f8fafc;
  border-color: #94a3b8;
  color: #0f172a;
}

.iui-btn-primary:hover {
  background: #f1f5f9;
}

.iui-btn-disabled {
  background: #e2e8f0;
  border-color: #cbd5e1;
  color: #64748b;
  cursor: not-allowed;
}
</style>
