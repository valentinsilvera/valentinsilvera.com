<template>
  <div class="layout">
    <AppTopBar />
    <main class="layout__main page-shell">
      <slot />
    </main>
  </div>
</template>

<style scoped>
.layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.layout__main {
  flex: 1;
  padding-top: 48px;
  padding-bottom: 64px;
}
</style>
