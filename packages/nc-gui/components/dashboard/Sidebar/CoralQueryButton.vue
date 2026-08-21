<script setup lang="ts">
const router = useRouter()

const basesStore = useBases()
const baseStore = useBase()
const { base } = storeToRefs(baseStore)
const { resolvedProject } = storeToRefs(basesStore)

const openCoralQuery = async () => {
  const targetBaseId = base.value?.id || resolvedProject.value?.id
  if (!targetBaseId) return

  await router.push(`/nc/${targetBaseId}/coral-query`)
}
</script>

<template>
  <div v-if="base?.id || resolvedProject?.id" class="flex flex-col gap-1">
    <NcSidebarMenuItem
      class="group !my-0 !h-10 !gap-3 !text-sm"
      data-testid="nc-coral-query-launcher"
      @click="openCoralQuery()"
    >
      <template #icon>
        <GeneralIcon icon="ncTable" class="flex-none h-5 w-5" />
      </template>
      <span>Coral Transects</span>
    </NcSidebarMenuItem>
  </div>
</template>