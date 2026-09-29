<template>
  <aside class="sidebar" :class="{ collapsed }">
    <div class="sidebar-header">
      <div v-if="!collapsed" class="brand">
        <h1>{{ t('nav.companyName') }}</h1>
        <span class="subtitle">{{ t('nav.subtitle') }}</span>
      </div>
      <div v-else class="brand-avatar" :title="t('nav.companyName')">CC</div>

      <button
        v-if="canToggle"
        class="collapse-toggle"
        :class="{ 'collapse-toggle-collapsed': collapsed }"
        :title="collapsed ? t('nav.expandSidebar') : t('nav.collapseSidebar')"
        @click="toggleCollapsed"
      >
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="11 17 6 12 11 7"></polyline>
          <polyline points="18 17 13 12 18 7"></polyline>
        </svg>
      </button>
    </div>

    <div class="nav-scroll">
      <nav class="nav-tabs">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          :class="{ active: $route.path === item.path }"
          :title="collapsed ? item.label : null"
        >
          <span class="nav-icon" v-html="item.icon"></span>
          <span v-if="!collapsed" class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>
    </div>

    <div class="sidebar-footer">
      <LanguageSwitcher :collapsed="collapsed" />
      <ProfileMenu
        :collapsed="collapsed"
        @show-profile-details="$emit('show-profile-details')"
        @show-tasks="$emit('show-tasks')"
      />
    </div>
  </aside>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useI18n } from '../composables/useI18n'
import LanguageSwitcher from './LanguageSwitcher.vue'
import ProfileMenu from './ProfileMenu.vue'

defineEmits(['show-profile-details', 'show-tasks'])

const { t } = useI18n()

const STORAGE_KEY = 'sidebar-collapsed'
const manuallyCollapsed = ref(false)
const isSmallScreen = ref(false)
let mediaQuery = null

const handleMediaChange = (e) => {
  isSmallScreen.value = e.matches
}

onMounted(() => {
  const stored = localStorage.getItem(STORAGE_KEY)
  manuallyCollapsed.value = stored === 'true'

  mediaQuery = window.matchMedia('(max-width: 1024px)')
  isSmallScreen.value = mediaQuery.matches
  mediaQuery.addEventListener('change', handleMediaChange)
})

onUnmounted(() => {
  if (mediaQuery) {
    mediaQuery.removeEventListener('change', handleMediaChange)
  }
})

const collapsed = computed(() => isSmallScreen.value || manuallyCollapsed.value)
const canToggle = computed(() => !isSmallScreen.value)

const toggleCollapsed = () => {
  manuallyCollapsed.value = !manuallyCollapsed.value
  localStorage.setItem(STORAGE_KEY, String(manuallyCollapsed.value))
}

const svgWrap = (inner) =>
  `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">${inner}</svg>`

const navItems = computed(() => [
  {
    path: '/',
    label: t('nav.overview'),
    icon: svgWrap('<path d="M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"></path><polyline points="9 22 9 12 15 12 15 22"></polyline>')
  },
  {
    path: '/inventory',
    label: t('nav.inventory'),
    icon: svgWrap('<path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z"></path><polyline points="3.27 6.96 12 12.01 20.73 6.96"></polyline><line x1="12" y1="22.08" x2="12" y2="12"></line>')
  },
  {
    path: '/orders',
    label: t('nav.orders'),
    icon: svgWrap('<circle cx="9" cy="21" r="1"></circle><circle cx="20" cy="21" r="1"></circle><path d="M1 1h4l2.68 13.39a2 2 0 0 0 2 1.61h9.72a2 2 0 0 0 2-1.61L23 6H6"></path>')
  },
  {
    path: '/spending',
    label: t('nav.finance'),
    icon: svgWrap('<line x1="12" y1="1" x2="12" y2="23"></line><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"></path>')
  },
  {
    path: '/demand',
    label: t('nav.demandForecast'),
    icon: svgWrap('<polyline points="23 6 13.5 15.5 8.5 10.5 1 18"></polyline><polyline points="17 6 23 6 23 12"></polyline>')
  },
  {
    path: '/restocking',
    label: t('nav.restocking'),
    icon: svgWrap('<polyline points="23 4 23 10 17 10"></polyline><polyline points="1 20 1 14 7 14"></polyline><path d="M20.49 9A9 9 0 0 0 5.64 5.64L1 10m22 4l-4.64 4.36A9 9 0 0 1 3.51 15"></path>')
  },
  {
    path: '/reports',
    label: t('nav.reports'),
    icon: svgWrap('<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><line x1="16" y1="13" x2="8" y2="13"></line><line x1="16" y1="17" x2="8" y2="17"></line><polyline points="10 9 9 9 8 9"></polyline>')
  }
])
</script>

<style scoped>
.sidebar {
  width: 240px;
  flex-shrink: 0;
  background: #ffffff;
  border-right: 1px solid #e2e8f0;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.05);
  position: sticky;
  top: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  z-index: 100;
  transition: width 0.2s ease;
}

.sidebar.collapsed {
  width: 72px;
}

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  padding: 1.25rem 1rem;
  border-bottom: 1px solid #e2e8f0;
}

.sidebar.collapsed .sidebar-header {
  flex-direction: column;
  padding: 1rem 0.5rem;
}

.brand {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.brand h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.brand .subtitle {
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 400;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.brand-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: linear-gradient(135deg, #2563eb 0%, #1e40af 100%);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 0.813rem;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.collapse-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  padding: 0;
  background: none;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  color: #64748b;
  cursor: pointer;
  transition: all 0.2s ease;
  flex-shrink: 0;
}

.collapse-toggle:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.collapse-toggle svg {
  transition: transform 0.2s ease;
}

.collapse-toggle-collapsed svg {
  transform: rotate(180deg);
}

.nav-scroll {
  overflow-y: auto;
  flex: 1;
}

.nav-tabs {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 1rem 0.75rem;
}

.sidebar.collapsed .nav-tabs {
  padding: 1rem 0.5rem;
  align-items: center;
}

.nav-tabs a {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 0.875rem;
  color: #64748b;
  text-decoration: none;
  font-weight: 500;
  font-size: 0.938rem;
  border-radius: 6px;
  border-left: 3px solid transparent;
  transition: all 0.2s ease;
  position: relative;
  white-space: nowrap;
}

.sidebar.collapsed .nav-tabs a {
  padding: 0.625rem;
  justify-content: center;
  width: 44px;
}

.nav-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

.nav-icon :deep(svg) {
  width: 20px;
  height: 20px;
}

.nav-label {
  overflow: hidden;
  text-overflow: ellipsis;
}

.nav-tabs a:hover {
  color: #0f172a;
  background: #f1f5f9;
}

.nav-tabs a.active {
  color: #2563eb;
  background: #eff6ff;
  border-left-color: #2563eb;
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 1rem 0.75rem;
  border-top: 1px solid #e2e8f0;
}

.sidebar.collapsed .sidebar-footer {
  align-items: center;
  padding: 1rem 0.5rem;
}
</style>
