<script setup lang="ts">
const isPublic = inject(IsPublicInj, ref(false))

const { isMobileMode } = storeToRefs(useConfigStore())

const router = useRouter()
const route = useRoute()

const { base } = storeToRefs(useBase())

const goToHome = async () => {
  const targetBaseId = base.value?.id || String(route.params.baseId || '')
  if (!targetBaseId) return

  await router.push({ path: `/nc/${targetBaseId}`, query: { page: 'home' } })
}
</script>

<template>
  <div v-if="!isPublic && !isMobileMode" class="empty:hidden">
    <button type="button" class="iui-topbar-logo-btn" @click="goToHome">
      <img src="/iui.jpeg" alt="IUI" class="iui-topbar-logo" />
    </button>
  </div>
</template>

<style scoped>
.iui-topbar-logo-btn {
  background: transparent;
  border: 0;
  padding: 0;
  cursor: pointer;
}

.iui-topbar-logo {
  height: 1.75rem;
  width: auto;
  object-fit: contain;
}
</style>
