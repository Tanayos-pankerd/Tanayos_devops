<template>
  <v-container>
    <v-row class="mb-4" align="center">
      <v-col>
        <h2>📜 ประวัติการใช้งาน (History Logs)</h2>
      </v-col>
      
      <v-col cols="auto" v-if="isAdmin">
        <v-btn 
          :color="isDeleteMode ? 'grey' : 'error'" 
          variant="tonal"
          prepend-icon="mdi-broom"
          @click="toggleDeleteMode"
        >
          {{ isDeleteMode ? 'ยกเลิกการลบ' : 'ล้างประวัติ' }}
        </v-btn>
      </v-col>
    </v-row>

    <v-card elevation="2" class="rounded-lg">
      <v-data-table
        v-model="selected"
        :headers="headers"
        :items="historyItems"
        item-value="id"
        :show-select="isDeleteMode" 
        :hover="true"
      >
        <template v-slot:item.status="{ item }">
          <v-chip :color="getStatusColor(item.status)" size="small">
            {{ item.status }}
          </v-chip>
        </template>
      </v-data-table>
    </v-card>

    <v-slide-y-reverse-transition>
      <div v-if="selected.length > 0" class="floating-action-bar">
        <v-card elevation="8" class="d-flex align-center px-6 py-3 rounded-pill bg-grey-darken-4">
          <span class="text-subtitle-1 font-weight-bold mr-4">
            เลือกอยู่ {{ selected.length }} รายการ
          </span>
          
          <v-spacer></v-spacer>

          <v-btn 
            color="red-accent-2" 
            variant="flat" 
            prepend-icon="mdi-delete-forever"
            @click="confirmDelete"
          >
            ลบข้อมูลที่เลือก
          </v-btn>
        </v-card>
      </div>
    </v-slide-y-reverse-transition>

  </v-container>
</template>

<script setup>
import { ref } from 'vue'

// --- Mock Data & Config ---
const isAdmin = ref(true) // สมมติว่าเป็น Admin (ของจริงเช็คจาก Store/Pinia)
const isDeleteMode = ref(false) // ตัวคุมโหมดลบ
const selected = ref([]) // เก็บ ID รายการที่ถูกติ๊ก

// ข้อมูลตัวอย่าง
const headers = [
  { title: 'กิจกรรม', key: 'action', align: 'start' },
  { title: 'ผู้ใช้งาน', key: 'user' },
  { title: 'เวลา', key: 'timestamp' },
  { title: 'สถานะ', key: 'status' },
]

const historyItems = ref([
  { id: 1, action: 'Login เข้าระบบ', user: 'User001', timestamp: '10:00 AM', status: 'Success' },
  { id: 2, action: 'จองโต๊ะ A1', user: 'User002', timestamp: '10:05 AM', status: 'Pending' },
  { id: 3, action: 'ยกเลิกการจอง', user: 'User001', timestamp: '10:15 AM', status: 'Failed' },
  { id: 4, action: 'ชำระเงิน', user: 'User003', timestamp: '10:20 AM', status: 'Success' },
  { id: 5, action: 'Logout', user: 'User002', timestamp: '10:30 AM', status: 'Success' },
])

// --- Functions ---

const toggleDeleteMode = () => {
  isDeleteMode.value = !isDeleteMode.value
  selected.value = [] // เคลียร์ที่เลือกไว้ตอนเปลี่ยนโหมด
}

const getStatusColor = (status) => {
  if (status === 'Success') return 'green'
  if (status === 'Failed') return 'red'
  return 'orange'
}

const confirmDelete = () => {
  // ตรงนี้ใส่ SweetAlert2 ได้เลย
  alert(`กำลังลบข้อมูล ${selected.value.length} รายการ... \n(ส่ง ID: ${selected.value} ไปหลังบ้าน)`)
  
  // จำลองการลบ
  historyItems.value = historyItems.value.filter(item => !selected.value.includes(item.id))
  selected.value = [] // รีเซ็ต
  isDeleteMode.value = false // ออกจากโหมดลบ
}
</script>

<style scoped>
/* CSS สำหรับแถบลอยด้านล่าง */
.floating-action-bar {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%); /* จัดกึ่งกลางจอ */
  z-index: 1000;
  width: 90%;
  max-width: 600px; /* ไม่ให้ยาวเกินไปบนจอคอม */
}
</style>