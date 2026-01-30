<script setup>
import { ref, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useAuthStore } from '~/stores/auth'

const route = useRoute()
const router = useRouter()
const config = useRuntimeConfig()
const authStore = useAuthStore()

const assignmentId = route.query.id 
const evidenceDialog = ref(false)
const currentIndicator = ref(null)
const currentFile = ref(null)

// ✅ 1. แก้ตรงนี้: เปลี่ยนเป็น [] เพื่อให้เริ่มต้น "ปิดทั้งหมด"
const openPanels = ref([]) 

// ... (ส่วน Fetch Data และ Logic อื่นๆ เหมือนเดิม ไม่ต้องแก้) ...
const { data: apiResponse, pending, refresh } = await useFetch(
    `${config.public.apiBase}/api/evaluator/assessments/${assignmentId}`, 
    { headers: { Authorization: `Bearer ${authStore.token}` } }
)

const assignmentInfo = computed(() => apiResponse.value?.assignment_info || {})
const evaluationForm = computed(() => apiResponse.value?.form_data || [])

const progress = computed(() => {
    if (!evaluationForm.value.length) return 0
    let total = 0, filled = 0
    evaluationForm.value.forEach(t => t.indicators.forEach(i => {
        total++
        if ((i.type === 'score_1_4' && i.score) || (i.type === 'yes_no'&& i.yes_no_val !== null)) filled++
    }))
    return total === 0 ? 0 : Math.round((filled / total) * 100)
})

const openEvidence = (indicator) => {
  currentIndicator.value = indicator
  currentFile.value = null
  evidenceDialog.value = true
}

const submitEvaluation = async () => {
    // ... (Logic บันทึกเหมือนเดิม) ...
    try {
        const results = []
        evaluationForm.value.forEach(topic => {
            topic.indicators.forEach(item => {
                if (item.score || item.yes_no_val !== null || item.note) {
                    results.push({
                        indicator_id: item.id,
                        score: item.score,
                        value_yes_no: item.yes_no_val,
                        notes: item.note
                    })
                }
            })
        })

        const payload = {
            assignment_id: assignmentId,
            results: results,
            status: progress.value === 100 ? 'completed' : 'in_progress'
        }

        await $fetch(`${config.public.apiBase}/api/evaluator/submit`, {
            method: 'POST',
            headers: { Authorization: `Bearer ${authStore.token}` },
            body: payload
        })

        alert('บันทึกข้อมูลเรียบร้อย!')
        router.push('/assess-hybrid') 
    } catch (err) {
        console.error(err)
        alert('เกิดข้อผิดพลาด: ' + (err.data?.message || err.message))
    }
}
</script>

<template>
    <div v-if="pending" class="d-flex justify-center align-center h-screen">
        <v-progress-circular indeterminate color="indigo" size="64"></v-progress-circular>
    </div>

    <v-container v-else class="pb-12">
        
        <v-card class="mb-6 bg-indigo-darken-2" elevation="3">
            <v-card-item>
                <div class="d-flex align-center">
                    <v-avatar color="white" size="56" class="mr-4">
                        <span class="text-indigo-darken-2 text-h5 font-weight-bold">
                            {{ assignmentInfo.evaluatee?.charAt(0) }}
                        </span>
                    </v-avatar>
                    <div>
                        <div class="text-h6">{{ assignmentInfo.evaluatee }}</div>
                        <div class="text-subtitle-2 text-indigo-lighten-4">
                            {{ assignmentInfo.position }} | {{ assignmentInfo.department }}
                        </div>
                    </div>
                    <v-spacer></v-spacer>
                    <div class="text-right d-none d-sm-block">
                        <div class="text-caption">ความคืบหน้า</div>
                        <div class="text-h4 font-weight-bold">{{ progress }}%</div>
                    </div>
                </div>
            </v-card-item>
            <v-progress-linear :model-value="progress" color="green-accent-3" height="6"></v-progress-linear>
        </v-card>

        <v-expansion-panels v-model="openPanels" multiple variant="default">
            <v-expansion-panel v-for="topic in evaluationForm" :key="topic.id" elevation="1" class="mb-4 rounded-lg border">
                <v-expansion-panel-title class="bg-grey-lighten-5 text-subtitle-1 font-weight-bold text-indigo-darken-3">
                    {{ topic.code }}: {{ topic.title }}
                </v-expansion-panel-title>

                <v-expansion-panel-text class="pt-2" eager>
                    
                    <div v-for="(item, idx) in topic.indicators" :key="item.id" 
                         :class="{ 'border-b pb-6 mb-6': idx !== topic.indicators.length - 1 }">
                        <v-row>
                            <v-col cols="12" md="6">
                                <div class="d-flex">
                                    <v-chip label size="small" color="grey-darken-3" class="mr-3 font-weight-bold" style="margin-top:2px;">
                                        {{ item.code }}
                                    </v-chip>
                                    <div>
                                        <div class="text-body-1 font-weight-medium mb-1">{{ item.name }}</div>
                                        <div class="text-caption text-grey-darken-1 mb-2">{{ item.desc }}</div>

                                        <v-btn v-if="item.files && item.files.length > 0" 
                                            variant="tonal" color="indigo" size="small"
                                            prepend-icon="mdi-paperclip" class="mt-2" 
                                            @click="openEvidence(item)">
                                            ดูหลักฐานแนบ ({{ item.files.length }})
                                        </v-btn>
                                        <v-chip v-else size="small" variant="text" color="grey" class="mt-2 px-0">
                                            <v-icon start>mdi-file-remove-outline</v-icon> ไม่มีไฟล์แนบ
                                        </v-chip>
                                    </div>
                                </div>
                            </v-col>

                            <v-col cols="12" md="6">
                                <v-card variant="outlined" class="pa-4 bg-grey-lighten-5 border-dashed" style="border-color: #ddd;">
                                    <div v-if="item.type === 'score_1_4'">
                                        <div class="d-flex justify-space-between mb-1">
                                            <span class="text-caption font-weight-bold text-grey-darken-2">ระดับคุณภาพ</span>
                                            <span v-if="item.score" class="text-caption font-weight-bold text-indigo">
                                                {{ item.score }}/4 คะแนน
                                            </span>
                                        </div>
                                        <v-slider v-model="item.score" :min="1" :max="4" :step="1" 
                                            show-ticks="always" step-size="1" tick-size="4" 
                                            color="indigo" thumb-label hide-details>
                                        </v-slider>
                                    </div>

                                    <div v-else-if="item.type === 'yes_no'">
                                        <span class="text-caption font-weight-bold text-grey-darken-2 d-block mb-2">ผลการประเมิน</span>
                                        <v-btn-toggle v-model="item.yes_no_val" density="compact" rounded="lg" divided mandatory class="w-100 border">
                                            <v-btn :value="1" color="success" class="flex-grow-1" variant="flat">ผ่าน</v-btn>
                                            <v-btn :value="0" color="error" class="flex-grow-1" variant="flat">ไม่ผ่าน</v-btn>
                                        </v-btn-toggle>
                                    </div>

                                    <v-text-field v-model="item.note" density="compact" variant="underlined" 
                                        label="ความคิดเห็นเพิ่มเติม" hide-details class="mt-2 text-body-2"
                                        prepend-icon="mdi-comment-text-outline">
                                    </v-text-field>
                                </v-card>
                            </v-col>
                        </v-row>
                    </div>

                </v-expansion-panel-text>
            </v-expansion-panel>
        </v-expansion-panels>

        <v-footer app elevation="5" class="bg-white pa-3 justify-end">
            <v-btn color="indigo-darken-2" size="large" width="200" elevation="2" prepend-icon="mdi-content-save-check"
                @click="submitEvaluation">
                บันทึกข้อมูล ({{ progress }}%)
            </v-btn>
        </v-footer>

        <v-dialog v-model="evidenceDialog" max-width="900" scrollable>
             <v-card v-if="currentIndicator" class="rounded-lg">
                <v-toolbar density="compact" color="white" class="border-b">
                    <v-toolbar-title class="text-subtitle-1 font-weight-bold">
                        <span class="text-indigo mr-2">{{ currentIndicator.code }}:</span> หลักฐานประกอบ
                    </v-toolbar-title>
                    <v-spacer></v-spacer>
                    <v-btn v-if="currentFile" :href="currentFile.url" target="_blank" variant="text" color="indigo" prepend-icon="mdi-download" class="mr-2">ดาวน์โหลด</v-btn>
                    <v-btn icon="mdi-close" variant="text" @click="evidenceDialog = false"></v-btn>
                </v-toolbar>

                <v-card-text class="bg-grey-lighten-4 pa-4" style="min-height: 400px;">
                    <v-row>
                        <v-col cols="12" md="4">
                            <div class="text-caption font-weight-bold mb-2 ml-1">รายการไฟล์ ({{ currentIndicator.files.length }})</div>
                            <v-list density="compact" rounded="lg" elevation="1" bg-color="white">
                                <v-list-item v-for="(file, i) in currentIndicator.files" :key="i" 
                                    :title="file.name"
                                    :prepend-icon="file.type === 'pdf' ? 'mdi-file-pdf-box' : 'mdi-image'" 
                                    :value="file" color="indigo" rounded="lg" class="mb-1"
                                    @click="currentFile = file">
                                </v-list-item>
                            </v-list>
                        </v-col>

                        <v-col cols="12" md="8" class="bg-grey-darken-4 pa-0 d-flex align-center justify-center">
                            <div v-if="!currentFile" class="text-grey text-center">
                                <v-icon size="64">mdi-eye-off-outline</v-icon>
                                <div class="mt-2">เลือกไฟล์เพื่อดูตัวอย่าง</div>
                            </div>
                            <v-img v-else-if="currentFile.type === 'image'" :src="currentFile.url" max-height="100%" contain></v-img>
                            <iframe v-else-if="currentFile.type === 'pdf'" :src="currentFile.url" width="100%" height="100%" style="border: none;"></iframe>
                             <div v-else class="text-center text-white">
                                <v-icon size="64">mdi-file-question</v-icon>
                                <div>ไม่รองรับการแสดงผล</div>
                            </div>
                        </v-col>
                    </v-row>
                </v-card-text>
            </v-card>
        </v-dialog>

    </v-container>
</template>