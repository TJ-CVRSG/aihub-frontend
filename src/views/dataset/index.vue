<script setup lang="ts">
import { h, onMounted, ref } from 'vue';
import {
  NButton,
  NDataTable,
  NForm,
  NFormItem,
  NInput,
  NModal,
  NPagination,
  NSelect,
  NUpload,
  useMessage
} from 'naive-ui';
import axios from 'axios';
import { Icon } from '@iconify/vue';

const showModal = ref(false);
const formRef = ref();
const message = useMessage();

const formModel = ref({
  name: '',
  type: 0,
  zipPath: '', // 仅用于显示
  zipFile: null, // 用于存储文件对象
  desc: ''
});

const typeOptions = [
  { label: '目标检测', value: 0 },
  { label: '分类', value: 1 },
  { label: '旋转框检测', value: 2 },
  { label: '分割', value: 3 },
  { label: '姿态估计', value: 4 }
];

function openModal() {
  showModal.value = true;
}
function closeModal() {
  showModal.value = false;
}
const taskEnumMap: { [key: number]: string } = {
  0: 'detect',
  1: 'classify',
  2: 'obb',
  3: 'segment',
  4: 'pose'
};

async function handleSubmit() {
  formRef.value?.validate(async (errors: any) => {
    if (!errors) {
      try {
        const token = localStorage.getItem('token');
        const formData = new FormData();
        formData.append('name', formModel.value.name);
        formData.append('task', taskEnumMap[formModel.value.type]);
        if (!formModel.value.zipFile) {
          throw new Error('zipFile must not be null');
        }
        formData.append('file', formModel.value.zipFile);
        formData.append('description', formModel.value.desc || '');
        const res = await axios.post('http://127.0.0.1:8000/v1/dataset/upload/', formData, {
          headers: { Authorization: `Bearer ${token}` }
        });
        if (res.data.code === 200) {
          message.success('数据集上传成功');
          closeModal();
          fetchDatasets(); // 刷新数据集列表
        } else if (res.data.code === 401) {
          message.error('Token失效, 请重新登录');
        } else {
          message.error(res.data.message || '上传失败');
        }
      } catch {
        message.error('网络错误，上传失败');
      }
    }
  });
}
function handleZipChange({ file }: any) {
  formModel.value.zipFile = file.file; // 原生 File 对象
  formModel.value.zipPath = file.name; // 仅用于显示
}

// 数据集列表相关状态
const datasets = ref([]);
const total = ref(0);
const page = ref(1);
const pageSize = ref(10);
const loading = ref(false);
const error = ref('');

// 任务类型映射
const taskMap = { detect: '目标检测', classify: '分类', segment: '分割', pose: '姿态估计', rotate: '旋转框检测' };

interface RowData {
  name: string;
  task: keyof typeof taskMap;
  class_count: number;
  image_count: number;
  train_count: number;
  val_count: number;
  test_count: number;
  description: string;
  path: string;
  size: number;
  upload_time: Date;
}

const columns = [
  { title: '名称', key: 'name' },
  { title: '任务类型', key: 'task', render: (row: RowData) => taskMap[row.task] || row.task },
  { title: '类别数', key: 'class_count' },
  { title: '图片数', key: 'image_count' },
  {
    title: '训练/验证/测试',
    key: 'split',
    render: (row: RowData) => `${row.train_count}/${row.val_count}/${row.test_count}`
  },
  { title: '描述', key: 'description' },
  // { title: '路径', key: 'path' },
  { title: '大小', key: 'size', render: (row: RowData) => formatSize(row.size) },
  { title: '上传时间', key: 'upload_time', render: (row: RowData) => formatTime(row.upload_time) }
];

function formatSize(size: number) {
  if (size > 1024 * 1024) return `${(size / 1024 / 1024).toFixed(2)} MB`;
  if (size > 1024) return `${(size / 1024).toFixed(2)} KB`;
  return `${size} B`;
}
function formatTime(time: Date) {
  return new Date(time).toLocaleString();
}

const todayDatasetCount = ref(0);
const todayModelCount = ref(0);

function isToday(isoString: string) {
  if (!isoString) return false;
  const d = new Date(isoString);
  if (Number.isNaN(d.getTime())) return false;
  const now = new Date();
  return d.getFullYear() === now.getFullYear() && d.getMonth() === now.getMonth() && d.getDate() === now.getDate();
}

async function fetchDatasets() {
  loading.value = true;
  error.value = '';
  try {
    const token = localStorage.getItem('token');
    const res = await axios.post(
      'http://127.0.0.1:8000/v1/dataset/get/',
      { page: 1, page_size: 9999 },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (res.data.code === 200) {
      datasets.value = res.data.datasets;
      total.value = res.data.total;
      // 今日新增统计
      todayDatasetCount.value = res.data.datasets.filter((item: any) => isToday(item.upload_time)).length;
    } else {
      error.value = res.data.message || '加载失败';
    }
  } catch {
    error.value = '网络错误';
  } finally {
    loading.value = false;
  }
}

async function fetchTodayModelCount() {
  try {
    const token = localStorage.getItem('token');
    const res = await axios.post(
      'http://127.0.0.1:8000/v1/model/get/',
      { page: 1, page_size: 9999 },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    if (res.data.code === 200) {
      todayModelCount.value = res.data.models.filter((item: any) => isToday(item.create_time)).length;
    }
  } catch {}
}

onMounted(() => {
  fetchDatasets();
  fetchTodayModelCount();
});

function handlePageChange(p: number) {
  page.value = p;
  fetchDatasets();
}
</script>

<template>
  <div>
    <div class="upload-btn-wrapper">
      <NButton type="primary" @click="openModal">上传数据集</NButton>
    </div>
    <NModal
      v-model:show="showModal"
      title="上传数据集"
      :icon="() => h(Icon, { icon: 'mdi:database-cog', style: 'font-size: 24px' })"
      preset="dialog"
      @close="closeModal"
    >
      <NForm ref="formRef" :model="formModel" label-width="80">
        <NFormItem label="数据集名称" path="name" required>
          <NInput v-model:value="formModel.name" placeholder="请输入数据集名称" />
        </NFormItem>
        <NFormItem label="任务类型" path="type" required>
          <NSelect v-model:value="formModel.type" :options="typeOptions" placeholder="请选择任务类型" />
        </NFormItem>
        <NFormItem label="zip绝对路径" path="zipPath" required>
          <NUpload :max="1" accept=".zip" :default-upload="false" @change="handleZipChange">
            <NButton>选择zip文件</NButton>
          </NUpload>
          <div v-if="formModel.zipPath" class="zip-selected">已选择: {{ formModel.zipPath }}</div>
        </NFormItem>
        <NFormItem label="数据集描述" path="desc">
          <NInput v-model:value="formModel.desc" type="textarea" placeholder="请输入描述" />
        </NFormItem>
      </NForm>
      <template #action>
        <NButton @click="closeModal">取消</NButton>
        <NButton type="primary" @click="handleSubmit">提交</NButton>
      </template>
    </NModal>
    <NDataTable :columns="columns" :data="datasets" :loading="loading" />
    <NPagination
      v-model:page="page"
      :page-size="pageSize"
      :page-count="Math.ceil(total / pageSize)"
      @update:page="handlePageChange"
    />
  </div>
</template>

<style scoped>
.upload-btn-wrapper {
  display: flex;
  justify-content: flex-end;
}
.zip-selected {
  margin-top: 8px;
  color: #888;
  font-size: 12px;
}
</style>
