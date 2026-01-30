<script setup>
import { ref, onMounted, computed } from 'vue' // ✅ เพิ่ม computed
import { useRouter } from 'vue-router'
import { useMenu } from '~/composables/useMenu'
import { useAuthStore } from '~/stores/auth'   // ✅ เพิ่มบรรทัดนี้

const router = useRouter()
const drawer = ref(true)
const authStore = useAuthStore() // ✅ เพิ่มบรรทัดนี้ ไม่งั้นใช้ authStore ไม่ได้

// ดึง Role จาก Store
const role = computed(() => authStore.user?.role || '')

// เรียกใช้ Composable
const { menu } = useMenu(role)

// ข้อมูลโปรไฟล์
const userProfile = computed(() => ({
  name: authStore.user?.name_th || 'Guest',
  email: authStore.user?.email || '',
  initials: (authStore.user?.name_th || 'G').charAt(0)
}))

onMounted(() => {
  authStore.hydrateFromStorage()
})

// ฟังก์ชัน Logout (ใช้ของ Store ดีกว่า Manual)
const logout = () => {
  authStore.logout()
  router.push('/login')
}
</script>

<template>
  <v-app>
    <v-navigation-drawer v-model="drawer" width="280" elevation="0" class="border-e">
      
      <div class="pa-4 d-flex align-center">
        <v-avatar color="primary" class="mr-3" rounded="lg">
          <v-icon icon="mdi-shield-account" color="white"></v-icon>
        </v-avatar>
        <div>
          <div class="text-subtitle-1 font-weight-bold">VEC Evaluation</div>
          <div class="text-caption text-medium-emphasis">System v1.0</div>
        </div>
      </div>

      <v-divider></v-divider>

      <v-list density="comfortable" nav class="mt-2">
        <template v-for="(section, i) in menu" :key="i">
          <div v-if="section.label" class="px-4 py-2 text-caption text-uppercase text-medium-emphasis font-weight-bold mt-2">
            {{ section.label }}
          </div>
          
          <v-list-item
            v-for="(it, j) in section.items"
            :key="j"
            :to="it.to"
            :prepend-icon="it.icon"
            :title="it.label"
            active-color="primary"
            rounded="lg"
            class="mb-1"
          />
        </template>
      </v-list>

      <template v-slot:append>
        <div class="pa-4">
          <v-btn block color="error" variant="tonal" prepend-icon="mdi-logout" @click="logout">
            ออกจากระบบ
          </v-btn>
        </div>
      </template>
    </v-navigation-drawer>

    <v-app-bar color="white" elevation="1" density="comfortable">
      <v-app-bar-nav-icon @click="drawer = !drawer" color="grey-darken-1" />
      
      <v-toolbar-title class="text-body-1 font-weight-bold text-grey-darken-2">
        ระบบประเมินบุคลากร
      </v-toolbar-title>

      <v-spacer />

      <v-menu location="bottom end" transition="scale-transition">
        <template v-slot:activator="{ props }">
          <v-btn icon v-bind="props" class="ml-2 mr-2">
            <v-avatar color="primary" size="36">
              <span class="text-caption font-weight-bold white--text">{{ userProfile.initials }}</span>
            </v-avatar>
          </v-btn>
        </template>
        <v-list width="200" elevation="2" rounded="lg">
          <v-list-item :subtitle="userProfile.email" :title="userProfile.name">
            <template v-slot:prepend>
              <v-avatar color="primary" size="24" class="mr-2">
                <span class="text-caption">{{ userProfile.initials }}</span>
              </v-avatar>
            </template>
          </v-list-item>
          <v-divider class="my-2"></v-divider>
          <v-list-item prepend-icon="mdi-logout" title="ออกจากระบบ" value="logout" color="error" @click="logout"></v-list-item>
        </v-list>
      </v-menu>
    </v-app-bar>

    <v-main class="bg-grey-lighten-4">
      <v-container fluid class="py-6 px-6">
        <slot /> 
      </v-container>
      
      <v-footer color="transparent" class="text-caption justify-center text-medium-emphasis mt-4">
        © 2025 VEC Skills — Evaluation System
      </v-footer>
    </v-main>
  </v-app>
</template>