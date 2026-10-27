<template>
  <div class="app-shell" :class="{ 'is-collapsed': sidebarCollapsed, 'drawer-open': sidebarOpen }">
    <aside class="sidebar" id="app-sidebar">
      <div class="sidebar-brand">
        <span class="brand-monogram" aria-hidden="true">{{ brandInitial }}</span>
        <div class="brand-text">
          <span class="brand-name">{{ t('nav.companyName') }}</span>
          <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
        </div>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          class="sidebar-link"
          :class="{ active: $route.path === item.path }"
          :title="t(item.labelKey)"
        >
          <svg class="nav-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path v-for="d in icons[item.icon]" :key="d" :d="d" />
          </svg>
          <span class="nav-label">{{ t(item.labelKey) }}</span>
        </router-link>
      </nav>

      <button
        class="sidebar-collapse"
        type="button"
        :aria-label="collapseLabel"
        :title="collapseLabel"
        :aria-expanded="String(!sidebarCollapsed)"
        @click="toggleCollapsed"
      >
        <svg class="collapse-icon" :class="{ rotated: sidebarCollapsed }" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <path :d="icons.chevron[0]" />
        </svg>
        <span class="nav-label">{{ collapseLabel }}</span>
      </button>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div v-if="sidebarOpen" class="sidebar-backdrop" @click="sidebarOpen = false"></div>

    <div class="app-main">
      <header class="topbar">
        <button
          class="menu-toggle"
          type="button"
          :aria-label="t('nav.toggleNavigation')"
          aria-controls="app-sidebar"
          :aria-expanded="String(sidebarOpen)"
          @click="sidebarOpen = !sidebarOpen"
        >
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" aria-hidden="true">
            <path v-for="d in icons.menu" :key="d" :d="d" />
          </svg>
        </button>
        <h1 class="topbar-title">{{ topbarTitle }}</h1>
        <FilterBar />
      </header>
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, computed, watch } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    const navItems = [
      { path: '/', labelKey: 'nav.overview', icon: 'grid' },
      { path: '/inventory', labelKey: 'nav.inventory', icon: 'box' },
      { path: '/orders', labelKey: 'nav.orders', icon: 'cart' },
      { path: '/spending', labelKey: 'nav.finance', icon: 'dollar' },
      { path: '/demand', labelKey: 'nav.demandForecast', icon: 'trending' },
      { path: '/reports', labelKey: 'nav.reports', icon: 'file' }
    ]

    // SVG path data per icon (24x24 viewBox, stroked)
    const icons = {
      grid: ['M3 3h7v7H3z', 'M14 3h7v7h-7z', 'M14 14h7v7h-7z', 'M3 14h7v7H3z'],
      box: ['M21 8l-9-5-9 5v8l9 5 9-5z', 'M3.3 7.5l8.7 5 8.7-5', 'M12 22V12.5'],
      cart: ['M3 3h2l2.4 12.2a2 2 0 0 0 2 1.8h8.2a2 2 0 0 0 2-1.6L21 8H6', 'M9 21a1 1 0 1 0 0-2 1 1 0 0 0 0 2z', 'M18 21a1 1 0 1 0 0-2 1 1 0 0 0 0 2z'],
      dollar: ['M12 2v20', 'M17 6H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6'],
      trending: ['M22 7l-8.5 8.5-5-5L2 17', 'M16 7h6v6'],
      file: ['M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z', 'M14 2v6h6', 'M8 13h8', 'M8 17h8'],
      chevron: ['M15 18l-6-6 6-6'],
      menu: ['M4 6h16', 'M4 12h16', 'M4 18h16']
    }

    // Mobile drawer state
    const sidebarOpen = ref(false)

    // User-controlled icons-only rail. localStorage can throw (blocked storage,
    // private mode), so every access is guarded and falls back to expanded.
    const readCollapsed = () => {
      try {
        return localStorage.getItem('sidebarCollapsed') === 'true'
      } catch (e) {
        return false
      }
    }
    const sidebarCollapsed = ref(readCollapsed())
    const toggleCollapsed = () => {
      sidebarCollapsed.value = !sidebarCollapsed.value
      try {
        localStorage.setItem('sidebarCollapsed', String(sidebarCollapsed.value))
      } catch (e) {
        // Ignore: the preference just won't persist
      }
    }

    // Close the mobile drawer after navigating. The sidebar stays mounted across
    // route changes, so without this it would remain open over the new page.
    watch(() => route.path, () => {
      sidebarOpen.value = false
    })

    const collapseLabel = computed(() =>
      sidebarCollapsed.value ? t('nav.expandSidebar') : t('nav.collapseSidebar')
    )
    const brandInitial = computed(() => (t('nav.companyName') || '').trim().charAt(0).toUpperCase())
    const topbarTitle = computed(() => {
      const active = navItems.find(item => item.path === route.path)
      return active ? t(active.labelKey) : t('nav.companyName')
    })

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      navItems,
      icons,
      sidebarOpen,
      sidebarCollapsed,
      toggleCollapsed,
      collapseLabel,
      brandInitial,
      topbarTitle,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
:root {
  /* Neutrals (slate) */
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-surface-muted: #f1f5f9;
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-text: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #64748b;
  --color-text-subtle: #94a3b8;

  /* Brand */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-primary-soft: #eff6ff;
  --color-primary-border: #bfdbfe;

  /* Status: fg / soft bg / strong text */
  --color-success: #10b981;  --color-success-soft: #d1fae5;  --color-success-strong: #065f46;
  --color-info: #3b82f6;     --color-info-soft: #dbeafe;     --color-info-strong: #1e40af;
  --color-warning: #f59e0b;  --color-warning-soft: #fef3c7;  --color-warning-strong: #92400e;
  --color-danger: #ef4444;   --color-danger-soft: #fee2e2;   --color-danger-strong: #991b1b;

  /* Sidebar */
  --sidebar-bg: #0f172a;
  --sidebar-text: #94a3b8;
  --sidebar-text-active: #ffffff;
  --sidebar-item-hover: rgba(148, 163, 184, 0.12);
  --sidebar-item-active: rgba(59, 130, 246, 0.18);

  /* Spacing (4px base) */
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px; --space-4: 16px;
  --space-5: 24px; --space-6: 32px; --space-7: 48px;

  /* Radius, shadow, type */
  --radius-sm: 6px; --radius-md: 8px; --radius-lg: 12px; --radius-full: 999px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.05);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 12px 32px rgba(15, 23, 42, 0.12);
  --font-xs: 0.75rem; --font-sm: 0.8125rem; --font-base: 0.875rem;
  --font-md: 1rem; --font-lg: 1.25rem; --font-xl: 1.5rem;

  /* Layout */
  --sidebar-width: 248px;
  --sidebar-collapsed: 72px;
  --topbar-height: 64px;
  --content-max: 1440px;
  --transition: 150ms ease;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: var(--color-bg);
  color: var(--color-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ---------- Layout shell ---------- */
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  transition: grid-template-columns var(--transition);
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: var(--sidebar-bg);
  color: var(--sidebar-text);
  padding: var(--space-4) var(--space-3);
  gap: var(--space-4);
  /* Above the topbar; menu dropdowns open upward inside this context */
  z-index: 100;
}

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  min-height: 40px;
}

.brand-monogram {
  display: none;
  width: 36px;
  height: 36px;
  border-radius: var(--radius-md);
  background: var(--color-primary);
  color: var(--sidebar-text-active);
  font-weight: 700;
  font-size: var(--font-md);
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.brand-name {
  color: var(--sidebar-text-active);
  font-size: var(--font-md);
  font-weight: 700;
  letter-spacing: -0.025em;
}

.brand-subtitle {
  font-size: var(--font-xs);
  color: var(--sidebar-text);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  flex: 1;
  overflow-y: auto;
}

.sidebar-link,
.sidebar-collapse {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-md);
  color: var(--sidebar-text);
  font-size: var(--font-base);
  font-weight: 500;
  text-decoration: none;
  transition: background var(--transition), color var(--transition);
}

.sidebar-link:hover,
.sidebar-collapse:hover {
  background: var(--sidebar-item-hover);
  color: var(--sidebar-text-active);
}

.sidebar-link.active {
  background: var(--sidebar-item-active);
  color: var(--sidebar-text-active);
}

.sidebar-link.active::before {
  content: '';
  position: absolute;
  left: 0;
  top: var(--space-2);
  bottom: var(--space-2);
  width: 3px;
  border-radius: var(--radius-full);
  background: var(--color-primary);
}

.nav-icon,
.collapse-icon {
  flex-shrink: 0;
}

.sidebar-collapse {
  width: 100%;
  background: none;
  border: none;
  cursor: pointer;
  font-family: inherit;
  text-align: left;
}

.collapse-icon {
  transition: transform var(--transition);
}

.collapse-icon.rotated {
  transform: rotate(180deg);
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-3);
  border-top: 1px solid rgba(148, 163, 184, 0.2);
}

.sidebar-backdrop {
  display: none;
}

.app-main {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: 50;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: var(--space-3) var(--space-4);
  min-height: var(--topbar-height);
  padding: var(--space-2) var(--space-6);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}

.topbar-title {
  font-size: var(--font-lg);
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.topbar > :last-child {
  margin-left: auto;
}

.menu-toggle {
  display: none;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  background: none;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  color: var(--color-text-secondary);
  cursor: pointer;
}

.menu-toggle:hover {
  background: var(--color-surface-muted);
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: var(--content-max);
  margin: 0 auto;
  padding: var(--space-6);
}

/* ---------- Icons-only mode ----------
   One shared rule set for the manual (.is-collapsed) and automatic (<=1024px)
   modes so both look identical. */
.app-shell.is-collapsed {
  grid-template-columns: var(--sidebar-collapsed) minmax(0, 1fr);
}

.app-shell.is-collapsed .brand-text,
.app-shell.is-collapsed .nav-label,
.app-shell.is-collapsed .language-label,
.app-shell.is-collapsed .profile-name,
.app-shell.is-collapsed .sidebar-footer .chevron {
  display: none;
}

.app-shell.is-collapsed .brand-monogram {
  display: flex;
}

.app-shell.is-collapsed .sidebar-brand,
.app-shell.is-collapsed .sidebar-link,
.app-shell.is-collapsed .sidebar-collapse,
.app-shell.is-collapsed .sidebar-footer .language-button,
.app-shell.is-collapsed .sidebar-footer .profile-button {
  justify-content: center;
  padding-left: 0;
  padding-right: 0;
}

/* Tablet: force icons-only whatever the stored preference (a 248px sidebar
   would eat too much width), and hide the collapse button since there is
   nothing to expand into. The drawer rules below take over at <=768px. */
@media (max-width: 1024px) {
  .app-shell {
    grid-template-columns: var(--sidebar-collapsed) minmax(0, 1fr);
  }

  .brand-text,
  .nav-label,
  .language-label,
  .profile-name,
  .sidebar-footer .chevron {
    display: none;
  }

  .brand-monogram {
    display: flex;
  }

  .sidebar-brand,
  .sidebar-link,
  .sidebar-footer .language-button,
  .sidebar-footer .profile-button {
    justify-content: center;
    padding-left: 0;
    padding-right: 0;
  }

  .sidebar-collapse {
    display: none;
  }

  .main-content {
    padding: var(--space-5);
  }

  .topbar {
    padding: var(--space-2) var(--space-5);
  }
}

/* Mobile: off-canvas drawer. Re-expands labels (overriding both icons-only
   modes) because the drawer is a full-width panel. */
@media (max-width: 768px) {
  .app-shell,
  .app-shell.is-collapsed {
    grid-template-columns: minmax(0, 1fr);
  }

  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    bottom: 0;
    width: var(--sidebar-width);
    z-index: 200;
    transform: translateX(-100%);
    transition: transform var(--transition);
  }

  .app-shell.drawer-open .sidebar {
    transform: translateX(0);
  }

  /* .is-collapsed variants are needed for equal specificity with the icons-only rules */
  .app-shell .nav-label,
  .app-shell.is-collapsed .nav-label,
  .app-shell .language-label,
  .app-shell.is-collapsed .language-label,
  .app-shell .profile-name,
  .app-shell.is-collapsed .profile-name {
    display: inline;
  }

  .app-shell .brand-text,
  .app-shell.is-collapsed .brand-text {
    display: flex;
  }

  .app-shell .sidebar-footer .chevron,
  .app-shell.is-collapsed .sidebar-footer .chevron {
    display: block;
  }

  .app-shell .brand-monogram,
  .app-shell.is-collapsed .brand-monogram {
    display: none;
  }

  .app-shell .sidebar-brand,
  .app-shell.is-collapsed .sidebar-brand,
  .app-shell .sidebar-link,
  .app-shell.is-collapsed .sidebar-link,
  .app-shell .sidebar-footer .language-button,
  .app-shell.is-collapsed .sidebar-footer .language-button,
  .app-shell .sidebar-footer .profile-button,
  .app-shell.is-collapsed .sidebar-footer .profile-button {
    justify-content: flex-start;
    padding-left: var(--space-3);
    padding-right: var(--space-3);
  }

  .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.5);
    z-index: 150;
  }

  .menu-toggle {
    display: inline-flex;
  }

  .topbar {
    padding: var(--space-2) var(--space-4);
  }

  .topbar > :last-child {
    margin-left: 0;
    width: 100%;
  }

  .main-content {
    padding: var(--space-4);
  }
}

/* ---------- Global component styles ---------- */
.page-header {
  margin-bottom: var(--space-5);
}

.page-header h2 {
  font-size: var(--font-xl);
  font-weight: 700;
  color: var(--color-text);
  margin-bottom: var(--space-1);
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--color-text-muted);
  font-size: var(--font-base);
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--space-4);
  margin-bottom: var(--space-5);
}

.stat-card,
.card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-lg);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
}

.stat-card {
  transition: box-shadow var(--transition), border-color var(--transition);
}

.stat-card:hover {
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-md);
}

.card {
  margin-bottom: var(--space-4);
}

.stat-label {
  color: var(--color-text-muted);
  font-size: var(--font-base);
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: var(--color-success);
}

.stat-card.danger .stat-value {
  color: var(--color-danger);
}

.stat-card.info .stat-value {
  color: var(--color-primary);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-4);
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: var(--font-md);
  font-weight: 600;
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--color-surface-muted);
  border-top: 1px solid var(--color-border);
  border-bottom: 1px solid var(--color-border);
}

th {
  position: sticky;
  top: 0;
  background: var(--color-surface-muted);
  text-align: left;
  padding: var(--space-2) var(--space-4);
  font-weight: 600;
  color: var(--color-text-muted);
  font-size: var(--font-xs);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

td {
  padding: var(--space-3) var(--space-4);
  border-top: 1px solid var(--color-surface-muted);
  color: var(--color-text-secondary);
  font-size: var(--font-base);
}

tbody tr {
  transition: background-color var(--transition);
}

tbody tr:hover {
  background: var(--color-surface-muted);
}

.badge {
  display: inline-block;
  padding: 2px var(--space-2);
  border-radius: var(--radius-full);
  font-size: var(--font-xs);
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success,
.badge.increasing {
  background: var(--color-success-soft);
  color: var(--color-success-strong);
}

.badge.warning,
.badge.medium {
  background: var(--color-warning-soft);
  color: var(--color-warning-strong);
}

.badge.danger,
.badge.decreasing,
.badge.high {
  background: var(--color-danger-soft);
  color: var(--color-danger-strong);
}

.badge.info,
.badge.low {
  background: var(--color-info-soft);
  color: var(--color-info-strong);
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.loading {
  text-align: center;
  padding: var(--space-7);
  color: var(--color-text-muted);
  font-size: var(--font-base);
}

.error {
  background: var(--color-danger-soft);
  border: 1px solid var(--color-danger-soft);
  color: var(--color-danger-strong);
  padding: var(--space-7) var(--space-4);
  text-align: center;
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: var(--font-base);
}
</style>
