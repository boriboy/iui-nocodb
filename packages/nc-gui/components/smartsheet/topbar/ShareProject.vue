<script setup lang="ts">
const isPublic = inject(IsPublicInj, ref(false))

const { isMobileMode } = storeToRefs(useConfigStore())
</script>

<template>
  <div v-if="!isPublic && !isMobileMode" class="iui-topbar-logos empty:hidden">
    <a href="https://iui-eilat.ac.il/" target="_blank" rel="noopener noreferrer" class="iui-topbar-logo-btn">
      <img src="/iui.jpeg" alt="IUI" class="iui-topbar-logo" />
    </a>
    <a
      href="https://iui-eilat.huji.ac.il/en/about-monitoring-program"
      target="_blank"
      rel="noopener noreferrer"
      class="iui-topbar-logo-btn"
    >
      <img src="/nmp.jpeg" alt="NMP" class="iui-topbar-nmp-logo" />
    </a>
    <a href="https://new.huji.ac.il/" target="_blank" rel="noopener noreferrer" class="iui-topbar-logo-btn">
      <img src="/HujiNew.svg" alt="HUJI" class="iui-topbar-huji-logo" />
    </a>
  </div>
</template>

<style scoped>
.iui-topbar-logos {
  display: flex;
  align-items: stretch;
  justify-content: flex-end;
  gap: 0.5rem;
  /* topbar height minus py-2 padding and bottom border */
  height: calc(var(--topbar-height) - 1rem - 1px);
  flex-shrink: 0;
  width: auto;
}

.iui-topbar-logo-btn {
  background: transparent;
  border: 0;
  padding: 0;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  align-self: stretch;
}

.iui-topbar-logo,
.iui-topbar-nmp-logo,
.iui-topbar-huji-logo {
  height: 100%;
  width: auto;
  max-width: none;
  object-fit: contain;
  display: block;
}
</style>
