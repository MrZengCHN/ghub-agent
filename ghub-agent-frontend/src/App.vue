<script setup>
import { RouterLink, RouterView } from "vue-router"
</script>

<template>
  <div class="app-shell">
    <header class="app-nav">
      <nav class="nav-bar" aria-label="Main navigation">
        <RouterLink to="/" class="nav-logo">
          <span class="logo-mark">GA</span>
          <span class="logo-title">GHUB Agent</span>
        </RouterLink>
                <div class="nav-links">
          <RouterLink to="/" class="nav-link">Home</RouterLink>
          <RouterLink to="/agents" class="nav-link">Agents</RouterLink>
          <RouterLink to="/ballistic" class="nav-link">弹道绘制</RouterLink>
          <RouterLink to="/about" class="nav-link">About</RouterLink>
        </div>
      </nav>
    </header>
    <main class="app-main">
      <RouterView />
    </main>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  height: 100vh;
  background-color: #f8fafc;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.app-nav {
  background: linear-gradient(135deg, #020617, #111827);
  color: #fff;
  box-shadow: 0 12px 30px rgba(15, 23, 42, 0.35);
  flex-shrink: 0;
}

.nav-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 18px;
  width: min(1100px, 100%);
  margin: 0 auto;
}

.nav-logo {
  font-weight: 700;
  font-size: 1rem;
  color: #fff;
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  gap: 12px;
}

.logo-mark {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border-radius: 9px;
  background: linear-gradient(135deg, #16a34a, #22d3ee);
  color: #0f172a;
  font-weight: 800;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.logo-title {
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.nav-links {
  display: flex;
  align-items: center;
  gap: 10px;
  background: rgba(15, 23, 42, 0.35);
  border: 1px solid rgba(148, 163, 184, 0.25);
  border-radius: 999px;
  padding: 5px;
  backdrop-filter: blur(12px);
}

.nav-link {
  color: #fff;
  text-decoration: none;
  font-weight: 500;
  padding: 6px 12px;
  border-radius: 999px;
  transition: background-color 0.2s ease, color 0.2s ease, transform 0.2s ease;
}

.nav-link:hover {
  background-color: rgba(148, 163, 184, 0.2);
  transform: translateY(-1px);
}

.nav-link.router-link-active {
  background: linear-gradient(135deg, #22d3ee, #3b82f6);
  color: #0f172a;
  font-weight: 600;
  box-shadow: 0 8px 20px rgba(34, 211, 238, 0.35);
}

.app-main {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
}

@media (max-width: 640px) {
  .nav-bar {
    flex-direction: column;
    align-items: stretch;
    gap: 18px;
    padding: 20px;
  }

  .nav-links {
    width: 100%;
    justify-content: center;
    gap: 8px;
  }

  .nav-link {
    flex: 1;
    text-align: center;
  }
}
</style>
