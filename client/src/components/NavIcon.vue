<template>
  <svg
    class="nav-icon"
    viewBox="0 0 24 24"
    :width="size"
    :height="size"
    fill="none"
    stroke="currentColor"
    stroke-width="1.75"
    stroke-linecap="round"
    stroke-linejoin="round"
    aria-hidden="true"
    focusable="false"
  >
    <path v-for="d in paths" :key="d" :d="d" />
  </svg>
</template>

<script>
import { computed } from 'vue'

const ICON_PATHS = {
  overview: [
    'M4 5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v5a1 1 0 0 1-1 1H5a1 1 0 0 1-1-1z',
    'M14 5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2a1 1 0 0 1-1 1h-4a1 1 0 0 1-1-1z',
    'M14 13a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v6a1 1 0 0 1-1 1h-4a1 1 0 0 1-1-1z',
    'M4 16a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v3a1 1 0 0 1-1 1H5a1 1 0 0 1-1-1z'
  ],
  inventory: [
    'M21 8l-9-5-9 5 9 5 9-5z',
    'M3 8v8l9 5 9-5V8',
    'M12 13v8',
    'M7.5 5.5l9 5'
  ],
  orders: [
    'M9 4h6a1 1 0 0 1 1 1v1a1 1 0 0 1-1 1H9a1 1 0 0 1-1-1V5a1 1 0 0 1 1-1z',
    'M16 5h2a2 2 0 0 1 2 2v12a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V7a2 2 0 0 1 2-2h2',
    'M9 12h6',
    'M9 16h4'
  ],
  finance: [
    'M12 3a9 9 0 1 0 0 18 9 9 0 0 0 0-18z',
    'M15 9.5c-.5-1-1.6-1.5-3-1.5-1.7 0-3 .8-3 2s1.3 1.7 3 2 3 .8 3 2-1.3 2-3 2c-1.4 0-2.5-.5-3-1.5',
    'M12 6.5V8',
    'M12 16v1.5'
  ],
  demand: [
    'M3 17l6-6 4 4 8-8',
    'M15 7h6v6'
  ],
  reports: [
    'M4 20h16',
    'M7 16v-4',
    'M12 16V7',
    'M17 16v-7'
  ],
  menu: [
    'M4 7h16',
    'M4 12h16',
    'M4 17h16'
  ],
  close: [
    'M6 6l12 12',
    'M18 6L6 18'
  ],
  logo: [
    'M12 2.75l8 4.5v9.5l-8 4.5-8-4.5v-9.5z',
    'M4 7.25l8 4.5 8-4.5',
    'M12 11.75v9.5'
  ]
}

export default {
  name: 'NavIcon',
  props: {
    name: {
      type: String,
      required: true,
      validator: (value) => Object.prototype.hasOwnProperty.call(ICON_PATHS, value)
    },
    size: {
      type: [Number, String],
      default: 20
    }
  },
  setup(props) {
    const paths = computed(() => ICON_PATHS[props.name] || [])

    return {
      paths
    }
  }
}
</script>

<style scoped>
.nav-icon {
  flex-shrink: 0;
  display: block;
}
</style>
