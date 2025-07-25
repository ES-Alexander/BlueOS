<template>
  <v-container>
    <v-row
      class="mb-6 mt-6"
      justify="center"
      no-gutters
    >
      <v-alert
        v-if="!current_network"
        border="top"
        colored-border
        type="info"
        elevation="2"
        dismissible
      >
        <h3>Welcome to BlueOS!</h3>
        Before you start, we highly recommend <a
          href="https://docs.bluerobotics.com/ardusub-zola/software/onboard/BlueOS-1.0/getting-started/#connect-wifi"
          target="_blank"
        >
          connecting to the internet
        </a>
        and performing a <a
          href="https://docs.bluerobotics.com/ardusub-zola/software/onboard/BlueOS-1.0/getting-started/#select-version"
          target="_blank"
        >
          system update to the latest available BlueOS version
        </a>
        .
      </v-alert>
      <self-health-test />
    </v-row>
    <div class="grid-holder">
      <div class="app-grid">
        <div
          v-for="({
            icon, title, text, route, component: Component, props, style,
          }, i) in apps"
          :key="i"
          class="app-grid-item"
          :style="{
            'grid-column': `span ${Math.max(1, Math.ceil((apps[i].size?.w || 1) * 12))}`,
            'grid-row': `span ${Math.max(1, Math.ceil((apps[i].size?.h || 1) * 4))}`,
          }"
        >
          <v-card
            class="px-3 rounded-xl app-card d-flex flex-column"
            :href="route"
          >
            <div class="card-header">
              <v-theme-provider dark>
                <v-row
                  class="py-3 px-3 d-flex justify-space-between flex-nowrap"
                >
                  <v-card-title
                    class="text-subtitle-2 font-weight-bold"
                    style="word-break: break-word;"
                    v-text="title"
                  />
                  <v-avatar
                    class="mt-2"
                    color="primary"
                    size="30"
                  >
                    <v-icon
                      v-text="icon"
                    />
                  </v-avatar>
                </v-row>
              </v-theme-provider>
              <v-card-text
                class="subtitle-1 text-justify-start text-clamp pt-0 pb-0"
                v-text="text"
              />
            </div>
            <div class="flex-grow-1 component-container">
              <component
                :is="Component"
                v-bind="props || {}"
                :style="style || {}"
              />
            </div>
          </v-card>
        </div>
      </div>
    </div>
  </v-container>
</template>

<script lang="ts">
import Vue from 'vue'

import SelfHealthTest from '@/components/health/SelfHealthTest.vue'
import GenericViewer from '@/components/vehiclesetup/viewers/GenericViewer.vue'
import mavlink from '@/store/mavlink'
import wifi from '@/store/wifi'
import { Network } from '@/types/wifi'
import mavlink_store_get from '@/utils/mavlink'
import CPUUsage from '@/widgets/CpuPie.vue'
import Networking from '@/widgets/Networking.vue'

interface AppItem {
  icon?: string
  title?: string
  text?: string
  route?: string
  component: typeof Vue
  size?: {
    w: number
    h: number
  }
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  props?: Record<string, any>
  style?: Record<string, string>
}

export default Vue.extend({
  name: 'MainView',
  components: {
    SelfHealthTest,
    GenericViewer,
  },
  data: () => ({
    windowHeight: window.innerHeight,
    windowWidth: window.innerWidth,
  }),
  computed: {
    apps(): AppItem[] {
      return [
        {
          icon: 'mdi-view-dashboard',
          title: 'Digital Twin',
          route: '#',
          component: GenericViewer,
          size: {
            w: 0.4,
            h: 1.2,
          },
          style: {
            transform: 'translateY(-45%)',
            maxHeight: '300px',
          },
          props: {
            autorotate: false,
            cameracontrols: false,
            orientation: this.orientation,
          },
        },
        {
          icon: 'mdi-cpu-64-bit',
          title: 'System Information',
          component: CPUUsage,
          size: {
            w: 0.4,
            h: 0.8,
          },
          props: {
            mode: 'graph',
          },
        },
        {
          icon: 'mdi-ethernet',
          title: 'Ethernet Status',
          component: Networking,
          size: {
            w: 0.1,
            h: 0.4,
          },
          props: {
            interface: 'eth0',
          },
        },
        {
          icon: 'mdi-wifi',
          title: 'WiFi Status',
          component: Networking,
          size: {
            w: 0.1,
            h: 0.4,
          },
          props: {
            interface: 'wlan0',
          },
        },
      ]
    },
    current_network(): Network | null {
      return wifi.current_network
    },
    orientation(): string {
      const msg = mavlink_store_get(mavlink, 'ATTITUDE.messageData.message') as
        { roll: number; pitch: number; yaw: number } | undefined
      if (!msg) return '0deg 0deg 0deg'
      return `${msg.roll}rad ${-msg.pitch}rad ${-msg.yaw}rad`
    },
  },
  mounted() {
    window.addEventListener('resize', this.handleResize)
    this.handleResize()
    mavlink.setMessageRefreshRate({ messageName: 'ATTITUDE', refreshRate: 10 })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
  },
  methods: {
    handleResize() {
      this.windowHeight = window.innerHeight
      this.windowWidth = window.innerWidth
    },
  },
})
</script>

<style scoped>
.rounded-card {
  border-radius: 50px;
}

div.pirate-marker {
  position: absolute;
  width: 35px;
  right: 0;
  top: 0px;
  opacity: 0.7;
}

div.pirate-marker.v-icon {
  font-size: 10px;
}

.text-clamp {
  display: -webkit-box;
  -webkit-line-clamp: 4;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.grid-holder {
  padding: 10px;
}

.app-grid {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  grid-auto-rows: 60px;
  gap: 8px;
  width: 100%;
}

@media (max-width: 1280px) {
  .app-grid {
    grid-template-columns: repeat(10, 1fr);
  }
}

@media (max-width: 960px) {
  .app-grid {
    grid-template-columns: repeat(5, 1fr);
  }
}

.app-grid-item {
  min-width: 0;
  min-height: 0;
  width: 100%;
  height: 100%;
}

.app-card {
  width: 100%;
  height: 100%;
  transition: transform 0.3s;
  display: flex;
  flex-direction: column;
}

.app-card:hover {
  transform: translateY(-5px);
}

.card-header {
  flex-shrink: 0;
}

.component-container {
  flex: 1;
  min-height: 0;
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: column;
}

.component-container > * {
  flex: 1;
  min-height: 0;
  height: 100%;
}

@media (max-width: 677px) {
  .app-grid {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .app-grid-item {
    width: 100%;
    max-width: 280px;
  }
}
</style>
