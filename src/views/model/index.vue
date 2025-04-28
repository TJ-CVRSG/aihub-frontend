<script setup lang="ts">
import { h, onMounted, ref } from 'vue';
import { NButton, NDataTable, NModal, NPagination } from 'naive-ui';
import axios from 'axios';
import { Icon } from '@iconify/vue';
import TrainingProgressModal from './TrainingProgressModal.vue';

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

// 默认模型类型
interface DefaultModel {
  id: number | string;
  name: string;
  task: keyof typeof taskMap | string;
  description?: string;
  thumbnail?: string;
  size?: number;
  create_time?: Date | string;
}

// 训练超参数类型
interface TrainingHyperparameters {
  lr0: number;
  lrf: number;
  momentum: number;
  weight_decay: number;
  warmup_epochs: number;
  warmup_momentum: number;
  warmup_bias_lr: number;
  box: number;
  cls: number;
  cls_pw: number;
  obj: number;
  obj_pw: number;
  iou_t: number;
  anchor_t: number;
  fl_gamma: number;
  hsv_h: number;
  hsv_s: number;
  hsv_v: number;
  degrees: number;
  translate: number;
  scale: number;
  shear: number;
  perspective: number;
  flipud: number;
  fliplr: number;
  mosaic: number;
  mixup: number;
  copy_paste: number;
}

// 训练细节类型
interface TrainingDetails {
  hyp: TrainingHyperparameters;
  epochs: number;
  batch_size: number;
  imgsz: number;
  rect: boolean;
  resume: boolean;
  nosave: boolean;
  noval: boolean;
  noautoanchor: boolean;
  noplots: boolean;
  evolve: any;
  bucket: string;
  cache: any;
  image_weights: boolean;
  device: string;
  multi_scale: boolean;
  single_cls: boolean;
  optimizer: string;
  sync_bn: boolean;
  workers: number;
  project: string;
  name: string;
  exist_ok: boolean;
  quad: boolean;
  cos_lr: boolean;
  label_smoothing: number;
  patience: number;
  freeze: number[];
  save_period: number;
  seed: number;
  local_rank: number;
  entity: any;
  upload_dataset: boolean;
  bbox_interval: number;
  artifact_alias: string;
  save_dir: string;
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
  if (trainStep.value === 2) {
    fetchDefaultModels();
  }
}
function prevTrainStep() {
  trainStep.value -= 1;
}
const selectedTrainDataset = ref<Dataset | null>(null);
function selectTrainDataset(dataset: Dataset) {
  selectedTrainDataset.value = dataset;
}

// 默认模型列表
const defaultModels = ref<DefaultModel[]>([]);
const selectedDefaultModel = ref<DefaultModel | null>(null);
const defaultModelsLoading = ref(false);
const defaultModelsError = ref('');

// 获取默认模型列表
async function fetchDefaultModels() {
  if (!selectedTrainDataset.value) return;

  defaultModelsLoading.value = true;
  defaultModelsError.value = '';
  try {
    const token = localStorage.getItem('token');
    const res = await axios.post(
      'http://127.0.0.1:8000/v1/model/getdefault/',
      {
        filter: {
          task: selectedTrainDataset.value.task
        }
      },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (res.data.code === 200) {
      defaultModels.value = res.data.models;
    } else {
      defaultModelsError.value = res.data.message || '加载失败';
    }
  } catch {
    defaultModelsError.value = '网络错误';
  } finally {
    defaultModelsLoading.value = false;
  }
}

// 选择默认模型
function selectDefaultModel(model: DefaultModel) {
  selectedDefaultModel.value = model;
}

// 训练细节
const trainingDetails = ref<TrainingDetails>({
  hyp: {
    lr0: 0.01,
    lrf: 0.01,
    momentum: 0.937,
    weight_decay: 0.0005,
    warmup_epochs: 3.0,
    warmup_momentum: 0.8,
    warmup_bias_lr: 0.1,
    box: 0.05,
    cls: 0.5,
    cls_pw: 1.0,
    obj: 1.0,
    obj_pw: 1.0,
    iou_t: 0.2,
    anchor_t: 4.0,
    fl_gamma: 0.0,
    hsv_h: 0.015,
    hsv_s: 0.7,
    hsv_v: 0.4,
    degrees: 0.0,
    translate: 0.1,
    scale: 0.5,
    shear: 0.0,
    perspective: 0.0,
    flipud: 0.0,
    fliplr: 0.5,
    mosaic: 1.0,
    mixup: 0.0,
    copy_paste: 0.0
  },
  epochs: 1000,
  batch_size: 4,
  imgsz: 1280,
  rect: false,
  resume: false,
  nosave: false,
  noval: false,
  noautoanchor: false,
  noplots: false,
  evolve: null,
  bucket: '',
  cache: null,
  image_weights: false,
  device: '0',
  multi_scale: false,
  single_cls: false,
  optimizer: 'SGD',
  sync_bn: false,
  workers: 8,
  project: 'runs\\train',
  name: 'exp',
  exist_ok: false,
  quad: false,
  cos_lr: false,
  label_smoothing: 0.0,
  patience: 100,
  freeze: [0],
  save_period: -1,
  seed: 0,
  local_rank: -1,
  entity: null,
  upload_dataset: false,
  bbox_interval: -1,
  artifact_alias: 'latest',
  save_dir: 'runs\\train\\exp'
});

// 是否显示高级训练参数
const showAdvancedParams = ref(false);

// 切换显示高级训练参数
function toggleAdvancedParams() {
  showAdvancedParams.value = !showAdvancedParams.value;
}

// 训练进度模态框状态
const showTrainingProgress = ref(false);
const currentTrainingModel = ref('');
const currentTrainingDataset = ref('');
const taskType = ref('');
const currentModelId = ref('');

// 开始训练
async function startTraining() {
  if (!selectedTrainDataset.value || !selectedDefaultModel.value) return;

  loading.value = true;
  error.value = '';

  try {
    const token = localStorage.getItem('token');

    // 生成时间戳作为唯一标识
    const timestamp = new Date().getTime();

    // 生成项目名称：模型名称_数据集名称_时间戳
    const projectName = `${selectedDefaultModel.value.name}_${selectedTrainDataset.value.name}_${timestamp}`;

    // 更新训练参数中的存储路径
    trainingDetails.value.project = 'runs/train';
    trainingDetails.value.name = projectName;
    trainingDetails.value.save_dir = `runs/train/${projectName}`;

    const res = await axios.post(
      'http://127.0.0.1:8000/v1/model/train/',
      {
        model_id: selectedDefaultModel.value.id,
        datasets_id: selectedTrainDataset.value.id,
        train_details: trainingDetails.value
      },
      { headers: { Authorization: `Bearer ${token}` } }
    );

    if (res.data.code === 200) {
      // 训练成功，显示训练进度窗口
      currentTrainingModel.value = selectedDefaultModel.value.name;
      currentTrainingDataset.value = selectedTrainDataset.value.name;
      taskType.value = selectedTrainDataset.value.task as string;
      currentModelId.value = selectedDefaultModel.value.id as string;
      showTrainingProgress.value = true;
      closeTrainDialog();
    } else if (res.data.code === 401) {
      error.value = '登录已过期，请重新登录';
    } else {
      error.value = res.data.message || '训练失败';
    }
  } catch (err) {
    console.error('训练请求失败:', err);
    error.value = '网络错误，请稍后重试';
  } finally {
    loading.value = false;
  }
}

// 关闭训练进度窗口
function handleTrainingProgressClose() {
  showTrainingProgress.value = false;
  // 刷新模型列表
  fetchModels();
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
        <div style="display: flex; height: 420px; background: #f8fafb; border-radius: 12px; overflow: hidden">
          <!-- 左侧模型列表 -->
          <div style="width: 270px; border-right: 1px solid #eee; overflow-y: auto; background: #fff; padding: 8px 0">
            <div
              v-if="defaultModelsLoading"
              style="display: flex; justify-content: center; align-items: center; height: 100%"
            >
              <span style="color: #888; font-size: 16px">加载中...</span>
            </div>
            <div
              v-else-if="defaultModelsError"
              style="display: flex; justify-content: center; align-items: center; height: 100%"
            >
              <span style="color: #ff4d4f; font-size: 16px">{{ defaultModelsError }}</span>
            </div>
            <div
              v-else-if="defaultModels.length === 0"
              style="display: flex; justify-content: center; align-items: center; height: 100%"
            >
              <span style="color: #888; font-size: 16px">没有找到符合条件的模型</span>
            </div>
            <template v-else>
              <div
                v-for="model in defaultModels"
                :key="model.id"
                :style="{
                  display: 'flex',
                  alignItems: 'center',
                  gap: '14px',
                  padding: '14px 12px',
                  margin: '8px 12px',
                  borderRadius: '10px',
                  cursor: 'pointer',
                  background:
                    selectedDefaultModel && selectedDefaultModel.id === model.id
                      ? 'var(--soy-primary-color, #646cff20)'
                      : '#f8fafb',
                  fontWeight: selectedDefaultModel && selectedDefaultModel.id === model.id ? 'bold' : 'normal',
                  border:
                    selectedDefaultModel && selectedDefaultModel.id === model.id
                      ? '2px solid var(--soy-primary-color, #646cff)'
                      : '2px solid transparent',
                  boxShadow:
                    selectedDefaultModel && selectedDefaultModel.id === model.id
                      ? '0 2px 8px var(--soy-primary-color, #646cff22)'
                      : 'none',
                  transition: 'all 0.2s'
                }"
                @click="selectDefaultModel(model)"
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
                        selectedDefaultModel && selectedDefaultModel.id === model.id
                          ? 'var(--soy-primary-color, #646cff20)'
                          : '#f8fafb';
                  }
                "
              >
                <img
                  :src="model.thumbnail || placeholderImg"
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
                    {{ model.name }}
                  </div>
                  <div
                    style="font-size: 12px; color: #888; white-space: nowrap; overflow: hidden; text-overflow: ellipsis"
                  >
                    {{ taskMap[model.task as keyof typeof taskMap] || model.task }}
                  </div>
                </div>
              </div>
            </template>
          </div>
          <!-- 右侧预览信息 -->
          <div style="flex: 1; padding: 32px 32px 24px 32px; overflow-y: auto; background: #f8fafb">
            <template v-if="selectedDefaultModel">
              <div style="font-size: 22px; font-weight: bold; margin-bottom: 8px; color: #222; letter-spacing: 1px">
                {{ selectedDefaultModel.name }}
              </div>
              <div
                style="margin-bottom: 12px; color: var(--soy-primary-color, #646cff); font-weight: 500; font-size: 15px"
              >
                {{ taskMap[selectedDefaultModel.task as keyof typeof taskMap] || selectedDefaultModel.task }}
              </div>
              <div style="margin-bottom: 12px; color: #666">
                描述:
                <span style="color: #444">{{ selectedDefaultModel.description || '无' }}</span>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                大小:
                <span style="color: #444">{{ formatSize(selectedDefaultModel.size) }}</span>
              </div>
              <div style="margin-bottom: 12px; color: #666">
                创建时间:
                <span style="color: #444">{{ formatTime(selectedDefaultModel.create_time) }}</span>
              </div>

              <!-- 数据集信息 -->
              <div style="margin-top: 24px; padding-top: 16px; border-top: 1px dashed #ddd">
                <div style="font-size: 18px; font-weight: bold; margin-bottom: 12px; color: #222">训练数据集</div>
                <div style="display: flex; align-items: center; gap: 12px; margin-bottom: 12px">
                  <img
                    :src="selectedTrainDataset?.thumbnail || placeholderImg"
                    alt="thumb"
                    style="
                      width: 40px;
                      height: 40px;
                      object-fit: cover;
                      border-radius: 6px;
                      background: #f0f0f0;
                      flex-shrink: 0;
                    "
                    @error="
                      (e: Event) => {
                        const t = e.target as HTMLImageElement | null;
                        if (t) t.src = placeholderImg;
                      }
                    "
                  />
                  <div>
                    <div style="font-weight: 500">{{ selectedTrainDataset?.name }}</div>
                    <div style="font-size: 12px; color: #888">
                      {{ taskMap[selectedTrainDataset?.task as keyof typeof taskMap] || selectedTrainDataset?.task }}
                    </div>
                  </div>
                </div>
                <div style="display: flex; gap: 16px; margin-bottom: 8px; font-size: 13px; color: #666">
                  <div>
                    类别数:
                    <b>{{ selectedTrainDataset?.class_count }}</b>
                  </div>
                  <div>
                    图片数:
                    <b>{{ selectedTrainDataset?.image_count }}</b>
                  </div>
                </div>
              </div>

              <!-- 训练参数 -->
              <div style="margin-top: 24px; padding-top: 16px; border-top: 1px dashed #ddd">
                <div style="font-size: 18px; font-weight: bold; margin-bottom: 12px; color: #222">训练参数</div>

                <!-- 基本参数 -->
                <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 16px">
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">训练轮数 (epochs)</div>
                    <input
                      v-model="trainingDetails.epochs"
                      type="number"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">批次大小 (batch_size)</div>
                    <input
                      v-model="trainingDetails.batch_size"
                      type="number"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">图像尺寸 (imgsz)</div>
                    <input
                      v-model="trainingDetails.imgsz"
                      type="number"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">优化器 (optimizer)</div>
                    <select
                      v-model="trainingDetails.optimizer"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    >
                      <option value="SGD">SGD</option>
                      <option value="Adam">Adam</option>
                      <option value="AdamW">AdamW</option>
                    </select>
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">设备 (device)</div>
                    <input
                      v-model="trainingDetails.device"
                      type="text"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">标签平滑 (label_smoothing)</div>
                    <input
                      v-model="trainingDetails.label_smoothing"
                      type="number"
                      step="0.01"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">早停耐心值 (patience)</div>
                    <input
                      v-model="trainingDetails.patience"
                      type="number"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                  <div>
                    <div style="font-size: 13px; color: #666; margin-bottom: 4px">数据加载线程数 (workers)</div>
                    <input
                      v-model="trainingDetails.workers"
                      type="number"
                      style="width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 14px"
                    />
                  </div>
                </div>

                <!-- 高级参数切换按钮 -->
                <div style="margin-bottom: 12px">
                  <button
                    style="
                      background: none;
                      border: none;
                      color: var(--soy-primary-color, #646cff);
                      font-size: 13px;
                      cursor: pointer;
                      display: flex;
                      align-items: center;
                      gap: 4px;
                    "
                    @click="toggleAdvancedParams"
                  >
                    {{ showAdvancedParams ? '隐藏高级参数' : '显示高级参数' }}
                    <span style="font-size: 12px">{{ showAdvancedParams ? '▼' : '▶' }}</span>
                  </button>
                </div>

                <!-- 高级参数 -->
                <div
                  v-if="showAdvancedParams"
                  style="margin-top: 12px; padding: 12px; background: #f0f2f5; border-radius: 8px"
                >
                  <div style="font-size: 14px; font-weight: 500; margin-bottom: 8px; color: #333">学习率参数</div>
                  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 12px">
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">初始学习率 (lr0)</div>
                      <input
                        v-model="trainingDetails.hyp.lr0"
                        type="number"
                        step="0.001"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">最终学习率 (lrf)</div>
                      <input
                        v-model="trainingDetails.hyp.lrf"
                        type="number"
                        step="0.001"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">预热轮次 (warmup_epochs)</div>
                      <input
                        v-model="trainingDetails.hyp.warmup_epochs"
                        type="number"
                        step="0.1"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                  </div>

                  <div style="font-size: 14px; font-weight: 500; margin-bottom: 8px; color: #333">优化器参数</div>
                  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 12px">
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">动量 (momentum)</div>
                      <input
                        v-model="trainingDetails.hyp.momentum"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">权重衰减 (weight_decay)</div>
                      <input
                        v-model="trainingDetails.hyp.weight_decay"
                        type="number"
                        step="0.0001"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                  </div>

                  <div style="font-size: 14px; font-weight: 500; margin-bottom: 8px; color: #333">损失函数权重</div>
                  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 12px">
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">边界框损失 (box)</div>
                      <input
                        v-model="trainingDetails.hyp.box"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">分类损失 (cls)</div>
                      <input
                        v-model="trainingDetails.hyp.cls"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">目标损失 (obj)</div>
                      <input
                        v-model="trainingDetails.hyp.obj"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">锚框匹配阈值 (anchor_t)</div>
                      <input
                        v-model="trainingDetails.hyp.anchor_t"
                        type="number"
                        step="0.1"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                  </div>

                  <div style="font-size: 14px; font-weight: 500; margin-bottom: 8px; color: #333">数据增强参数</div>
                  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 12px">
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">HSV-H (hsv_h)</div>
                      <input
                        v-model="trainingDetails.hyp.hsv_h"
                        type="number"
                        step="0.001"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">HSV-S (hsv_s)</div>
                      <input
                        v-model="trainingDetails.hyp.hsv_s"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">HSV-V (hsv_v)</div>
                      <input
                        v-model="trainingDetails.hyp.hsv_v"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">旋转角度 (degrees)</div>
                      <input
                        v-model="trainingDetails.hyp.degrees"
                        type="number"
                        step="0.1"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">平移 (translate)</div>
                      <input
                        v-model="trainingDetails.hyp.translate"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">缩放 (scale)</div>
                      <input
                        v-model="trainingDetails.hyp.scale"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">水平翻转 (fliplr)</div>
                      <input
                        v-model="trainingDetails.hyp.fliplr"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                    <div>
                      <div style="font-size: 12px; color: #666; margin-bottom: 2px">马赛克 (mosaic)</div>
                      <input
                        v-model="trainingDetails.hyp.mosaic"
                        type="number"
                        step="0.01"
                        style="
                          width: 100%;
                          padding: 4px 6px;
                          border: 1px solid #ddd;
                          border-radius: 4px;
                          font-size: 13px;
                        "
                      />
                    </div>
                  </div>
                </div>
              </div>
            </template>
            <template v-else>
              <div style="color: #aaa; font-size: 16px; text-align: center; margin-top: 80px">
                请选择左侧模型查看详情
              </div>
            </template>
          </div>
        </div>
        <div style="display: flex; justify-content: flex-end; gap: 12px; margin-top: 16px">
          <NButton @click="prevTrainStep">上一步</NButton>
          <NButton
            type="primary"
            :disabled="!selectedDefaultModel || loading"
            :loading="loading"
            @click="startTraining"
          >
            {{ loading ? '正在启动训练...' : '开始训练' }}
          </NButton>
        </div>
        <!-- 显示错误信息 -->
        <div v-if="error" style="margin-top: 12px; color: #ff4d4f; text-align: center">
          {{ error }}
        </div>
      </template>
    </NModal>

    <!-- 训练进度模态框 -->
    <TrainingProgressModal
      v-model:show="showTrainingProgress"
      :model-name="currentTrainingModel"
      :dataset-name="currentTrainingDataset"
      :task-type="taskType"
      :model-id="currentModelId"
      @close="handleTrainingProgressClose"
    />

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
