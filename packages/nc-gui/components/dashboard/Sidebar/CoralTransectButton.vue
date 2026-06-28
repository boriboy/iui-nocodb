<script setup lang="ts">
const router = useRouter()

const baseStore = useBase()
const { base } = storeToRefs(baseStore)

const openCoralTransectEditor = async () => {
  if (!base.value?.id) return

  await router.push(`/nc/${base.value.id}/coral-transects`)
}
</script>

<template>
  <NcSidebarMenuItem
    v-if="base?.id"
    class="group !my-1 !h-11 !gap-3 !text-sm"
    data-testid="nc-coral-transect-launcher"
    @click="openCoralTransectEditor()"
  >
    <template #icon>
      <GeneralIcon icon="ncTable" class="flex-none h-5 w-5" />
    </template>
    <span>Coral transects</span>
  </NcSidebarMenuItem>
</template>