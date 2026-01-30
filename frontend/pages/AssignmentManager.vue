<template>
  <div class="pa-4">
    <v-row>
      <v-col cols="12" md="4">
        <v-card elevation="2" rounded="lg" class="h-100">
          <v-card-item>
            <v-card-title class="font-weight-bold" :class="isEditing ? 'text-amber-darken-3' : 'text-indigo'">
              <v-icon :icon="isEditing ? 'mdi-pencil-box' : 'mdi-link-variant-plus'" start></v-icon>
              {{ isEditing ? 'แก้ไขการจับคู่' : 'จับคู่การประเมิน' }}
            </v-card-title>
            <v-card-subtitle>เลือกปี, กรรมการ และผู้รับการประเมิน</v-card-subtitle>
          </v-card-item>

          <v-divider></v-divider>

          <v-card-text class="d-flex flex-column gap-4 mt-2">
            
            <v-select
              v-model="selectedPeriod"
              :items="periods"
              item-title="name_th"
              item-value="id"
              label="เลือกรอบการประเมิน"
              variant="outlined"
              color="indigo"
              prepend-inner-icon="mdi-calendar-clock"
              density="comfortable"
              :loading="loading"
            ></v-select>

            <v-divider class="my-2 border-opacity-50"></v-divider>

            <v-autocomplete
              v-model="form.evaluator_id"
              :items="evaluators"
              item-title="name_th"
              item-value="id"
              label="เลือกกรรมการ (Evaluator)"
              placeholder="พิมพ์ชื่อเพื่อค้นหา..."
              variant="outlined"
              color="teal"
              prepend-inner-icon="mdi-account-tie"
              density="comfortable"
              clearable
              :loading="loading"
            >
              <template v-slot:item="{ props, item }">
                <v-list-item v-bind="props" :subtitle="'Role: ' + item.raw.role"></v-list-item>
              </template>
            </v-autocomplete>

            <v-autocomplete
              v-model="form.evaluatee_id"
              :items="evaluatees"
              item-title="name_th"
              item-value="id"
              label="เลือกผู้รับการประเมิน (Evaluatee)"
              placeholder="พิมพ์ชื่อ หรือ แผนก..."
              variant="outlined"
              color="amber-darken-3"
              prepend-inner-icon="mdi-account-school"
              density="comfortable"
              clearable
              :loading="loading"
              :custom-filter="filterByDept"
            >
              <template v-slot:item="{ props, item }">
                <v-list-item 
                  v-bind="props" 
                  :subtitle="item.raw.dept_name || 'ไม่ระบุแผนก'"
                ></v-list-item>
              </template>
            </v-autocomplete>

            <div class="d-flex flex-column gap-2 mt-4">
              <v-btn
                block
                :color="isEditing ? 'amber-darken-3' : 'indigo-darken-1'"
                size="large"
                rounded="pill"
                :prepend-icon="isEditing ? 'mdi-content-save-edit' : 'mdi-plus-circle'"
                :disabled="!isValidForm || loading"
                :loading="submitting"
                @click="saveAssignment"
              >
                {{ isEditing ? 'บันทึกการแก้ไข (Update)' : 'จับคู่ (Assign)' }}
              </v-btn>

              <v-btn
                v-if="isEditing"
                block
                variant="text"
                color="grey-darken-1"
                rounded="pill"
                prepend-icon="mdi-close"
                @click="cancelEdit"
              >
                ยกเลิก
              </v-btn>
            </div>

          </v-card-text>
        </v-card>
      </v-col>

      <v-col cols="12" md="8">
        <v-card elevation="2" rounded="lg" class="h-100">
          <v-data-table
            :headers="headers"
            :items="filteredAssignments"
            :search="search"
            density="comfortable"
            hover
          >
            <template v-slot:top>
              <v-toolbar flat color="white">
                <v-toolbar-title class="font-weight-bold">
                  รายการจับคู่ ({{ filteredAssignments.length }} คู่)
                </v-toolbar-title>
                <v-spacer></v-spacer>
                <v-text-field
                  v-model="search"
                  prepend-inner-icon="mdi-magnify"
                  label="ค้นหา..."
                  single-line
                  hide-details
                  density="compact"
                  variant="outlined"
                  class="mr-4"
                  style="max-width: 250px;"
                ></v-text-field>
              </v-toolbar>
            </template>

            <template v-slot:item.evaluator="{ item }">
              <div class="d-flex align-center py-2">
                <v-avatar color="teal-lighten-5" size="32" class="mr-2">
                  <span class="teal--text text-caption font-weight-bold">ER</span>
                </v-avatar>
                <div>
                  <div class="font-weight-medium">{{ item.evaluator_name }}</div>
                  <div class="text-caption text-grey">{{ item.evaluator_role }}</div>
                </div>
              </div>
            </template>

            <template v-slot:item.arrow>
              <v-icon color="grey-lighten-1">mdi-arrow-right</v-icon>
            </template>

            <template v-slot:item.evaluatee="{ item }">
              <div class="d-flex align-center py-2">
                <v-avatar color="amber-lighten-5" size="32" class="mr-2">
                  <span class="amber--text text-caption font-weight-bold">EE</span>
                </v-avatar>
                <div>
                  <div class="font-weight-medium">{{ item.evaluatee_name }}</div>
                  <div class="text-caption text-grey">{{ item.evaluatee_dept || '-' }}</div>
                </div>
              </div>
            </template>

            <template v-slot:item.actions="{ item }">
              <div class="d-flex justify-end">
                <v-btn
                  icon="mdi-pencil"
                  variant="text"
                  color="amber-darken-2"
                  size="small"
                  class="mr-1"
                  @click="editAssignment(item)"
                ></v-btn>
                <v-btn
                  icon="mdi-delete"
                  variant="text"
                  color="red-lighten-1"
                  size="small"
                  @click="removeAssignment(item.id)"
                ></v-btn>
              </div>
            </template>
            
            <template v-slot:no-data>
               <div class="pa-4 text-center text-grey">ยังไม่มีข้อมูลการจับคู่</div>
            </template>
          </v-data-table>
        </v-card>
      </v-col>
    </v-row>

    <v-snackbar v-model="snackbar" :color="snackbarColor" timeout="3000">
      {{ snackbarText }}
      <template v-slot:actions>
        <v-btn variant="text" @click="snackbar = false">ปิด</v-btn>
      </template>
    </v-snackbar>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import axios from 'axios';


definePageMeta({
  middleware: 'auth'
})
// --- State ---
const periods = ref([]);
const evaluators = ref([]);
const evaluatees = ref([]);
const assignments = ref([]);

const selectedPeriod = ref(null);
const form = ref({ evaluator_id: null, evaluatee_id: null });
const search = ref('');
const loading = ref(false);
const submitting = ref(false);

// ✅ State สำหรับการแก้ไข
const isEditing = ref(false);
const editingId = ref(null);

const snackbar = ref(false);
const snackbarText = ref('');
const snackbarColor = ref('success');

const headers = [
  { title: 'ผู้ประเมิน (Evaluator)', key: 'evaluator', width: '35%', value: item => `${item.evaluator_name} ${item.evaluator_role}` },
  { title: '', key: 'arrow', align: 'center', sortable: false, width: '50px' },
  { title: 'ผู้รับการประเมิน (Evaluatee)', key: 'evaluatee', width: '35%', value: item => `${item.evaluatee_name} ${item.evaluatee_dept}` },
  { title: 'จัดการ', key: 'actions', align: 'end', sortable: false },
];

// --- Computed ---
const filteredAssignments = computed(() => {
  if (!selectedPeriod.value) return [];
  return assignments.value.filter(a => a.period_id === selectedPeriod.value);
});

const isValidForm = computed(() => {
  return selectedPeriod.value && form.value.evaluator_id && form.value.evaluatee_id;
});

const filterByDept = (itemTitle, queryText, item) => {
    const textName = item.raw.name_th ? item.raw.name_th.toLowerCase() : '';
    const textDept = item.raw.dept_name ? item.raw.dept_name.toLowerCase() : '';
    const searchText = queryText.toLowerCase();
    return textName.indexOf(searchText) > -1 || textDept.indexOf(searchText) > -1;
};

// --- API Helpers ---
const getAuthHeader = () => {
  return { headers: { Authorization: `Bearer ${localStorage.getItem('token')}` } };
};

const fetchData = async () => {
  loading.value = true;
  try {
    const header = getAuthHeader();
    const [resPeriods, resEvaluators, resEvaluatees, resAssignments] = await Promise.all([
        axios.get('http://localhost:7000/api/admin/periods', header),
        axios.get('http://localhost:7000/api/admin/users/evaluators', header),
        axios.get('http://localhost:7000/api/admin/users/evaluatees', header),
        axios.get('http://localhost:7000/api/admin/assignments', header)
    ]);

    periods.value = resPeriods.data;
    evaluators.value = resEvaluators.data;
    evaluatees.value = resEvaluatees.data;
    assignments.value = resAssignments.data;

    // Auto-select ปีที่ Active (ถ้ายังไม่เลือก)
    const activePeriod = periods.value.find(p => p.is_active === 1 || p.active === 1);
    if (activePeriod && !selectedPeriod.value) {
        selectedPeriod.value = activePeriod.id;
    }

  } catch (error) {
    console.error("Error fetching data:", error);
    showSnackbar('โหลดข้อมูลไม่สำเร็จ', 'error');
  } finally {
    loading.value = false;
  }
};

// --- Actions ---

// 1. ฟังก์ชัน Save (ใช้ร่วมกันทั้ง Create และ Update)
const saveAssignment = async () => {
  if (!isValidForm.value) return;
  submitting.value = true;
  
  try {
    const payload = {
        period_id: selectedPeriod.value,
        evaluator_id: form.value.evaluator_id,
        evaluatee_id: form.value.evaluatee_id
    };

    if (isEditing.value) {
        // --- กรณีแก้ไข (UPDATE) ---
        await axios.put(`http://localhost:7000/api/admin/assignments/${editingId.value}`, payload, getAuthHeader());
        showSnackbar('แก้ไขข้อมูลเรียบร้อย');
    } else {
        // --- กรณีสร้างใหม่ (CREATE) ---
        await axios.post('http://localhost:7000/api/admin/assignments', payload, getAuthHeader());
        showSnackbar('จับคู่สำเร็จเรียบร้อย');
    }
    
    // โหลดตารางใหม่
    const res = await axios.get('http://localhost:7000/api/admin/assignments', getAuthHeader());
    assignments.value = res.data;

    // Reset Form
    cancelEdit(); 

  } catch (error) {
    showSnackbar(error.response?.data?.message || 'เกิดข้อผิดพลาด', 'error');
  } finally {
    submitting.value = false;
  }
};

// 2. ฟังก์ชันเริ่มแก้ไข
const editAssignment = (item) => {
    isEditing.value = true;
    editingId.value = item.id;
    
    // ดึงข้อมูลจากแถวมาใส่ฟอร์ม
    form.value = {
        evaluator_id: item.evaluator_id,
        evaluatee_id: item.evaluatee_id
    };
    
    // ปรับปีให้ตรงกับรายการที่แก้ (เผื่อดูปีอื่นอยู่)
    selectedPeriod.value = item.period_id;
};

// 3. ฟังก์ชันยกเลิกแก้ไข
const cancelEdit = () => {
    isEditing.value = false;
    editingId.value = null;
    form.value = { evaluator_id: null, evaluatee_id: null };
};

const removeAssignment = async (id) => {
  if (!confirm('ยืนยันที่จะลบการจับคู่นี้?')) return;
  try {
    await axios.delete(`http://localhost:7000/api/admin/assignments/${id}`, getAuthHeader());
    assignments.value = assignments.value.filter(a => a.id !== id);
    showSnackbar('ลบรายการเรียบร้อย');
    
    // ถ้ายกเลิกตัวที่กำลัง Edit อยู่พอดี ให้ Reset Form ด้วย
    if (editingId.value === id) cancelEdit();
    
  } catch (error) {
    showSnackbar('ลบไม่สำเร็จ', 'error');
  }
};

const showSnackbar = (text, color = 'success') => {
  snackbarText.value = text;
  snackbarColor.value = color;
  snackbar.value = true;
};

onMounted(() => {
  fetchData();
});
</script>