<template>
  <div class="pa-4">
    <v-card elevation="4" rounded="lg">
      <v-data-table 
        :headers="headers" 
        :items="periods" 
        :loading="loading"
        hover 
        density="comfortable"
      >
        <template v-slot:top>
          <v-toolbar color="teal-darken-1" flat rounded="t-lg">
            <v-toolbar-title class="font-weight-bold">
              <v-icon icon="mdi-calendar-clock" start></v-icon>
              จัดการรอบการประเมิน (Evaluation Periods)
            </v-toolbar-title>
            <v-spacer></v-spacer>
            <v-btn 
              color="white" 
              variant="flat" 
              prepend-icon="mdi-plus-circle" 
              rounded="pill"
              class="text-none me-4"
              @click="openDialog()"
            >
              เปิดรอบประเมินใหม่
            </v-btn>
          </v-toolbar>
        </template>

        <template v-slot:item.period_date="{ item }">
          <div class="text-caption">
            <div>เริ่ม: {{ formatDate(item.start_date) }}</div>
            <div>สิ้นสุด: {{ formatDate(item.end_date) }}</div>
          </div>
        </template>

        <template v-slot:item.is_active="{ item }">
          <v-switch
            v-model="item.is_active"
            :model-value="item.is_active === 1 || item.is_active === true"
            color="success"
            hide-details
            density="compact"
            inset
            @update:model-value="toggleStatus(item)"
          ></v-switch>
        </template>

        <template v-slot:item.actions="{ item }">
          <div class="d-flex justify-end">
            <v-btn icon="mdi-pencil" size="small" variant="text" color="amber-darken-2" @click="openDialog(item)"></v-btn>
            <v-btn icon="mdi-delete" size="small" variant="text" color="red-lighten-1" @click="deletePeriod(item.id)"></v-btn>
          </div>
        </template>

        <template v-slot:no-data>
          <div class="pa-4 text-center text-medium-emphasis">ไม่พบข้อมูลรอบการประเมิน</div>
        </template>
      </v-data-table>
    </v-card>

    <v-dialog v-model="dialog" max-width="600px" persistent>
      <v-card rounded="xl">
        <v-toolbar :color="isEditing ? 'amber-darken-1' : 'teal'" theme="dark">
          <v-toolbar-title>{{ isEditing ? 'แก้ไขรอบการประเมิน' : 'สร้างรอบการประเมินใหม่' }}</v-toolbar-title>
          <v-btn icon="mdi-close" variant="text" @click="dialog = false"></v-btn>
        </v-toolbar>

        <v-card-text class="pa-6">
          <v-form ref="form" v-model="valid">
            <v-row dense>
              <v-col cols="12" md="6">
                <v-text-field
                  v-model="formModel.code"
                  label="รหัสรอบ (Code)"
                  hint="เช่น 1/2568"
                  persistent-hint
                  variant="outlined"
                  density="comfortable"
                  :rules="[v => !!v || 'กรุณากรอกรหัส']"
                ></v-text-field>
              </v-col>

              <v-col cols="12" md="6">
                <v-text-field
                  v-model="formModel.buddhist_year"
                  label="ปีงบประมาณ (พ.ศ.)"
                  type="number"
                  variant="outlined"
                  density="comfortable"
                  :rules="[v => !!v || 'กรุณากรอกปี พ.ศ.']"
                ></v-text-field>
              </v-col>

              <v-col cols="12">
                <v-text-field
                  v-model="formModel.name_th"
                  label="ชื่อรอบการประเมิน (ภาษาไทย)"
                  variant="outlined"
                  density="comfortable"
                  :rules="[v => !!v || 'กรุณากรอกชื่อรอบ']"
                ></v-text-field>
              </v-col>

              <v-col cols="12" md="6">
                <v-text-field
                  v-model="formModel.start_date"
                  label="วันที่เริ่มต้น"
                  type="date"
                  variant="outlined"
                  density="comfortable"
                  :rules="[v => !!v || 'กรุณาเลือกวันเริ่มต้น']"
                ></v-text-field>
              </v-col>
              <v-col cols="12" md="6">
                <v-text-field
                  v-model="formModel.end_date"
                  label="วันที่สิ้นสุด"
                  type="date"
                  variant="outlined"
                  density="comfortable"
                  :rules="[v => !!v || 'กรุณาเลือกวันสิ้นสุด']"
                ></v-text-field>
              </v-col>
              
              <v-col cols="12">
                <v-checkbox
                  v-model="formModel.is_active"
                  label="เปิดใช้งานทันที (Active)"
                  color="success"
                  hide-details
                ></v-checkbox>
              </v-col>
            </v-row>
          </v-form>
        </v-card-text>

        <v-divider></v-divider>

        <v-card-actions class="pa-4 bg-grey-lighten-5">
          <v-spacer></v-spacer>
          <v-btn variant="text" color="grey-darken-1" @click="dialog = false">ยกเลิก</v-btn>
          <v-btn 
            variant="elevated" 
            :color="isEditing ? 'amber-darken-3' : 'teal-darken-1'" 
            class="px-6"
            rounded="pill"
            @click="savePeriod"
          >
            บันทึกข้อมูล
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

// --- State Variables ---
const periods = ref([]);
const loading = ref(false);
const dialog = ref(false);
const valid = ref(false);
const formModel = ref({
  id: null,
  code: '',
  name_th: '',
  buddhist_year: new Date().getFullYear() + 543, // Default ปีปัจจุบัน
  start_date: '',
  end_date: '',
  is_active: false
});

// --- Table Configuration ---
const headers = [
  { title: 'รหัส (Code)', key: 'code', width: '100px' },
  { title: 'ปี (พ.ศ.)', key: 'buddhist_year', align: 'center', width: '100px' },
  { title: 'ชื่อรอบการประเมิน', key: 'name_th' },
  { title: 'ระยะเวลา', key: 'period_date', sortable: false },
  { title: 'สถานะ (Active)', key: 'is_active', align: 'center', width: '120px' },
  { title: 'จัดการ', key: 'actions', align: 'end', sortable: false, width: '100px' },
];

const isEditing = computed(() => !!formModel.value.id);

// --- API Methods ---
const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return { headers: { Authorization: `Bearer ${token}` } };
};

const fetchPeriods = async () => {
  loading.value = true;
  try {
    // GET: ดึงข้อมูลทั้งหมด
    const res = await axios.get('http://localhost:7000/api/admin/periods', getAuthHeader());
    periods.value = res.data;
  } catch (error) {
    console.error("Error fetching periods:", error);
  } finally {
    loading.value = false;
  }
};

const savePeriod = async () => {
  if (!formModel.value.code || !formModel.value.name_th) return; // Simple validation

  try {
    const payload = { ...formModel.value };
    // Convert boolean checkbox to 0/1 for DB
    payload.is_active = payload.is_active ? 1 : 0; 

    if (isEditing.value) {
      // PUT: แก้ไข
      await axios.put(`http://localhost:7000/api/admin/periods/${payload.id}`, payload, getAuthHeader());
    } else {
      // POST: สร้างใหม่
      await axios.post('http://localhost:7000/api/admin/periods', payload, getAuthHeader());
    }
    dialog.value = false;
    fetchPeriods(); // Reload table
  } catch (error) {
    alert('บันทึกไม่สำเร็จ: ' + (error.response?.data?.message || error.message));
  }
};


const deletePeriod = async (id) => {
  if (!confirm('ยืนยันการลบรอบการประเมินนี้? ข้อมูลการประเมินที่เกี่ยวข้องอาจได้รับผลกระทบ')) return;
  try {
    await axios.delete(`http://localhost:7000/api/admin/periods/${id}`, getAuthHeader());
    fetchPeriods();
  } catch (error) {
    alert('ลบไม่สำเร็จ: ' + error.message);
  }
};

const toggleStatus = async (item) => {
  try {
    // สลับค่า 0 <-> 1
    const newStatus = item.is_active;
    
    // ยิง API เฉพาะสำหรับเปลี่ยนสถานะ (เพื่อให้ Backend ไปปิด Active ปีอื่นให้อัตโนมัติ)
    await axios.put(`http://localhost:7000/api/admin/periods/${item.id}/status`, { 
      is_active: newStatus 
    }, getAuthHeader());
    
    fetchPeriods(); // โหลดใหม่เพื่อให้เห็นว่าปีอื่นถูกปิด Active แล้ว (ถ้ามี Logic นั้น)
  } catch (error) {
    console.error("Toggle error:", error);
    item.is_active = !item.is_active; // Revert switch UI if failed
  }
};

// --- Helper Methods ---
const openDialog = (item = null) => {
  if (item) {
    // Clone item to formModel
    // ต้องแปลงวันที่จาก timestamp (ถ้ามี) เป็น YYYY-MM-DD เพื่อใส่ใน <input type="date">
    formModel.value = { 
      ...item,
      start_date: item.start_date ? item.start_date.split('T')[0] : '',
      end_date: item.end_date ? item.end_date.split('T')[0] : '',
      is_active: item.is_active === 1
    };
  } else {
    // Reset form for new entry
    formModel.value = {
      id: null,
      code: '',
      name_th: '',
      buddhist_year: new Date().getFullYear() + 543,
      start_date: '',
      end_date: '',
      is_active: false
    };
  }
  dialog.value = true;
};

const formatDate = (dateStr) => {
  if (!dateStr) return '-';
  const date = new Date(dateStr);
  return date.toLocaleDateString('th-TH', { day: '2-digit', month: 'short', year: 'numeric' });
};

onMounted(() => {
  fetchPeriods();
});

definePageMeta({
  middleware: 'auth'
})
</script>