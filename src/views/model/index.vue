<script setup lang="ts">
import { h, onMounted, ref } from 'vue';
import { NButton, NDataTable, NModal, NPagination } from 'naive-ui';
import axios from 'axios';
import { Icon } from '@iconify/vue';

const selectedDataset = ref<Dataset | null>(null);
const placeholderImg = 'src/assets/svg-icon/avatar.svg';

// 数据集类型
interface Dataset {
  id: number | string;
  name: string;
  task: keyof typeof taskMap | string;
  class_count?: number;
  image_count?: number;
  train_count?: number;
  val_count?: number;
  test_count?: number;
  classes?: Array<{ name: string }>;
  description?: string;
  path?: string;
  size?: number;
  upload_time?: Date | string;
  thumbnail?: string;
}

// 模型类型
interface ModelRow {
  name: string;
  task: keyof typeof taskMap | string;
  status: keyof typeof statusMap | string;
  weight_type: keyof typeof weightTypeMap | string;
  size: number;
  path: string;
  create_time: Date | string;
}

// 数据集列表相关状态
const datasets = ref<Dataset[]>([]);
const loading = ref(false);
const error = ref('');

// 任务类型映射
const taskMap = { detect: '目标检测', classify: '分类', segment: '分割', pose: '姿态估计', rotate: '旋转框检测' };

async function fetchDatasets() {
  loading.value = true;
  error.value = '';
  try {
    const token = localStorage.getItem('token');
    // const test_token = 'c6ccd5481e78655b26f4a2f64eb8ef905449fd9a';
    // 不分页，获取全部数据集
    const res = await axios.post(
      'http://127.0.0.1:8000/v1/dataset/get/',
      { page: 1, page_size: 9999 },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (res.data.code === 200) {
      datasets.value = res.data.datasets;
      // 默认选中第一个
      if (datasets.value.length > 0) selectedDataset.value = datasets.value[0];
    } else {
      error.value = res.data.message || '加载失败';
    }
  } catch {
    error.value = '网络错误';
  } finally {
    loading.value = false;
  }
}

// function selectDataset(dataset: Dataset) {
//   selectedDataset.value = dataset;
// }

onMounted(fetchDatasets);

function formatSize(size?: number) {
  if (typeof size !== 'number') return '-';
  if (size > 1024 * 1024) return `${(size / 1024 / 1024).toFixed(2)} MB`;
  if (size > 1024) return `${(size / 1024).toFixed(2)} KB`;
  return `${size} B`;
}
function formatTime(time?: Date | string) {
  if (!time) return '-';
  if (typeof time === 'string') {
    const d = new Date(time);
    if (Number.isNaN(d.getTime())) return time;
    return d.toLocaleString();
  }
  if (time instanceof Date) return time.toLocaleString();
  return '-';
}

const models = ref<ModelRow[]>([]);
const total = ref(0);
const page = ref(1);
const pageSize = ref(10);
// const message = useMessage();

const statusMap = { initialized: '已初始化', trained: '已训练', quantized: '已量化' };
const weightTypeMap = {
  float32: 'float32',
  int4: 'int4',
  int8: 'int8',
  int16: 'int16',
  float8: 'float8',
  float16: 'float16'
};

const columns = [
  { title: '模型名称', key: 'name' },
  { title: '任务类型', key: 'task', render: (row: ModelRow) => taskMap[row.task as keyof typeof taskMap] || row.task },
  {
    title: '状态',
    key: 'status',
    render: (row: ModelRow) => statusMap[row.status as keyof typeof statusMap] || row.status
  },
  {
    title: '量化类型',
    key: 'weight_type',
    render: (row: ModelRow) => weightTypeMap[row.weight_type as keyof typeof weightTypeMap] || row.weight_type
  },
  { title: '大小(MB)', key: 'size', render: (row: ModelRow) => formatSize(row.size) },
  { title: '存储路径', key: 'path' },
  { title: '创建时间', key: 'create_time', render: (row: ModelRow) => formatTime(row.create_time) }
];

async function fetchModels() {
  loading.value = true;
  error.value = '';
  try {
    const token = localStorage.getItem('token');
    // const test_token = 'c6ccd5481e78655b26f4a2f64eb8ef905449fd9a';
    const res = await axios.post(
      'http://127.0.0.1:8000/v1/model/get/',
      { page: page.value, page_size: pageSize.value },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (res.data.code === 200) {
      models.value = res.data.models;
      total.value = res.data.total;
    } else {
      error.value = res.data.message || '加载失败';
    }
  } catch {
    error.value = '网络错误';
  } finally {
    loading.value = false;
  }
}

onMounted(fetchModels);
function handlePageChange(p: number) {
  page.value = p;
  fetchModels();
}

const showTrainDialog = ref(false);
const trainStep = ref(1); // 1: 选择数据集, 2: 选择模型

function openTrainDialog() {
  showTrainDialog.value = true;
  trainStep.value = 1;
}
function closeTrainDialog() {
  showTrainDialog.value = false;
}
function nextTrainStep() {
  trainStep.value += 1;
}
function prevTrainStep() {
  trainStep.value -= 1;
}
const selectedTrainDataset = ref<Dataset | null>(null);
function selectTrainDataset(dataset: Dataset) {
  selectedTrainDataset.value = dataset;
}
</script>

<template>
  <div>
    <div style="display: flex; justify-content: flex-end; margin-bottom: 16px">
      <NButton type="primary" @click="openTrainDialog">Train</NButton>
    </div>
    <NModal
      v-model:show="showTrainDialog"
      :title="trainStep === 1 ? '选择数据集' : '选择模型'"
      :icon="() => h(Icon, { icon: 'mdi:rocket-launch', style: 'font-size: 24px' })"
      preset="dialog"
      style="width: 900px; border-radius: 16px; box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12)"
    >
      <template v-if="trainStep === 1">
        <div style="display: flex; height: 420px; background: #f8fafb; border-radius: 12px; overflow: hidden">
          <!-- 左侧数据集列表 -->
          <div style="width: 270px; border-right: 1px solid #eee; overflow-y: auto; background: #fff; padding: 8px 0">
            <div
              v-for="dataset in datasets"
              :key="dataset.id"
              :style="{
                display: 'flex',
                alignItems: 'center',
                gap: '14px',
                padding: '14px 12px',
                margin: '8px 12px',
                borderRadius: '10px',
                cursor: 'pointer',
                background:
                  selectedTrainDataset && selectedTrainDataset.id === dataset.id
                    ? 'var(--soy-primary-color, #646cff20)'
                    : '#f8fafb',
                fontWeight: selectedTrainDataset && selectedTrainDataset.id === dataset.id ? 'bold' : 'normal',
                border:
                  selectedTrainDataset && selectedTrainDataset.id === dataset.id
                    ? '2px solid var(--soy-primary-color, #646cff)'
                    : '2px solid transparent',
                boxShadow:
                  selectedTrainDataset && selectedTrainDataset.id === dataset.id
                    ? '0 2px 8px var(--soy-primary-color, #646cff22)'
                    : 'none',
                transition: 'all 0.2s'
              }"
              @click="selectTrainDataset(dataset)"
              @mouseover="
                (e: MouseEvent) => {
                  const t = e.currentTarget as HTMLElement | null;
                  if (t) t.style.background = 'var(--soy-primary-color, #646cff10)';
                }
              "
              @mouseleave="
                (e: MouseEvent) => {
                  const t = e.currentTarget as HTMLElement | null;
                  if (t)
                    t.style.background =
                      selectedTrainDataset && selectedTrainDataset.id === dataset.id
                        ? 'var(--soy-primary-color, #646cff20)'
                        : '#f8fafb';
                }
              "
            >
              <img
                :src="dataset.thumbnail || placeholderImg"
                alt="thumb"
                style="
                  width: 44px;
                  height: 44px;
                  object-fit: cover;
                  border-radius: 8px;
                  background: #f0f0f0;
                  flex-shrink: 0;
                  box-shadow: 0 2px 8px #f0f1f2;
                "
                @error="
                  (e: Event) => {
                    const t = e.target as HTMLImageElement | null;
                    if (t) t.src = placeholderImg;
                  }
                "
              />
              <div style="flex: 1; min-width: 0">
                <div style="white-space: nowrap; overflow: hidden; text-overflow: ellipsis; font-size: 16px">
                  {{ dataset.name }}
                </div>
                <div
                  style="font-size: 12px; color: #888; white-space: nowrap; overflow: hidden; text-overflow: ellipsis"
                >
                  {{ taskMap[dataset.task as keyof typeof taskMap] || dataset.task }}
                </div>
              </div>
            </div>
          </div>
          <!-- 右侧预览信息 -->
          <div style="flex: 1; padding: 32px 32px 24px 32px; overflow-y: auto; background: #f8fafb">
            <template v-if="selectedTrainDataset">
              <div style="font-size: 22px; font-weight: bold; margin-bottom: 8px; color: #222; letter-spacing: 1px">
                {{ selectedTrainDataset.name }}
              </div>
              <div
                style="margin-bottom: 12px; color: var(--soy-primary-color, #646cff); font-weight: 500; font-size: 15px"
              >
                {{ taskMap[selectedTrainDataset.task as keyof typeof taskMap] || selectedTrainDataset.task }}
              </div>
              <div style="display: flex; gap: 32px; margin-bottom: 12px">
                <div>
                  类别数:
                  <b>{{ selectedTrainDataset.class_count }}</b>
                </div>
                <div>
                  图片数:
                  <b>{{ selectedTrainDataset.image_count }}</b>
                </div>
              </div>
              <div style="margin-bottom: 12px">
                <span>类别:</span>
                <div style="display: flex; flex-wrap: wrap; gap: 8px; margin-top: 4px">
                  <span
                    v-for="cls in Array.isArray(selectedTrainDataset.classes) ? selectedTrainDataset.classes : []"
                    :key="cls.name"
                    style="
                      background: rgba(var(--soy-primary-color-rgb, 100, 108, 255), 0.08);
                      color: var(--soy-primary-color, #646cff);
                      border-radius: 6px;
                      padding: 2px 10px;
                      font-size: 13px;
                      border: 1px solid var(--soy-primary-color, #646cff);
                      display: inline-block;
                    "
                  >
                    {{ cls.name }}
                  </span>
                </div>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                训练/验证/测试:
                <b>
                  {{ selectedTrainDataset.train_count }}/{{ selectedTrainDataset.val_count }}/{{
                    selectedTrainDataset.test_count
                  }}
                </b>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                描述:
                <span style="color: #444">{{ selectedTrainDataset.description || '无' }}</span>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                路径:
                <span style="color: #444">{{ selectedTrainDataset.path }}</span>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                大小:
                <span style="color: #444">{{ formatSize(selectedTrainDataset.size) }}</span>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                上传时间:
                <span style="color: #444">{{ formatTime(selectedTrainDataset.upload_time) }}</span>
              </div>
            </template>
            <template v-else>
              <div style="color: #aaa; font-size: 16px; text-align: center; margin-top: 80px">
                请选择左侧数据集查看详情
              </div>
            </template>
          </div>
        </div>
        <div style="display: flex; justify-content: flex-end; margin-top: 16px">
          <NButton type="primary" :disabled="!selectedTrainDataset" @click="nextTrainStep">下一步</NButton>
        </div>
      </template>
      <template v-else-if="trainStep === 2">
        <div style="height: 420px; display: flex; align-items: center; justify-content: center">
          <span style="color: #888; font-size: 20px">模型选择界面（待实现）</span>
        </div>
        <div style="display: flex; justify-content: flex-end; gap: 12px; margin-top: 16px">
          <NButton @click="prevTrainStep">上一步</NButton>
          <NButton type="primary" @click="closeTrainDialog">关闭</NButton>
        </div>
      </template>
    </NModal>
    <NDataTable :columns="columns" :data="models" :loading="loading" />
    <NPagination
      v-model:page="page"
      :page-size="pageSize"
      :page-count="Math.ceil(total / pageSize)"
      @update:page="handlePageChange"
    />
  </div>
</template>

<style scoped>
.n-card {
  transition: box-shadow 0.2s;
}
.n-card:hover {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}
</style>
