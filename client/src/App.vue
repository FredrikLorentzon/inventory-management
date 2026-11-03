<template>
  <div class="app" :class="{ 'nav-open': mobileNavOpen, 'is-scrolled': isScrolled }">
    <aside id="app-sidebar" class="sidebar">
      <div class="sidebar-brand">
        <div class="brand-mark">
          <NavIcon name="logo" :size="20" />
        </div>
        <div class="brand-text">
          <span class="brand-name">{{ t('nav.companyName') }}</span>
          <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
        </div>
        <button
          type="button"
          class="icon-button sidebar-close"
          :aria-label="t('nav.closeNavigation')"
          @click="closeMobileNav"
        >
          <NavIcon name="close" />
        </button>
      </div>

      <nav
        ref="navRef"
        class="sidebar-nav"
        :class="{ 'has-indicator': indicator.visible }"
        :aria-label="t('nav.mainNavigation')"
      >
        <span
          class="nav-indicator"
          :class="{ 'is-ready': indicatorReady, 'is-visible': indicator.visible }"
          :style="indicatorStyle"
          aria-hidden="true"
        ></span>
        <router-link to="/" :class="{ active: $route.path === '/' }">
          <NavIcon name="overview" />
          <span class="nav-label">{{ t('nav.overview') }}</span>
        </router-link>
        <router-link to="/inventory" :class="{ active: $route.path === '/inventory' }">
          <NavIcon name="inventory" />
          <span class="nav-label">{{ t('nav.inventory') }}</span>
        </router-link>
        <router-link to="/orders" :class="{ active: $route.path === '/orders' }">
          <NavIcon name="orders" />
          <span class="nav-label">{{ t('nav.orders') }}</span>
        </router-link>
        <router-link to="/spending" :class="{ active: $route.path === '/spending' }">
          <NavIcon name="finance" />
          <span class="nav-label">{{ t('nav.finance') }}</span>
        </router-link>
        <router-link to="/demand" :class="{ active: $route.path === '/demand' }">
          <NavIcon name="demand" />
          <span class="nav-label">{{ t('nav.demandForecast') }}</span>
        </router-link>
        <router-link to="/reports" :class="{ active: $route.path === '/reports' }">
          <NavIcon name="reports" />
          <span class="nav-label">{{ t('nav.reports') }}</span>
        </router-link>
      </nav>
    </aside>

    <transition name="backdrop">
      <div
        v-if="mobileNavOpen"
        class="sidebar-backdrop"
        aria-hidden="true"
        @click="closeMobileNav"
      ></div>
    </transition>

    <div class="main-column">
      <header class="topbar">
        <div class="topbar-inner">
          <button
            type="button"
            class="icon-button menu-button"
            :aria-label="t('nav.openNavigation')"
            aria-controls="app-sidebar"
            :aria-expanded="mobileNavOpen ? 'true' : 'false'"
            @click="toggleMobileNav"
          >
            <NavIcon name="menu" />
          </button>
          <div class="topbar-title">
            <span class="topbar-eyebrow">{{ t('nav.companyName') }}</span>
            <span class="topbar-separator" aria-hidden="true">/</span>
            <transition name="title-swap" mode="out-in">
              <span :key="currentPageLabel" class="topbar-page">{{ currentPageLabel }}</span>
            </transition>
          </div>
          <div class="topbar-actions">
            <LanguageSwitcher />
            <ProfileMenu
              @show-profile-details="showProfileDetails = true"
              @show-tasks="showTasks = true"
            />
          </div>
        </div>
      </header>

      <FilterBar />

      <main ref="mainRef" class="main-content" :class="{ 'is-entering': pageEntering }">
        <router-view v-slot="{ Component }">
          <transition name="page" mode="out-in" appear>
            <component :is="Component" :key="$route.path" />
          </transition>
        </router-view>
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
import { ref, onMounted, onBeforeUnmount, computed, watch, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import { useFilters } from './composables/useFilters'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'
import NavIcon from './components/NavIcon.vue'

const MOBILE_BREAKPOINT = 768
const ENTRANCE_WINDOW_MS = 1800

const NAV_LABEL_KEYS = {
  '/': 'nav.overview',
  '/inventory': 'nav.inventory',
  '/orders': 'nav.orders',
  '/spending': 'nav.finance',
  '/demand': 'nav.demandForecast',
  '/reports': 'nav.reports'
}

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
    NavIcon
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

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

    const route = useRoute()
    const router = useRouter()
    const { selectedPeriod, selectedLocation, selectedCategory, selectedStatus } = useFilters()

    const currentPageLabel = computed(() => {
      const key = NAV_LABEL_KEYS[route.path]
      return key ? t(key) : ''
    })

    const mobileNavOpen = ref(false)
    const isScrolled = ref(false)

    const toggleMobileNav = () => {
      mobileNavOpen.value = !mobileNavOpen.value
    }

    const closeMobileNav = () => {
      mobileNavOpen.value = false
    }

    const navRef = ref(null)
    const mainRef = ref(null)
    const indicator = ref({ top: 0, height: 0, visible: false })
    const indicatorReady = ref(false)

    const indicatorStyle = computed(() => ({
      transform: `translateY(${indicator.value.top}px)`,
      height: `${indicator.value.height}px`
    }))

    // The nav is the offsetParent of its links, so offsetTop is the distance the
    // pill has to travel. offsetTop ignores the drawer transform on mobile.
    const updateIndicator = () => {
      const nav = navRef.value
      if (!nav) return
      const activeLink = nav.querySelector('a.active')
      if (!activeLink) {
        indicator.value = { ...indicator.value, visible: false }
        return
      }
      indicator.value = {
        top: activeLink.offsetTop,
        height: activeLink.offsetHeight,
        visible: true
      }
    }

    // Staggered entrance plays only right after a route change. Filter changes
    // toggle each view's loading state and remount its content, so they close the
    // entrance window instead of replaying the animation. The window also closes
    // on its own once running animations settle, so later re-renders stay still.
    const pageEntering = ref(true)
    let entranceTimer = null
    let entranceId = 0

    const startPageEntrance = () => {
      const id = ++entranceId
      pageEntering.value = true
      clearTimeout(entranceTimer)
      entranceTimer = setTimeout(() => {
        const main = mainRef.value
        const running = main && typeof main.getAnimations === 'function'
          ? main.getAnimations({ subtree: true })
          : []
        Promise.all(running.map(animation => animation.finished.catch(() => null))).then(() => {
          if (id === entranceId) pageEntering.value = false
        })
      }, ENTRANCE_WINDOW_MS)
    }

    const endPageEntrance = () => {
      entranceId++
      clearTimeout(entranceTimer)
      pageEntering.value = false
    }

    watch(() => route.path, () => {
      mobileNavOpen.value = false
      startPageEntrance()
      nextTick(updateIndicator)
    })

    watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], endPageEntrance)

    const handleScroll = () => {
      isScrolled.value = window.scrollY > 4
    }

    const handleResize = () => {
      if (window.innerWidth >= MOBILE_BREAKPOINT) mobileNavOpen.value = false
      updateIndicator()
    }

    const handleKeydown = (event) => {
      if (event.key === 'Escape' && mobileNavOpen.value) closeMobileNav()
    }

    let navResizeObserver = null

    onMounted(() => {
      window.addEventListener('scroll', handleScroll, { passive: true })
      window.addEventListener('resize', handleResize)
      window.addEventListener('keydown', handleKeydown)
      handleScroll()
      startPageEntrance()

      if (typeof ResizeObserver !== 'undefined' && navRef.value) {
        navResizeObserver = new ResizeObserver(updateIndicator)
        navResizeObserver.observe(navRef.value)
      }

      if (document.fonts && document.fonts.ready) {
        document.fonts.ready.then(updateIndicator)
      }

      // Place the pill on the resolved route first, then enable its transition two
      // frames later so it does not glide in from the top on page load.
      router.isReady().then(() => nextTick(() => {
        updateIndicator()
        requestAnimationFrame(() => requestAnimationFrame(() => {
          indicatorReady.value = true
        }))
      }))
    })

    onBeforeUnmount(() => {
      window.removeEventListener('scroll', handleScroll)
      window.removeEventListener('resize', handleResize)
      window.removeEventListener('keydown', handleKeydown)
      if (navResizeObserver) navResizeObserver.disconnect()
      clearTimeout(entranceTimer)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      currentPageLabel,
      mobileNavOpen,
      isScrolled,
      toggleMobileNav,
      closeMobileNav,
      navRef,
      mainRef,
      indicator,
      indicatorReady,
      indicatorStyle,
      pageEntering
    }
  }
}
</script>

<style>
:root {
  --color-text: #0f172a;
  --color-text-body: #1e293b;
  --color-text-secondary: #475569;
  --color-text-muted: #64748b;
  --color-text-subtle: #94a3b8;
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-bg: #f8fafc;
  --color-bg-translucent: rgb(248 250 252 / 0.78);
  --color-surface: #ffffff;
  --color-surface-muted: #f1f5f9;
  --color-sidebar: #0f172a;
  --color-sidebar-deep: #0b1220;
  --color-sidebar-text: #94a3b8;
  --color-sidebar-text-hover: #e2e8f0;
  --color-sidebar-text-active: #ffffff;
  --color-sidebar-hover: rgb(148 163 184 / 0.1);
  --color-sidebar-divider: rgb(148 163 184 / 0.12);
  --color-accent: #2563eb;
  --color-accent-bright: #3b82f6;
  --color-accent-light: #60a5fa;
  --color-accent-soft: #eff6ff;

  --color-success: #059669;
  --color-warning: #ea580c;
  --color-danger: #dc2626;
  --color-info: #2563eb;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 24px;
  --space-6: 32px;
  --space-7: 48px;

  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 14px;
  --radius-pill: 999px;

  --shadow-card: 0 1px 2px rgb(15 23 42 / 0.04), 0 1px 3px rgb(15 23 42 / 0.06);
  --shadow-raised: 0 4px 12px rgb(15 23 42 / 0.08), 0 12px 32px -12px rgb(15 23 42 / 0.12);
  --focus-ring: 0 0 0 3px rgb(37 99 235 / 0.35);

  --sidebar-width: 240px;
  --topbar-height: 60px;
  --content-max: 1400px;
  --content-padding: var(--space-6);

  --ease-out: cubic-bezier(0.22, 1, 0.36, 1);
  --ease-in: cubic-bezier(0.55, 0, 1, 0.45);
  --ease-glide: cubic-bezier(0.34, 1.25, 0.64, 1);
  --duration-fast: 150ms;
  --duration-base: 220ms;
  --duration-slow: 420ms;
  --duration-enter: 560ms;
  --stagger-step: 55ms;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: var(--color-bg);
  color: var(--color-text-body);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  min-height: 100vh;
}

.sidebar {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  z-index: 50;
  width: var(--sidebar-width);
  display: flex;
  flex-direction: column;
  background:
    radial-gradient(120% 50% at 0% 0%, rgb(37 99 235 / 0.14), transparent 60%),
    linear-gradient(180deg, var(--color-sidebar) 0%, var(--color-sidebar-deep) 100%);
  border-right: 1px solid rgb(15 23 42 / 0.6);
  color: var(--color-sidebar-text);
}

.sidebar-brand {
  height: var(--topbar-height);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: 0 var(--space-4);
  border-bottom: 1px solid var(--color-sidebar-divider);
}

.brand-mark {
  width: 36px;
  height: 36px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--radius-md);
  color: var(--color-sidebar-text-active);
  background: linear-gradient(135deg, #334155 0%, var(--color-accent) 100%);
  box-shadow:
    inset 0 1px 0 rgb(255 255 255 / 0.18),
    0 0 0 1px rgb(96 165 250 / 0.25),
    0 6px 20px -4px rgb(37 99 235 / 0.55);
  transition: box-shadow var(--duration-slow) var(--ease-out), transform var(--duration-slow) var(--ease-out);
}

.sidebar-brand:hover .brand-mark {
  transform: rotate(-6deg) scale(1.04);
  box-shadow:
    inset 0 1px 0 rgb(255 255 255 / 0.22),
    0 0 0 1px rgb(96 165 250 / 0.4),
    0 8px 28px -4px rgb(37 99 235 / 0.75);
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
  flex: 1;
}

.brand-name {
  font-size: 0.875rem;
  font-weight: 700;
  color: var(--color-sidebar-text-active);
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.brand-subtitle {
  font-size: 0.75rem;
  color: var(--color-sidebar-text);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-nav {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  padding: var(--space-4) var(--space-3);
  overflow-y: auto;
}

.sidebar-nav a {
  position: relative;
  z-index: 1;
  height: 40px;
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: 0 var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--color-sidebar-text);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition:
    color var(--duration-base) var(--ease-out),
    background-color var(--duration-base) var(--ease-out),
    transform var(--duration-fast) var(--ease-out);
}

.sidebar-nav a .nav-icon {
  transition: color var(--duration-base) var(--ease-out), transform var(--duration-base) var(--ease-out);
}

.sidebar-nav a:hover {
  color: var(--color-sidebar-text-hover);
  background: var(--color-sidebar-hover);
}

.sidebar-nav a:hover .nav-icon {
  transform: translateX(1px);
}

.sidebar-nav a:active {
  transform: scale(0.98);
}

.sidebar-nav a:focus-visible {
  outline: none;
  box-shadow: 0 0 0 2px var(--color-accent-light);
}

.sidebar-nav a.active {
  color: var(--color-sidebar-text-active);
  background: rgb(37 99 235 / 0.2);
}

.sidebar-nav a.active .nav-icon {
  color: var(--color-accent-light);
}

.sidebar-nav.has-indicator a.active {
  background: transparent;
}

.nav-label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.nav-indicator {
  position: absolute;
  top: 0;
  left: var(--space-3);
  right: var(--space-3);
  z-index: 0;
  border-radius: var(--radius-sm);
  background: linear-gradient(90deg, rgb(37 99 235 / 0.3) 0%, rgb(37 99 235 / 0.12) 100%);
  box-shadow:
    inset 0 0 0 1px rgb(59 130 246 / 0.28),
    0 8px 24px -12px rgb(37 99 235 / 0.7);
  opacity: 0;
  pointer-events: none;
  will-change: transform;
}

.nav-indicator::before {
  content: '';
  position: absolute;
  top: 10px;
  bottom: 10px;
  left: 0;
  width: 3px;
  border-radius: var(--radius-pill);
  background: var(--color-accent-bright);
  box-shadow: 0 0 12px rgb(59 130 246 / 0.9);
}

.nav-indicator.is-visible {
  opacity: 1;
}

.nav-indicator.is-ready {
  transition:
    transform var(--duration-slow) var(--ease-glide),
    height var(--duration-slow) var(--ease-glide),
    opacity var(--duration-base) var(--ease-out);
}

.sidebar-brand .sidebar-close {
  display: none;
  color: var(--color-sidebar-text);
  background: transparent;
  border-color: var(--color-sidebar-divider);
}

.sidebar-brand .sidebar-close:hover {
  color: var(--color-sidebar-text-active);
  background: var(--color-sidebar-hover);
}

.sidebar-backdrop {
  display: none;
}

.main-column {
  margin-left: var(--sidebar-width);
  min-width: 0;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: 100;
  height: var(--topbar-height);
  background: var(--color-bg-translucent);
  -webkit-backdrop-filter: saturate(180%) blur(14px);
  backdrop-filter: saturate(180%) blur(14px);
  border-bottom: 1px solid transparent;
  transition:
    border-color var(--duration-base) var(--ease-out),
    box-shadow var(--duration-base) var(--ease-out);
}

.app.is-scrolled .topbar {
  border-bottom-color: var(--color-border);
}

.app.is-scrolled .filters-bar {
  box-shadow: 0 10px 24px -18px rgb(15 23 42 / 0.35);
}

.topbar-inner {
  height: 100%;
  max-width: calc(var(--content-max) + 2 * var(--content-padding));
  margin: 0 auto;
  padding: 0 var(--content-padding);
  display: flex;
  align-items: center;
  gap: var(--space-3);
}

.topbar-title {
  display: flex;
  align-items: baseline;
  gap: var(--space-2);
  min-width: 0;
  flex: 1;
}

.topbar-eyebrow {
  font-size: 0.875rem;
  color: var(--color-text-muted);
  white-space: nowrap;
}

.topbar-separator {
  color: var(--color-border-strong);
}

.topbar-page {
  display: inline-block;
  font-size: 0.938rem;
  font-weight: 600;
  color: var(--color-text);
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.topbar-actions {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  flex-shrink: 0;
}

.topbar-actions .language-button,
.topbar-actions .profile-button {
  transition:
    background-color var(--duration-base) var(--ease-out),
    border-color var(--duration-base) var(--ease-out),
    transform var(--duration-fast) var(--ease-out);
}

.topbar-actions .language-button:active,
.topbar-actions .profile-button:active {
  transform: scale(0.97);
}

.icon-button {
  width: 36px;
  height: 36px;
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-sm);
  background: var(--color-surface);
  color: var(--color-text-secondary);
  cursor: pointer;
  transition:
    background-color var(--duration-base) var(--ease-out),
    color var(--duration-base) var(--ease-out),
    transform var(--duration-fast) var(--ease-out);
}

.icon-button:hover {
  background: var(--color-bg);
  color: var(--color-text);
}

.icon-button:active {
  transform: scale(0.94);
}

.icon-button:focus-visible {
  outline: none;
  box-shadow: var(--focus-ring);
}

.topbar-inner .menu-button {
  display: none;
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: calc(var(--content-max) + 2 * var(--content-padding));
  margin: 0 auto;
  padding: var(--space-5) var(--content-padding) var(--space-7);
}

.page-header {
  margin-bottom: var(--space-5);
}

.page-header h2 {
  font-size: 1.75rem;
  font-weight: 700;
  color: var(--color-text);
  margin-bottom: var(--space-1);
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--color-text-muted);
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(240px, 100%), 1fr));
  gap: var(--space-5);
  margin-bottom: var(--space-5);
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-md);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-card);
  transition:
    transform var(--duration-base) var(--ease-out),
    box-shadow var(--duration-base) var(--ease-out),
    border-color var(--duration-base) var(--ease-out);
}

.stat-card:hover {
  transform: translateY(-2px);
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-raised);
}

.stat-label {
  color: var(--color-text-muted);
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
  font-variant-numeric: tabular-nums;
}

.stat-card.warning .stat-value {
  color: var(--color-warning);
}

.stat-card.success .stat-value {
  color: var(--color-success);
}

.stat-card.danger .stat-value {
  color: var(--color-danger);
}

.stat-card.info .stat-value {
  color: var(--color-info);
}

.card {
  background: var(--color-surface);
  border-radius: var(--radius-md);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-card);
  margin-bottom: var(--space-5);
  transition:
    transform var(--duration-base) var(--ease-out),
    box-shadow var(--duration-base) var(--ease-out),
    border-color var(--duration-base) var(--ease-out);
}

.card:hover {
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-raised);
}

.card.chart-card:hover {
  transform: translateY(-2px);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: var(--space-3);
  margin-bottom: var(--space-4);
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: var(--color-text);
  letter-spacing: -0.01em;
}

.table-container {
  overflow-x: auto;
  margin: 0 calc(-1 * var(--space-5));
  padding: 0 var(--space-5);
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--color-bg);
}

th {
  text-align: left;
  padding: var(--space-3);
  font-weight: 600;
  color: var(--color-text-muted);
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border-bottom: 1px solid var(--color-border);
  white-space: nowrap;
}

th:first-child {
  border-top-left-radius: var(--radius-sm);
}

th:last-child {
  border-top-right-radius: var(--radius-sm);
}

td {
  padding: var(--space-3);
  height: 48px;
  border-bottom: 1px solid var(--color-surface-muted);
  color: #334155;
  font-size: 0.875rem;
  font-variant-numeric: tabular-nums;
}

tbody tr:last-child td {
  border-bottom: none;
}

tbody tr {
  transition: background-color var(--duration-fast) var(--ease-out);
}

tbody tr:hover {
  background: var(--color-bg);
}

.badge {
  display: inline-flex;
  align-items: center;
  padding: var(--space-1) 10px;
  border-radius: var(--radius-pill);
  font-size: 0.75rem;
  font-weight: 600;
  line-height: 1.3;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  white-space: nowrap;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: var(--space-7);
  color: var(--color-text-muted);
  font-size: 0.938rem;
  animation: fade-in var(--duration-slow) var(--ease-out) 120ms backwards;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: var(--space-4);
  border-radius: var(--radius-sm);
  margin: var(--space-4) 0;
  font-size: 0.938rem;
}

.page-enter-active {
  transition:
    opacity var(--duration-base) var(--ease-out),
    transform var(--duration-base) var(--ease-out);
}

.page-leave-active {
  transition:
    opacity 160ms var(--ease-in),
    transform 160ms var(--ease-in);
}

.page-enter-from {
  opacity: 0;
  transform: translateY(8px);
}

.page-leave-to {
  opacity: 0;
  transform: translateY(-8px);
}

.title-swap-enter-active,
.title-swap-leave-active {
  transition:
    opacity 180ms var(--ease-out),
    transform 180ms var(--ease-out);
}

.title-swap-enter-from {
  opacity: 0;
  transform: translateY(6px);
}

.title-swap-leave-to {
  opacity: 0;
  transform: translateY(-6px);
}

.backdrop-enter-active,
.backdrop-leave-active {
  transition: opacity 280ms var(--ease-out);
}

.backdrop-enter-from,
.backdrop-leave-to {
  opacity: 0;
}

/* Motion: staggered entrance, scoped to the route entry window (.is-entering).
   Sections are the children of each view's content block (the unclassed v-else
   div). --enter-section and --enter-tile add up to the stagger step, capped at 8. */

.main-content > * > div:not([class]) > :nth-child(1) { --enter-section: 1; }
.main-content > * > div:not([class]) > :nth-child(2) { --enter-section: 2; }
.main-content > * > div:not([class]) > :nth-child(3) { --enter-section: 3; }
.main-content > * > div:not([class]) > :nth-child(4) { --enter-section: 4; }
.main-content > * > div:not([class]) > :nth-child(5) { --enter-section: 5; }
.main-content > * > div:not([class]) > :nth-child(6) { --enter-section: 6; }
.main-content > * > div:not([class]) > :nth-child(7) { --enter-section: 7; }
.main-content > * > div:not([class]) > :nth-child(n + 8) { --enter-section: 8; }

.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(1) { --enter-tile: 0; }
.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(2) { --enter-tile: 1; }
.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(3) { --enter-tile: 2; }
.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(4) { --enter-tile: 3; }
.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(5) { --enter-tile: 4; }
.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(6) { --enter-tile: 5; }
.main-content > * > div:not([class]) > * :is(.card, .stat-card, .kpi-card, .trend-card, .section-title):nth-child(n + 7) { --enter-tile: 6; }

.main-content.is-entering > * > .page-header {
  animation: rise-in var(--duration-enter) var(--ease-out) backwards;
}

.main-content.is-entering > * > div:not([class]) :is(.card, .stat-card, .kpi-card, .trend-card, .section-title) {
  --enter-delay: calc(min(var(--enter-section, 0) + var(--enter-tile, 0), 8) * var(--stagger-step));
  animation: rise-in var(--duration-enter) var(--ease-out) backwards;
  animation-delay: var(--enter-delay);
}

.main-content.is-entering > * > div:not([class]) tbody tr:nth-child(-n + 8) {
  animation: row-in var(--duration-enter) var(--ease-out) backwards;
  animation-delay: calc(var(--enter-delay, 0ms) + 120ms + var(--enter-row, 0) * 35ms);
}

.main-content > * > div:not([class]) tbody tr:nth-child(2) { --enter-row: 1; }
.main-content > * > div:not([class]) tbody tr:nth-child(3) { --enter-row: 2; }
.main-content > * > div:not([class]) tbody tr:nth-child(4) { --enter-row: 3; }
.main-content > * > div:not([class]) tbody tr:nth-child(5) { --enter-row: 4; }
.main-content > * > div:not([class]) tbody tr:nth-child(6) { --enter-row: 5; }
.main-content > * > div:not([class]) tbody tr:nth-child(7) { --enter-row: 6; }
.main-content > * > div:not([class]) tbody tr:nth-child(8) { --enter-row: 7; }

.main-content.is-entering .kpi-progress {
  transform-origin: left center;
  animation: grow-x 900ms var(--ease-out) backwards;
  animation-delay: calc(var(--enter-delay, 0ms) + 220ms);
}

@keyframes rise-in {
  from {
    opacity: 0;
    transform: translateY(14px) scale(0.985);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

@keyframes row-in {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

@keyframes grow-x {
  from {
    transform: scaleX(0);
  }
  to {
    transform: scaleX(1);
  }
}

@keyframes fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@media (max-width: 1024px) {
  :root {
    --content-padding: var(--space-5);
  }
}

@media (max-width: 767px) {
  :root {
    --content-padding: var(--space-4);
  }

  .sidebar {
    z-index: 200;
    width: min(280px, 84vw);
    transform: translateX(-100%);
    visibility: hidden;
    transition:
      transform 320ms var(--ease-in),
      visibility 0s linear 320ms;
  }

  .app.nav-open .sidebar {
    transform: none;
    visibility: visible;
    box-shadow: 24px 0 48px -12px rgb(15 23 42 / 0.45);
    transition:
      transform 380ms var(--ease-out),
      visibility 0s;
  }

  .sidebar-brand .sidebar-close {
    display: inline-flex;
  }

  .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: 150;
    background: rgb(15 23 42 / 0.45);
    -webkit-backdrop-filter: blur(4px);
    backdrop-filter: blur(4px);
  }

  .main-column {
    margin-left: 0;
  }

  .topbar-inner .menu-button {
    display: inline-flex;
  }

  .topbar-eyebrow,
  .topbar-separator,
  .topbar-actions .language-label,
  .topbar-actions .profile-name {
    display: none;
  }

  .topbar-actions {
    gap: var(--space-2);
  }

  .main-content {
    padding-top: var(--space-4);
  }

  .page-header h2 {
    font-size: 1.5rem;
  }

  .table-container {
    margin: 0 calc(-1 * var(--space-4));
    padding: 0 var(--space-4);
  }

  .card {
    padding: var(--space-4);
  }
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-delay: 0ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    transition-delay: 0ms !important;
    scroll-behavior: auto !important;
  }

  .stat-card:hover,
  .card.chart-card:hover,
  .kpi-card:hover,
  .trend-card:hover,
  .sidebar-brand:hover .brand-mark {
    transform: none;
  }
}
</style>
