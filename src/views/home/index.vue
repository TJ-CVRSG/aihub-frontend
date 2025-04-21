<script setup lang="ts">
import { computed, onMounted, ref } from 'vue';
import { NCard, NGi, NGrid, NStatistic } from 'naive-ui';
import axios from 'axios';
import { useAppStore } from '@/store/modules/app';
import HeaderBanner from './modules/header-banner.vue';

const appStore = useAppStore();

const gap = computed(() => (appStore.isMobile ? 0 : 16));

const datasetTotal = ref<number>(0);
const modelTotal = ref<number>(0);
const todayDatasetCount = ref<number>(0);
const todayModelCount = ref<number>(0);

// const recentTrainCount = ref<number>(0);
// const resourceUsage = ref({ storage: 0, gpu: 0, cpu: 0 });

// 用于前端统计今日新增数据集和模型数量
function isToday(isoString: string) {
  if (!isoString) return false;
  const d = new Date(isoString);
  if (Number.isNaN(d.getTime())) return false;
  const now = new Date();
  return d.getFullYear() === now.getFullYear() && d.getMonth() === now.getMonth() && d.getDate() === now.getDate();
}

async function fetchHomeStats() {
  try {
    const token = localStorage.getItem('token');
    // 获取数据集总数和今日新增
    const datasetRes = await axios.post(
      'http://127.0.0.1:8000/v1/dataset/get/',
      { page: 1, page_size: 9999 },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (datasetRes.data.code === 200) {
      datasetTotal.value = datasetRes.data.total || 0;
      todayDatasetCount.value = datasetRes.data.datasets.filter((item: any) => isToday(item.upload_time)).length;
    }
    // 获取模型总数和今日新增
    const modelRes = await axios.post(
      'http://127.0.0.1:8000/v1/model/get/',
      { page: 1, page_size: 9999 },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (modelRes.data.code === 200) {
      modelTotal.value = modelRes.data.total || 0;
      todayModelCount.value = modelRes.data.models.filter((item: any) => isToday(item.create_time)).length;
    }
  } catch {
    // 可加错误提示
  }
}

onMounted(fetchHomeStats);
</script>

<template>
  <NSpace vertical :size="16">
    <HeaderBanner />
    <NGrid :x-gap="gap" :y-gap="16" cols="2 s:2 m:4 l:4" responsive="screen" class="mb-8">
      <NGi>
        <NCard :bordered="false" class="flex flex-col items-center justify-center card-wrapper">
          <NStatistic label="数据集总数" :value="datasetTotal" />
        </NCard>
      </NGi>
      <NGi>
        <NCard :bordered="false" class="flex flex-col items-center justify-center card-wrapper">
          <NStatistic label="模型总数" :value="modelTotal" />
        </NCard>
      </NGi>
      <NGi>
        <NCard :bordered="false" class="flex flex-col items-center justify-center card-wrapper">
          <NStatistic label="今日新增数据集" :value="todayDatasetCount" />
        </NCard>
      </NGi>
      <NGi>
        <NCard :bordered="false" class="flex flex-col items-center justify-center card-wrapper">
          <NStatistic label="今日新增模型" :value="todayModelCount" />
        </NCard>
      </NGi>
    </NGrid>
  </NSpace>
</template>

<style scoped></style>
