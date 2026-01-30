<script setup>
import { useRouter } from 'vue-router'
import { useAuthStore } from '~/stores/auth'

const router = useRouter()
const config = useRuntimeConfig()
const authStore = useAuthStore()

// ดึงข้อมูล
const { data: assignments, pending, error } = await useFetch(`${config.public.apiBase}/api/evaluator/assignments`, {
    headers: { Authorization: `Bearer ${authStore.token}` } // ส่ง Token ไปด้วย
})

// แปลงเป็น Array กัน error
const items = computed(() => assignments.value || [])

// ไปหน้าประเมิน
const selectEvaluatee = (id) => {
    router.push(`/assess-hybrid?id=${id}`)
}

// Helper: สีสถานะ
const getStatusColor = (status) => {
    if(status === 'completed') return 'success'
    if(status === 'in_progress') return 'warning'
    return 'grey'
}
</script>

<template>
  <v-container>
    <div class="mb-6">
      <h1 class="text-h5 font-weight-bold text-indigo-darken-2">
        <v-icon start>mdi-account-group-outline</v-icon> รายการประเมินบุคลากร
      </h1>
    </div>

    <div v-if="pending" class="text-center mt-10">
        <v-progress-circular indeterminate color="indigo"></v-progress-circular>
    </div>

    <div v-else-if="error" class="text-center text-error mt-10">
        เกิดข้อผิดพลาดในการโหลดข้อมูล: {{ error.message }}
    </div>

    <div v-else-if="items.length === 0" class="text-center mt-10 text-grey">
        ไม่มีรายการที่ต้องประเมินในขณะนี้
    </div>

    <v-row v-else>
      <v-col v-for="item in items" :key="item.id" cols="12" sm="6" md="4">
        <v-card hover @click="selectEvaluatee(item.id)" class="h-100">
             <div :class="`bg-${getStatusColor(item.status)}-lighten-5 pa-2 text-right`">
                 <v-chip :color="getStatusColor(item.status)" size="x-small" label>
                    {{ item.status || 'pending' }}
                 </v-chip>
            </div>
            <v-card-item class="text-center pt-4">
                <v-avatar size="80" color="indigo-lighten-4" class="mb-3">
                    <span class="text-h4 font-weight-bold text-indigo-darken-2">
                        {{ item.evaluatee_name?.charAt(0) }}
                    </span>
                </v-avatar>
                <div class="text-h6 font-weight-bold">{{ item.evaluatee_name }}</div>
                <div class="text-body-2 text-grey">{{ item.position }} | {{ item.department }}</div>
                <div class="text-caption text-indigo mt-1">รอบ: {{ item.period_name }}</div>
            </v-card-item>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>