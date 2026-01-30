<template>
  <div class="pa-4">
    <v-card elevation="4" rounded="lg">
      <v-data-table :headers="headers" :items="users" :loading="loading" hover density="comfortable">
        <template v-slot:top>
          <v-toolbar color="indigo-darken-1" flat rounded="t-lg">
            <v-toolbar-title class="font-weight-bold">
              <v-icon icon="mdi-account-group" start></v-icon>
              รายชื่อผู้ใช้งาน (Admin List)
            </v-toolbar-title>

            <v-spacer></v-spacer>

            <v-btn class="me-4 text-none" color="white" variant="flat" prepend-icon="mdi-plus-circle" rounded="pill"
              text="Add New User" @click="add"></v-btn>
          </v-toolbar>
        </template>

        <template v-slot:item.status="{ value }">
          <v-chip :color="value === 'active' ? 'success' : 'grey'" :text="value" class="text-uppercase font-weight-bold"
            size="small" variant="flat" prepend-icon="mdi-circle-medium">
          </v-chip>
        </template>

        <template v-slot:item.created_at="{ value }">
          {{ new Date(value).toLocaleDateString('th-TH') }}
        </template>

        <template v-slot:item.actions="{ item }">
          <div class="d-flex ga-2 justify-end">
            <v-btn icon="mdi-pencil" size="small" color="amber-darken-2" variant="text" @click="edit(item)"></v-btn>
            <v-btn icon="mdi-delete" size="small" color="red-lighten-1" variant="text" @click="remove(item.id)"></v-btn>
          </div>
        </template>

        <template v-slot:no-data>
          <div class="pa-4 text-center">
            <v-icon icon="mdi-alert-circle-outline" size="large" color="grey-lighten-1" class="mb-2"></v-icon>
            <div class="text-medium-emphasis mb-2">ไม่พบข้อมูลในระบบ</div>
            <v-btn prepend-icon="mdi-refresh" rounded="pill" color="indigo" variant="outlined" @click="fetchData">
              Reload Data
            </v-btn>
          </div>
        </template>
      </v-data-table>
    </v-card>

    <v-dialog v-model="dialog" max-width="500" transition="dialog-bottom-transition">
      <v-card rounded="xl">
        <v-toolbar :color="isEditing ? 'amber-darken-1' : 'indigo'" theme="dark">
          <v-toolbar-title class="text-h6 font-weight-bold">
            <v-icon :icon="isEditing ? 'mdi-pencil-box' : 'mdi-account-plus'" start></v-icon>
            {{ isEditing ? 'Edit User' : 'New User' }}
          </v-toolbar-title>
          <v-btn icon="mdi-close" variant="text" @click="dialog = false"></v-btn>
        </v-toolbar>

        <v-card-text class="pa-6">
          <v-row dense>
            <v-col cols="12">
              <v-text-field v-model="formModel.email" label="Email Address" prepend-inner-icon="mdi-email-outline"
                variant="outlined" color="indigo" density="comfortable"></v-text-field>
            </v-col>

            <v-col cols="12" md="6">
              <v-select v-model="formModel.status" label="Status" :items="['active', 'disabled']"
                prepend-inner-icon="mdi-toggle-switch-outline" variant="outlined" color="indigo"
                density="comfortable"></v-select>
            </v-col>

            <v-col cols="12" md="6">
              <v-select v-model="formModel.role" label="Role" :items="['admin', 'evaluatee', 'evaluator']"
                prepend-inner-icon="mdi-badge-account-outline" variant="outlined" color="indigo"
                density="comfortable"></v-select>
            </v-col>

            <v-col cols="12" md="6">
              <v-select v-model="formModel.department_id" label="สังกัดแผนก (Department)" :items="departmentList"
                item-title="name_th" item-value="id" prepend-inner-icon="mdi-domain" variant="outlined" color="indigo"
                density="comfortable" clearable></v-select>
            </v-col>

            <v-col cols="12" md="6">
              <v-select v-model="formModel.org_group_id" label="กลุ่มงาน (Org Group)" :items="orgGroupList"
                item-title="name_th" item-value="id" prepend-inner-icon="mdi-office-building-cog" variant="outlined"
                color="indigo" density="comfortable" clearable></v-select>
            </v-col>
            <v-col cols="12">
              <v-text-field v-model="formModel.name_th" label="ชื่อภาษาไทย" prepend-inner-icon="mdi-account-outline"
                variant="outlined" color="indigo" density="comfortable"></v-text-field>
            </v-col>

            <v-col cols="12">
              <v-text-field v-model="formModel.password_hash" label="Password" :type="visible ? 'text' : 'password'"
                :append-inner-icon="visible ? 'mdi-eye-off' : 'mdi-eye'" @click:append-inner="visible = !visible"
                prepend-inner-icon="mdi-lock-outline" variant="outlined" color="indigo"
                :hint="isEditing ? 'ปล่อยว่างหากไม่ต้องการเปลี่ยนรหัสผ่าน' : 'กำหนดรหัสผ่านเริ่มต้น'" persistent-hint
                density="comfortable"></v-text-field>
            </v-col>
          </v-row>
        </v-card-text>

        <v-divider></v-divider>

        <v-card-actions class="pa-4 bg-grey-lighten-5">
          <v-btn text="Cancel" variant="text" color="grey-darken-1" @click="dialog = false"></v-btn>
          <v-spacer></v-spacer>
          <v-btn :text="isEditing ? 'Update Changes' : 'Save User'"
            :color="isEditing ? 'amber-darken-3' : 'indigo-darken-1'" variant="elevated" rounded="pill" class="px-6"
            @click="save"></v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { onMounted, ref, computed } from 'vue'
import { useRouter } from 'vue-router' // ใช้สำหรับ Redirect
import axios from 'axios'

// --- 1. Setup & Auth Check ---
const router = useRouter()
const users = ref([]) // เปลี่ยนชื่อจาก categories เป็น users ให้สื่อความหมาย
const loading = ref(false)
const dialog = ref(false)
const visible = ref(false)
// ✅ เพิ่มตัวแปรเก็บลิสต์สำหรับ Dropdown
const departmentList = ref([])
const orgGroupList = ref([])

// Headers Table
const headers = [
  { title: 'ID', key: 'id', align: 'start', width: '80px' },
  { title: 'EMAIL', key: 'email' },
  { title: 'ชื่อไทย', key: 'name_th' },
  { title: 'ROLE', key: 'role', align: 'center' }, // เพิ่ม Role
  { title: 'DEPT', key: 'department_id', align: 'center' }, // ✅ เพิ่ม
  { title: 'ORG', key: 'org_group_id', align: 'center' },   // ✅ เพิ่ม
  { title: 'STATUS', key: 'status', align: 'center' },
  { title: 'CREATED', key: 'created_at', align: 'end' },
  { title: 'ACTIONS', key: 'actions', align: 'end', sortable: false },
]

// Form Model
const formModel = ref({
  id: null,
  email: '',
  name_th: '',
  password_hash: '',
  status: 'active',
  role: 'evaluatee',
  department_id: null, // ✅ เพิ่ม
  org_group_id: null   // ✅ เพิ่ม
})
const isEditing = computed(() => !!formModel.value.id)

function resetForm() {
  formModel.value = {
    id: null,
    email: '',
    name_th: '',
    password_hash: '',
    status: 'active',
    role: 'evaluatee',
    department_id: null, // ✅ เพิ่ม
    org_group_id: null   // ✅ เพิ่ม
  }
}

// --- 2. ฟังก์ชันตรวจสอบสิทธิ์ (Admin Guard) ---
async function checkAuth() {
  const token = localStorage.getItem('token');
  const userStr = localStorage.getItem('user_info');

  if (!token) {
    alert('กรุณาเข้าสู่ระบบก่อน');
    router.push('/login');
    return false;
  }

  // เช็ค Role เบื้องต้นจาก LocalStorage (และเดี๋ยว Backend จะเช็ค Token อีกทีตอนยิง API)
  if (userStr) {
    const user = JSON.parse(userStr);
    if (user.role !== 'admin') {
      alert('คุณไม่มีสิทธิ์เข้าถึงหน้านี้ (Admin Only)');
      router.push('/'); // ดีดกลับหน้าแรก
      return false;
    }
  }
  return true;
}

// --- 3. ฟังก์ชันดึงข้อมูล ---
const fetchData = async () => {
  const isAuth = await checkAuth();
  if (!isAuth) return;

  loading.value = true;
  try {
    const token = localStorage.getItem('token'); // ดึง Token
    const { data } = await axios.get("http://localhost:7000/api/admin/users", {
      headers: { Authorization: `Bearer ${token}` }, // ใส่ Token ใน Header
    });
    users.value = data;
  } catch (error) {
    console.error("Error fetching data:", error);
    if (error.response && error.response.status === 403) {
      alert('Session หมดอายุหรือไม่มีสิทธิ์');
      router.push('/login');
    }
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  checkAuth().then(isAuth => {
      if (isAuth) {
          fetchData();
          fetchMasterData(); // ✅ เรียกโหลดข้อมูล Dropdown ตอนเปิดหน้า
      }
  });
})

// --- 4. จัดการ Events ---

function add() {
  resetForm();
  dialog.value = true;
}

function edit(item) {
  // Clone object เพื่อไม่ให้กระทบตารางตอนพิมพ์
  formModel.value = { ...item, password_hash: '' };
  dialog.value = true;
}

async function remove(id) {
  if (!confirm("แน่ใจนะว่าจะลบผู้ใช้นี้?")) return;

  try {
    const token = localStorage.getItem('token');
    await axios.delete(`http://localhost:7000/api/admin/users/${id}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    await fetchData(); // โหลดข้อมูลใหม่
  } catch (error) {
    console.error("Delete error:", error);
    alert('ลบไม่สำเร็จ: ' + error.message);
  }
}

async function save() {
  try {
    const token = localStorage.getItem('token');
    const headers = { Authorization: `Bearer ${token}` };

    // API URL แบบ RESTful
    const baseUrl = "http://localhost:7000/api/admin/users";

    if (isEditing.value) {
      // Update (PUT)
      await axios.put(`${baseUrl}/${formModel.value.id}`, formModel.value, { headers });
    } else {
      // Create (POST)
      await axios.post(baseUrl, formModel.value, { headers });
    }

    dialog.value = false;
    await fetchData();

  } catch (error) {
    console.error("Save error:", error);
    alert('บันทึกไม่สำเร็จ: ' + (error.response?.data?.message || error.message));
  }
}

// ✅ ฟังก์ชันดึงข้อมูล Master Data (แผนก/กลุ่มงาน)
const fetchMasterData = async () => {
    try {
        const token = localStorage.getItem('token');
        // ยิงไปขอรายชื่อแผนกและกลุ่มงาน
        const [deptRes, orgRes] = await Promise.all([
            axios.get("http://localhost:7000/api/admin/departments", { headers: { Authorization: `Bearer ${token}` } }),
            axios.get("http://localhost:7000/api/admin/org-groups", { headers: { Authorization: `Bearer ${token}` } })
        ]);
        
        departmentList.value = deptRes.data;
        orgGroupList.value = orgRes.data;
    } catch (error) {
        console.error("Error fetching master data:", error);
    }
}

definePageMeta({
  middleware: 'auth'
})
</script>