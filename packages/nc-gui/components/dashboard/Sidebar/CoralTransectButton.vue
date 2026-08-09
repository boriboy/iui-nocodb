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
  <div v-if="false && base?.id" class="flex flex-col gap-1">
    <NcSidebarMenuItem
      class="group !my-0 !h-10 !gap-3 !text-sm"
      data-testid="nc-coral-transect-launcher"
      @click="openCoralTransectEditor()"
    >
      <template #icon>
        <GeneralIcon icon="ncTable" class="flex-none h-5 w-5" />
      </template>
      <span>Coral transects</span>
    </NcSidebarMenuItem>
  </div>
</template>