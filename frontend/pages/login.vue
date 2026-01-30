<template>
  <v-app>
    <v-main class="bg-grey-lighten-3">
      <v-container class="fill-height" fluid>
        <v-row align="center" justify="center">
          <v-col cols="12" sm="8" md="6" lg="4">

            <v-card class="elevation-12 rounded-xl pa-4">

              <div class="text-center my-4">
                <h2 class="text-h4 font-weight-bold text-primary">Welcome</h2>
                <p class="text-subtitle-1 text-grey">
                  {{ isLogin ? 'Login to system' : 'Register new user' }}
                </p>
              </div>

              <v-window v-model="step">

                <v-window-item :value="1">
                  <v-form ref="loginForm" @submit.prevent="handleLogin">
                    <v-text-field v-model="loginData.email" prepend-inner-icon="mdi-email" label="Email"
                      variant="outlined" class="mb-2" color="primary"></v-text-field>

                    <v-text-field v-model="loginData.password" prepend-inner-icon="mdi-lock" label="Password"
                      variant="outlined" :type="showPass ? 'text' : 'password'"
                      :append-inner-icon="showPass ? 'mdi-eye' : 'mdi-eye-off'"
                      @click:append-inner="showPass = !showPass" class="mb-2" color="primary"></v-text-field>

                    <v-btn block color="primary" size="large" class="rounded-lg font-weight-bold mt-4" elevation="4"
                      type="submit" :loading="loading">
                      Login
                    </v-btn>

                    <div class="text-center mt-6">
                      <span class="text-body-2 text-grey">Don't have an account? </span>
                      <a href="#" class="text-decoration-none text-primary font-weight-bold" @click.prevent="step = 2">
                        Register
                      </a>
                    </div>
                  </v-form>
                </v-window-item>

                <v-window-item :value="2">
                  <v-form ref="registerForm" @submit.prevent="handleRegister">

                    <v-text-field v-model="regData.email" prepend-inner-icon="mdi-email" label="Email"
                      variant="outlined" density="compact" class="mb-2" color="secondary"></v-text-field>

                    <v-text-field v-model="regData.password" prepend-inner-icon="mdi-lock" label="Password"
                      variant="outlined" density="compact" type="password" class="mb-2"
                      color="secondary"></v-text-field>

                    <v-text-field v-model="regData.name_th" prepend-inner-icon="mdi-account" label="Name (TH)"
                      variant="outlined" density="compact" class="mb-2" color="secondary"></v-text-field>

                    <v-select v-model="regData.department_id" :items="data_departments" item-title="name_th"
                      item-value="id" prepend-inner-icon="mdi-domain" label="Department" variant="outlined"
                      density="compact" class="mb-2" color="secondary" :loading="loadingData"></v-select>
                    <v-select v-model="regData.org_group_id" :items="org_groups" item-title="name_th" item-value="id"
                      prepend-inner-icon="mdi-office-building" label="Org Group" variant="outlined" density="compact"
                      class="mb-4" color="secondary" :loading="loadingData"></v-select>
                    <v-btn block color="secondary" size="large" class="rounded-lg font-weight-bold" elevation="4"
                      type="submit" :loading="loading">
                      Create User
                    </v-btn>

                    <div class="text-center mt-6">
                      <span class="text-body-2 text-grey">Back to login? </span>
                      <a href="#" class="text-decoration-none text-secondary font-weight-bold"
                        @click.prevent="step = 1">
                        Login
                      </a>
                    </div>
                  </v-form>
                </v-window-item>

              </v-window>
            </v-card>

            <v-snackbar v-model="snackbar.show" :color="snackbar.color" location="top">
              {{ snackbar.text }}
              <template v-slot:actions>
                <v-btn variant="text" @click="snackbar.show = false">Close</v-btn>
              </template>
            </v-snackbar>

          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup>
import { ref, computed, reactive, onMounted } from 'vue' // เพิ่ม onMounted
import axios from 'axios'
import { useAuthStore } from '~/stores/auth'

definePageMeta({
  layout: false
})

// --- State Variables ---
const step = ref(1)
const showPass = ref(false)
const loading = ref(false)
const loadingData = ref(false) // loading สำหรับตอนดึงข้อมูล Dropdown
const isLogin = computed(() => step.value === 1)
const authStore = useAuthStore() // ✅ 2. เรียกใช้
const router = useRouter()

// ตัวแปรสำหรับเก็บข้อมูล Dropdown
const data_departments = ref([])
const org_groups = ref([])

// ข้อมูลสำหรับ Login
const loginData = reactive({
  email: '',
  password: ''
})

// ข้อมูลสำหรับ Register
const regData = reactive({
  email: '',
  password: '',
  name_th: '',
  department_id: null, // เปลี่ยนค่าเริ่มต้นเป็น null เพื่อให้ Select แสดง label สวยๆ
  org_group_id: null
})

// แจ้งเตือน (Snackbar)
const snackbar = reactive({
  show: false,
  text: '',
  color: 'success'
})

// --- Functions ---

// ฟังก์ชันดึงข้อมูล Dropdown (ทำงานเมื่อเปิดหน้าเว็บ)
const fetchDropdownData = async () => {
  loadingData.value = true
  try {
    // ยิงพร้อมกัน 2 API เพื่อความเร็ว (ใส่ path ให้ตรงกับ Backend ของนาย)
    const [deptRes, orgRes] = await Promise.all([
      axios.get('http://localhost:7000/api/users/departments_show_all'),
      axios.get('http://localhost:7000/api/users/org_groups_show_all')
    ])

    // รับค่าลงตัวแปร
    data_departments.value = deptRes.data.data_departments
    org_groups.value = orgRes.data.data_org_groups

    console.log('Departments loaded:', data_departments.value)
    console.log('Org Groups loaded:', org_groups.value)

  } catch (error) {
    console.error('Error fetching dropdown data:', error)
    showSnackbar('ไม่สามารถดึงข้อมูลแผนกหรือกลุ่มงานได้', 'error')
  } finally {
    loadingData.value = false
  }
}

// เรียกใช้ function ดึงข้อมูลตอน Component ถูก Mount
onMounted(() => {
  fetchDropdownData()
})

// 1. ฟังก์ชัน Login
const handleLogin = async () => {
  loading.value = true
  try {
    const response = await axios.post('http://localhost:7000/api/auth/login', {
      email: loginData.email,
      password: loginData.password
    })

    authStore.setAuth(response.data.token, response.data.user)

    showSnackbar('Login Successful!', 'success')

    // เช็ค Role
    const role = authStore.user?.role // หรือ response.data.user.role

    // ✅ แก้ไขเงื่อนไขให้เป็น if - else if - else
    if (role === 'admin') {
      await router.push('/') // ใช้ await เพื่อความชัวร์
    } else if (role === 'evaluatee') {
      await router.push('/EvidenceSubmission')  
    }  else if (role === 'evaluator') {
      await router.push('/evaluation_index') 
    }else {
      // กรณี Login ผ่านแต่ไม่มี Role หรือเป็น Role อื่น
      // ไม่ควรส่งกลับไปหน้า Login ซ้ำ (เพราะ Login ผ่านแล้ว) อาจจะไปหน้าแรกแทน
      await router.push('/')
    }
  } catch (error) {
    console.error('Login Error:', error)
    showSnackbar('Login Failed: ' + (error.response?.data?.message || error.message), 'error')
  } finally {
    loading.value = false
  }
}

// 2. ฟังก์ชัน Register
const handleRegister = async () => {
  loading.value = true
  try {
    const payload = {
      email: regData.email,
      password: regData.password,
      name_th: regData.name_th,
      department_id: regData.department_id,
      org_group_id: regData.org_group_id
    }

    const response = await axios.post('http://localhost:7000/api/users/create', payload)

    console.log('Register Success:', response.data)
    showSnackbar('User Created Successfully!', 'success')

    setTimeout(() => {
      step.value = 1
    }, 1500)

  } catch (error) {
    console.error('Register Error:', error)
    showSnackbar('Registration Failed: ' + (error.response?.data?.message || error.message), 'error')
  } finally {
    loading.value = false
  }
}

const showSnackbar = (text, color) => {
  snackbar.text = text
  snackbar.color = color
  snackbar.show = true
}
</script>