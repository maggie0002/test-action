<template>
  <q-layout view="hHh LpR fFf">
    <q-header bordered reveal v-bind="qHeaderStyle.header">
      <q-toolbar>
        <q-btn
          class="q-mr-sm"
          size="12px"
          flat
          dense
          round
          icon="menu"
          aria-label="Menu"
          @click="isLeftDrawerOpen = !isLeftDrawerOpen"
        />

        <router-link v-if="qHeaderStyle.logo_white" to="/">
          <q-avatar square size="sm">
            <img :src="qHeaderStyle.logo_white" />
          </q-avatar>
        </router-link>

        <q-toolbar-title v-bind="qHeaderStyle.title">
          <div v-if="$q.screen.gt.xs">
            {{ $t('title') }}
          </div>
          <q-slide-transition v-else :duration="1000">
            <div v-if="deviceName" class="text-subtitle1">
              <q-icon
                v-if="$q.screen.gt.xs"
                class="q-mr-xs"
                name="local_offer"
              />{{ deviceName }}
            </div>
          </q-slide-transition>
        </q-toolbar-title>

        <q-slide-transition :duration="1000">
          <div
            v-if="deviceName && $q.screen.gt.xs"
            class="text-subtitle1 q-mr-md"
          >
            <q-icon class="q-mr-xs" name="local_offer" />{{ deviceName }}
          </div>
        </q-slide-transition>
        <q-separator size="2px" class="q-mr-sm" vertical inset />
        <q-btn
          size="12px"
          :loading="isChangingLang"
          icon="translate"
          round
          flat
          dense
        >
          <q-menu>
            <q-list style="min-width: 100px">
              <q-item
                v-for="language in localeOptions"
                :key="language.value"
                v-close-popup
                clickable
                @click="changeLang(language.value)"
              >
                <q-item-section>{{ language.label }}</q-item-section>
                <q-item-section v-if="language.value === locale" avatar>
                  <q-icon size="xs" name="check" />
                </q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </q-btn>

        <div>
          <Reboot />
        </div>

        <div>
          <Shutdown />
        </div>
      </q-toolbar>
    </q-header>

    <q-drawer v-model="isLeftDrawerOpen" show-if-above :width="210" bordered>
      <MenuItems
        v-for="link in menuItems"
        :key="link.label"
        v-bind="link"
        :active="link.path === currentLink"
        @click="currentLink = link.path"
      />
    </q-drawer>

    <q-page-container>
      <div>
        <q-slide-transition>
          <q-banner
            v-if="isNetworkDown"
            dense
            inline-actions
            class="text-white bg-negative"
          >
            {{ $t('system.network.network_connection_down') }}
            <template #action>
              <q-btn
                flat
                padding="0"
                :label="$t('general.close')"
                @click="isNetworkDown = false"
              />
            </template>
          </q-banner>
        </q-slide-transition>
        <q-slide-transition>
          <q-banner
            v-if="isNetworkUp"
            dense
            inline-actions
            class="text-white bg-positive"
          >
            {{ $t('system.network.network_connection_restored') }}
          </q-banner>
        </q-slide-transition>
      </div>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script lang="ts">
import { AxiosResponse } from 'axios'
import { loadLanguageAsync } from 'boot/i18n'
import Reboot from 'components/SystemReboot.vue'
import Shutdown from 'components/SystemShutdown.vue'
import MenuItems from 'components/MainLayoutMenuItems.vue'
import { supervisor } from 'src/api/supervisor'
import localeOptions from 'src/config/localeOptions'
import menuList from 'src/config/menuList'
import { qHeaderStyle } from 'src/config/qStyles'
import { defineComponent, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'

export default defineComponent({
  name: 'MainLayout',

  components: {
    MenuItems,
    Reboot,
    Shutdown
  },

  setup() {
    const { locale } = useI18n({ useScope: 'global' })
    const $router = useRouter()

    const deviceName = ref<string>()
    const isChangingLang = ref<boolean>(false)
    const isNetworkDown = ref<boolean>(false)
    const isNetworkUp = ref<boolean>(false)

    onMounted(() => {
      void getDeviceName()
    })

    // Import and activate language chosen from the selector in the header
    async function changeLang(isoName: string) {
      // Start loading indicator
      isChangingLang.value = true

      // Load the language and set it as the current language
      await loadLanguageAsync(isoName)

      isChangingLang.value = false
    }

    // Get device name for displaying in header
    async function getDeviceName() {
      const response = (await supervisor.v2.device_name()) as AxiosResponse<{
        deviceName: string
      }>
      deviceName.value = response.data.deviceName
    }

    // Listeners for when the system is connected to a Wifi network and remove
    // banner notification
    window.addEventListener('online', () => {
      isNetworkDown.value = false
      isNetworkUp.value = true
      setTimeout(() => {
        isNetworkUp.value = false
      }, 3000)
    })

    // Listeners for when the system is not connected to a Wifi network and display
    // banner notification
    window.addEventListener('offline', () => {
      isNetworkDown.value = true
    })

    return {
      changeLang,
      currentLink: ref($router.currentRoute.value.name),
      deviceName,
      isChangingLang,
      isNetworkDown,
      isNetworkUp,
      isLeftDrawerOpen: ref<boolean>(false),
      locale,
      localeOptions,
      menuItems: menuList,
      qHeaderStyle
    }
  }
})
</script>
