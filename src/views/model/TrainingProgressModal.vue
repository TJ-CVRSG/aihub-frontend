<script setup lang="ts">
import { type Ref, computed, h, nextTick, onMounted, onUnmounted, ref, watch } from 'vue';
import { NButton, NModal, NProgress } from 'naive-ui';
import { Icon } from '@iconify/vue';
import * as echarts from 'echarts';

// 定义props
const props = defineProps<{
  show: boolean;
  modelName: string;
  datasetName: string;
  taskType: string;
  modelId: string | number;
  totalEpochs?: number; // 添加总轮次参数，可选
}>();

// 定义emit
const emit = defineEmits(['update:show', 'close']);

// 训练状态
const trainingStatus = ref('准备中...');
const currentEpoch = ref(0);
const totalEpochs = ref(props.totalEpochs || 1000); // 使用props值，如果没有则默认1000

// 监听props.totalEpochs变化
watch(
  () => props.totalEpochs,
  newValue => {
    if (newValue) {
      totalEpochs.value = newValue;
    }
  }
);

// 图表引用
const metricsChartRef = ref<HTMLElement | null>(null);
const boxLossChartRef = ref<HTMLElement | null>(null);
const clsLossChartRef = ref<HTMLElement | null>(null);
const dflLossChartRef = ref<HTMLElement | null>(null);

// 图表实例
let metricsChart: echarts.ECharts | null = null;
let boxLossChart: echarts.ECharts | null = null;
let clsLossChart: echarts.ECharts | null = null;
let dflLossChart: echarts.ECharts | null = null;

// WebSocket连接
let ws: WebSocket | null = null;

// 训练数据存储
const trainingData = ref<{
  epochs: number[];
  trainBoxLoss: number[];
  trainClsLoss: number[];
  trainDflLoss: number[];
  valBoxLoss: number[];
  valClsLoss: number[];
  valDflLoss: number[];
  metrics: {
    precision: number[];
    recall: number[];
    mAP50: number[];
    mAP5095: number[];
  };
}>({
  epochs: [],
  trainBoxLoss: [],
  trainClsLoss: [],
  trainDflLoss: [],
  valBoxLoss: [],
  valClsLoss: [],
  valDflLoss: [],
  metrics: {
    precision: [],
    recall: [],
    mAP50: [],
    mAP5095: []
  }
});

// 根据任务类型判断是否显示特定指标
const taskMetrics = computed(() => {
  switch (props.taskType) {
    case 'detect':
      return {
        // 训练损失指标（带前缀）
        trainLoss: ['train/box_loss', 'train/dfl_loss', 'train/cls_loss'],
        // 验证损失指标（带前缀）
        valLoss: ['val/box_loss', 'val/dfl_loss', 'val/cls_loss'],
        // 评估指标（带前缀）
        metrics: ['metrics/precision', 'metrics/recall', 'metrics/mAP_0.5', 'metrics/mAP_0.5:0.95'],
        // 学习率指标（带前缀）
        learningRates: ['x/lr0', 'x/lr1', 'x/lr2']
      };
    case 'classify':
      return {
        trainLoss: ['train/cls_loss'],
        valLoss: ['val/cls_loss'],
        metrics: ['metrics/accuracy', 'metrics/precision', 'metrics/recall', 'metrics/f1'],
        learningRates: ['x/lr0', 'x/lr1', 'x/lr2']
      };
    case 'segment':
      return {
        trainLoss: ['train/box_loss', 'train/dfl_loss', 'train/cls_loss', 'train/seg_loss'],
        valLoss: ['val/box_loss', 'val/dfl_loss', 'val/cls_loss', 'val/seg_loss'],
        metrics: ['metrics/precision', 'metrics/recall', 'metrics/mAP_0.5', 'metrics/mAP_0.5:0.95', 'metrics/mIoU'],
        learningRates: ['x/lr0', 'x/lr1', 'x/lr2']
      };
    case 'pose':
      return {
        trainLoss: ['train/box_loss', 'train/pose_loss', 'train/dfl_loss', 'train/cls_loss'],
        valLoss: ['val/box_loss', 'val/pose_loss', 'val/dfl_loss', 'val/cls_loss'],
        metrics: ['metrics/precision', 'metrics/recall', 'metrics/mAP_0.5', 'metrics/mAP_0.5:0.95'],
        learningRates: ['x/lr0', 'x/lr1', 'x/lr2']
      };
    default:
      return {
        trainLoss: ['train/cls_loss'],
        valLoss: ['val/cls_loss'],
        metrics: ['metrics/accuracy'],
        learningRates: ['x/lr0', 'x/lr1', 'x/lr2']
      };
  }
});

// 重置训练数据
const resetTrainingData = () => {
  trainingData.value = {
    epochs: [],
    trainBoxLoss: [],
    trainClsLoss: [],
    trainDflLoss: [],
    valBoxLoss: [],
    valClsLoss: [],
    valDflLoss: [],
    metrics: {
      precision: [],
      recall: [],
      mAP50: [],
      mAP5095: []
    }
  };
  currentEpoch.value = 0;
  trainingStatus.value = '准备中...';
};

// 初始化图表
const initCharts = () => {
  // 重置训练数据
  resetTrainingData();

  console.log('开始初始化图表, DOM引用状态:', {
    metricsChartRef: Boolean(metricsChartRef.value),
    boxLossChartRef: Boolean(boxLossChartRef.value),
    clsLossChartRef: Boolean(clsLossChartRef.value),
    dflLossChartRef: Boolean(dflLossChartRef.value)
  });

  console.log('当前任务类型:', props.taskType, '使用指标:', taskMetrics.value);

  // 获取当前任务类型的指标配置
  const metrics = taskMetrics.value;

  // 更改图表DOM容器标题，而不是在图表内添加标题
  const updateChartContainer = (container: HTMLElement | null, title: string, isVisible: boolean) => {
    if (!container || !container.parentElement) return;

    // 查找或创建标题元素
    let titleElement = container.parentElement.querySelector('.chart-title') as HTMLElement;
    if (!titleElement) {
      titleElement = document.createElement('div');
      titleElement.className = 'chart-title';
      titleElement.style.fontWeight = 'bold';
      titleElement.style.fontSize = '14px';
      titleElement.style.marginBottom = '8px';
      titleElement.style.textAlign = 'left';
      container.parentElement.insertBefore(titleElement, container);
    }

    // 设置标题并控制显示状态
    titleElement.textContent = title;
    container.parentElement.style.display = isVisible ? 'block' : 'none';
  };

  // 为确保所有DOM元素已渲染，使用nextTick
  nextTick(() => {
    // 初始化指标图表
    if (metricsChartRef.value) {
      // 添加标题
      updateChartContainer(metricsChartRef.value, 'Metrics', true);

      if (metricsChart) metricsChart.dispose();
      metricsChart = echarts.init(metricsChartRef.value);

      // 根据任务类型构建指标系列
      const metricsLegendData: string[] = [];
      const metricsSeriesConfig: {
        name: string;
        type: string;
        data: any[];
        smooth: boolean;
        symbolSize: number;
        lineStyle: { width: number };
      }[] = [];

      // 映射指标名称
      const metricNameMap: Record<string, string> = {
        'metrics/precision': 'Precision',
        'metrics/recall': 'Recall',
        'metrics/mAP_0.5': 'mAP50',
        'metrics/mAP_0.5:0.95': 'mAP50-95',
        'metrics/accuracy': 'Accuracy',
        'metrics/f1': 'F1 Score',
        'metrics/mIoU': 'mIoU'
      };

      // 直接使用taskMetrics中的所有metrics
      metrics.metrics.forEach(metricKey => {
        // 获取指标显示名称
        const displayName = metricNameMap[metricKey] || metricKey.replace('metrics/', '');

        // 添加到图表配置
        metricsSeriesConfig.push({
          name: displayName,
          type: 'line',
          data: [],
          smooth: true,
          symbolSize: 5,
          lineStyle: { width: 2 }
        });
        metricsLegendData.push(displayName);
      });

      const metricsOption = {
        tooltip: {
          trigger: 'axis'
        },
        legend: {
          data: metricsLegendData
        },
        grid: {
          top: 30,
          left: '3%',
          right: '4%',
          bottom: '3%',
          containLabel: true
        },
        xAxis: {
          type: 'category',
          data: trainingData.value.epochs
        },
        yAxis: {
          type: 'value'
        },
        series: metricsSeriesConfig
      };
      metricsChart.setOption(metricsOption);
      console.log('指标图表初始化完成, 使用指标:', metricsLegendData);
    } else {
      console.warn('指标图表DOM引用不存在');
    }

    // 通用的损失图表配置
    const lossChartOption = {
      tooltip: {
        trigger: 'axis'
      },
      legend: {
        data: ['Train', 'Val']
      },
      grid: {
        top: 30, // 为标题留出足够空间
        left: '3%',
        right: '4%',
        bottom: '3%',
        containLabel: true
      },
      xAxis: {
        type: 'category',
        data: trainingData.value.epochs
      },
      yAxis: {
        type: 'value'
      },
      series: [
        {
          name: 'Train',
          type: 'line',
          data: [],
          smooth: true,
          symbolSize: 5,
          lineStyle: { width: 2 }
        },
        {
          name: 'Val',
          type: 'line',
          data: [],
          smooth: true,
          symbolSize: 5,
          lineStyle: { width: 2 }
        }
      ]
    };

    // 损失指标映射
    const lossChartMap: Record<
      string,
      {
        ref: Ref<HTMLElement | null>;
        chart: echarts.ECharts | null;
        displayName: string;
        trainKey: string;
        valKey: string;
      }
    > = {
      box_loss: {
        ref: boxLossChartRef,
        chart: boxLossChart,
        displayName: 'Box Loss',
        trainKey: 'train/box_loss',
        valKey: 'val/box_loss'
      },
      cls_loss: {
        ref: clsLossChartRef,
        chart: clsLossChart,
        displayName: 'Class Loss',
        trainKey: 'train/cls_loss',
        valKey: 'val/cls_loss'
      },
      dfl_loss: {
        ref: dflLossChartRef,
        chart: dflLossChart,
        displayName: 'DFL Loss',
        trainKey: 'train/dfl_loss',
        valKey: 'val/dfl_loss'
      }
    };

    // 获取任务需要的损失指标（去掉前缀）
    const requiredLossMetrics = metrics.trainLoss.map(key => key.replace('train/', ''));

    // 初始化损失图表
    Object.keys(lossChartMap).forEach(lossType => {
      const chartInfo = lossChartMap[lossType];
      const isRequired = requiredLossMetrics.includes(lossType);
      const domRef = chartInfo.ref.value;

      if (domRef) {
        // 更新容器和标题
        updateChartContainer(domRef, chartInfo.displayName, isRequired);

        // 如果不需要此图表，跳过初始化
        if (!isRequired) return;

        // 总是先清理旧图表
        if (chartInfo.chart) {
          chartInfo.chart.dispose();
        }

        // 初始化图表
        const newChart = echarts.init(domRef);
        newChart.setOption(lossChartOption);

        // 更新全局图表引用
        if (lossType === 'box_loss') boxLossChart = newChart;
        if (lossType === 'cls_loss') clsLossChart = newChart;
        if (lossType === 'dfl_loss') dflLossChart = newChart;

        console.log(`${chartInfo.displayName}图表初始化完成`);
      }
    });
  });
};

// 构建指标图表配置
const buildMetricsConfig = () => {
  const metricsSeriesConfig = [];
  const metricsLegendData = [];
  const metrics = taskMetrics.value;

  if (metrics.metrics.includes('metrics/precision')) {
    metricsSeriesConfig.push({
      name: 'Precision',
      type: 'line',
      data: trainingData.value.metrics.precision,
      smooth: true
    });
    metricsLegendData.push('Precision');
  }

  if (metrics.metrics.includes('metrics/recall')) {
    metricsSeriesConfig.push({
      name: 'Recall',
      type: 'line',
      data: trainingData.value.metrics.recall,
      smooth: true
    });
    metricsLegendData.push('Recall');
  }

  if (metrics.metrics.includes('metrics/mAP_0.5')) {
    metricsSeriesConfig.push({
      name: 'mAP50',
      type: 'line',
      data: trainingData.value.metrics.mAP50,
      smooth: true
    });
    metricsLegendData.push('mAP50');
  }

  if (metrics.metrics.includes('metrics/mAP_0.5:0.95')) {
    metricsSeriesConfig.push({
      name: 'mAP50-95',
      type: 'line',
      data: trainingData.value.metrics.mAP5095,
      smooth: true
    });
    metricsLegendData.push('mAP50-95');
  }

  return { series: metricsSeriesConfig, legend: metricsLegendData };
};

// 更新损失图表
const updateLossCharts = () => {
  const metrics = taskMetrics.value;
  const hasBoxLoss = metrics.trainLoss.includes('train/box_loss');
  const hasClsLoss = metrics.trainLoss.includes('train/cls_loss');
  const hasDflLoss = metrics.trainLoss.includes('train/dfl_loss');

  if (boxLossChart && hasBoxLoss) {
    boxLossChart.setOption({
      xAxis: { data: trainingData.value.epochs },
      series: [{ data: trainingData.value.trainBoxLoss }, { data: trainingData.value.valBoxLoss }]
    });
    boxLossChart.resize();
    // 显示图表
    if (boxLossChartRef.value && boxLossChartRef.value.parentElement) {
      boxLossChartRef.value.parentElement.style.display = 'block';
    }
  } else if (boxLossChartRef.value && boxLossChartRef.value.parentElement) {
    // 隐藏图表
    boxLossChartRef.value.parentElement.style.display = 'none';
  }

  if (clsLossChart && hasClsLoss) {
    clsLossChart.setOption({
      xAxis: { data: trainingData.value.epochs },
      series: [{ data: trainingData.value.trainClsLoss }, { data: trainingData.value.valClsLoss }]
    });
    clsLossChart.resize();
    // 显示图表
    if (clsLossChartRef.value && clsLossChartRef.value.parentElement) {
      clsLossChartRef.value.parentElement.style.display = 'block';
    }
  } else if (clsLossChartRef.value && clsLossChartRef.value.parentElement) {
    // 隐藏图表
    clsLossChartRef.value.parentElement.style.display = 'none';
  }

  if (dflLossChart && hasDflLoss) {
    dflLossChart.setOption({
      xAxis: { data: trainingData.value.epochs },
      series: [{ data: trainingData.value.trainDflLoss }, { data: trainingData.value.valDflLoss }]
    });
    dflLossChart.resize();
    // 显示图表
    if (dflLossChartRef.value && dflLossChartRef.value.parentElement) {
      dflLossChartRef.value.parentElement.style.display = 'block';
    }
  } else if (dflLossChartRef.value && dflLossChartRef.value.parentElement) {
    // 隐藏图表
    dflLossChartRef.value.parentElement.style.display = 'none';
  }
};

// 更新图表数据
const updateCharts = (data: any, _forceUpdate = false) => {
  // 如果有新数据，更新数据源
  if (data) {
    // 使用Vue的响应式更新
    const newData = {
      epochs: [...trainingData.value.epochs, data.epoch],
      trainBoxLoss: [...trainingData.value.trainBoxLoss],
      trainClsLoss: [...trainingData.value.trainClsLoss],
      trainDflLoss: [...trainingData.value.trainDflLoss],
      valBoxLoss: [...trainingData.value.valBoxLoss],
      valClsLoss: [...trainingData.value.valClsLoss],
      valDflLoss: [...trainingData.value.valDflLoss],
      metrics: {
        precision: [...trainingData.value.metrics.precision],
        recall: [...trainingData.value.metrics.recall],
        mAP50: [...trainingData.value.metrics.mAP50],
        mAP5095: [...trainingData.value.metrics.mAP5095]
      }
    };

    // 使用taskMetrics处理指标数据
    const metrics = taskMetrics.value;

    // 处理训练损失
    metrics.trainLoss.forEach(key => {
      if (key in data) {
        if (key === 'train/box_loss') newData.trainBoxLoss.push(data[key]);
        if (key === 'train/dfl_loss') newData.trainDflLoss.push(data[key]);
        if (key === 'train/cls_loss') newData.trainClsLoss.push(data[key]);
      }
    });

    // 处理验证损失
    metrics.valLoss.forEach(key => {
      if (key in data) {
        if (key === 'val/box_loss') newData.valBoxLoss.push(data[key]);
        if (key === 'val/dfl_loss') newData.valDflLoss.push(data[key]);
        if (key === 'val/cls_loss') newData.valClsLoss.push(data[key]);
      }
    });

    // 处理评估指标
    metrics.metrics.forEach(key => {
      if (key in data) {
        if (key === 'metrics/precision') newData.metrics.precision.push(data[key]);
        if (key === 'metrics/recall') newData.metrics.recall.push(data[key]);
        if (key === 'metrics/mAP_0.5') newData.metrics.mAP50.push(data[key]);
        if (key === 'metrics/mAP_0.5:0.95') newData.metrics.mAP5095.push(data[key]);
      }
    });

    // 更新响应式数据
    trainingData.value = newData;
  }

  // 使用nextTick确保DOM更新后再更新图表
  nextTick(() => {
    console.log('更新图表数据, 当前数据长度:', trainingData.value.epochs.length);

    // 更新评估指标图表
    if (metricsChart) {
      const config = buildMetricsConfig();
      metricsChart.setOption({
        xAxis: {
          data: trainingData.value.epochs
        },
        legend: {
          data: config.legend
        },
        series: config.series
      });
      metricsChart.resize();
    } else {
      console.warn('更新失败: 指标图表未初始化');
    }

    // 更新损失图表
    updateLossCharts();
  });
};

// 处理训练状态更新
const handleTrainingStatus = (data: any) => {
  if (data.status === 'training' && data.data) {
    currentEpoch.value = data.data.epoch;
    trainingStatus.value = '训练中...';

    // 使用taskMetrics提取对应任务类型的指标
    const metrics = taskMetrics.value;

    // 打印当前任务类型和使用的指标
    console.log('当前任务类型:', props.taskType);
    console.log('使用指标集:', metrics);

    // 打印接收到的实际指标值（仅打印已定义的指标）
    const receivedValues: Record<string, number> = {};
    [...metrics.trainLoss, ...metrics.valLoss, ...metrics.metrics, ...metrics.learningRates].forEach(key => {
      if (key in data.data) {
        receivedValues[key] = data.data[key];
      }
    });
    console.log('接收到的指标值:', receivedValues);

    updateCharts(data.data);
  } else if (data.status === 'finished') {
    trainingStatus.value = '训练完成';
    if (data.data) updateCharts(data.data);
    ws?.close();
  } else if (data.status === 'suspend') {
    trainingStatus.value = '训练暂停';
  } else if (data.status === 'early_stop') {
    trainingStatus.value = '训练提前停止';
    if (data.data) {
      currentEpoch.value = data.data.epoch;
      updateCharts(data.data);
    }
    ws?.close();
  }
};

// 建立WebSocket连接
const connectWebSocket = () => {
  try {
    // 先断开旧连接
    if (ws) {
      console.log('断开旧WebSocket连接');
      ws.close();
    }

    console.log('尝试建立WebSocket连接:', `ws://localhost:8000/ws/training/${props.modelId.toString()}`);

    ws = new WebSocket(`ws://localhost:8000/ws/training/${props.modelId.toString()}`);

    console.log('WebSocket实例创建:', Boolean(ws));

    ws.onopen = () => {
      console.log('WebSocket连接成功,链接地址:', ws?.url);
    };

    ws.onmessage = event => {
      console.log('WebSocket收到消息, 原始数据:', event.data);
      try {
        const data = JSON.parse(event.data);

        // 根据状态类型进行不同处理
        if (data.status === 'training') {
          // 训练中状态的处理逻辑
          console.log('收到训练进度更新:', {
            状态: data.status,
            轮次: data.data?.epoch,
            训练损失: {
              box_loss: data.data?.['train/box_loss'],
              dfl_loss: data.data?.['train/dfl_loss'],
              cls_loss: data.data?.['train/cls_loss']
            },
            验证损失: {
              box_loss: data.data?.['val/box_loss'],
              dfl_loss: data.data?.['val/dfl_loss'],
              cls_loss: data.data?.['val/cls_loss']
            },
            评估指标: {
              precision: data.data?.['metrics/precision'],
              recall: data.data?.['metrics/recall'],
              mAP50: data.data?.['metrics/mAP_0.5'],
              mAP5095: data.data?.['metrics/mAP_0.5:0.95']
            }
          });
          handleTrainingStatus(data);
        } else if (data.status === 'finished' || data.status === 'early_stop') {
          // 训练完成或提前停止状态的处理逻辑
          console.log('收到训练结束消息:', {
            状态: data.status,
            模型ID: data.model_id,
            消息: data.message,
            最佳模型路径: data.best_model_path
          });

          // 更新训练状态
          trainingStatus.value = data.status === 'finished' ? '训练完成' : '训练提前停止';

          // 关闭WebSocket连接
          ws?.close();
        } else {
          // 其他状态直接传递给handleTrainingStatus处理
          handleTrainingStatus(data);
        }
      } catch (err) {
        console.error('处理WebSocket消息时出错:', err);
      }
    };

    ws.onerror = error => {
      console.error('WebSocket连接错误:', error);
      trainingStatus.value = '连接错误';
    };

    ws.onclose = event => {
      console.log('WebSocket连接关闭, 状态码:', event.code, '原因:', event.reason);
    };
  } catch (err) {
    console.error('创建WebSocket连接时出错:', err);
    trainingStatus.value = '连接初始化失败';
  }
};

// 关闭模态框
const handleClose = () => {
  if (ws) {
    ws.close();
  }
  emit('update:show', false);
  emit('close');
};

// 生命周期钩子
onMounted(() => {
  console.log('TrainingProgressModal挂载, show:', props.show, 'modelId:', props.modelId);
});

// 观察props.show的变化
watch(
  () => props.show,
  newVal => {
    console.log('训练进度窗口显示状态变化:', newVal);
    if (newVal) {
      // 如果模态框显示，先初始化图表再建立WebSocket连接
      nextTick(() => {
        initCharts();
        connectWebSocket();
      });
    } else {
      // 如果模态框关闭，断开WebSocket连接
      ws?.close();
    }
  },
  { immediate: true } // 组件创建时立即执行一次
);

// 使用taskMetrics更新图表视图
watch(
  () => props.taskType,
  () => {
    console.log('任务类型变更为:', props.taskType, '使用指标:', taskMetrics.value);
    // 任务类型变更时重新初始化图表
    if (props.show) {
      nextTick(() => {
        initCharts();
      });
    }
  },
  { immediate: true }
);

onUnmounted(() => {
  if (ws) {
    ws.close();
  }
  metricsChart?.dispose();
  boxLossChart?.dispose();
  clsLossChart?.dispose();
  dflLossChart?.dispose();
});
</script>

<template>
  <NModal
    :show="show"
    :title="`训练进度 - ${modelName} on ${datasetName}`"
    :icon="() => h(Icon, { icon: 'mdi:rocket-launch', style: 'font-size: 24px' })"
    preset="dialog"
    style="width: 1200px; border-radius: 16px; box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12)"
    @update:show="value => emit('update:show', value)"
  >
    <!-- 训练状态和进度 -->
    <div style="margin-bottom: 24px">
      <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px">
        <div style="font-size: 16px; color: #666">{{ trainingStatus }}</div>
        <div style="font-size: 16px; color: #666">当前轮次: {{ currentEpoch }}/{{ totalEpochs }}</div>
      </div>
      <NProgress
        type="line"
        :percentage="Number(((currentEpoch / totalEpochs) * 100).toFixed(1))"
        indicator-placement="inside"
        color="var(--soy-primary-color, #646cff)"
      />
    </div>

    <!-- 图表区域 -->
    <div style="display: flex; flex-direction: column; gap: 16px; margin-bottom: 24px">
      <!-- 指标图表容器 - 不包含标题，标题将通过JS动态添加 -->
      <div style="background: #f8fafb; border-radius: 12px; padding: 16px; height: 400px">
        <div ref="metricsChartRef" style="width: 100%; height: 340px" />
      </div>

      <!-- 损失图表区域 - 不包含标题，标题将通过JS动态添加 -->
      <div style="display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 16px">
        <div style="background: #f8fafb; border-radius: 12px; padding: 16px; height: 250px">
          <div ref="boxLossChartRef" style="width: 100%; height: 180px" />
        </div>
        <div style="background: #f8fafb; border-radius: 12px; padding: 16px; height: 250px">
          <div ref="clsLossChartRef" style="width: 100%; height: 180px" />
        </div>
        <div style="background: #f8fafb; border-radius: 12px; padding: 16px; height: 250px">
          <div ref="dflLossChartRef" style="width: 100%; height: 180px" />
        </div>
      </div>
    </div>

    <!-- 底部按钮 -->
    <div style="display: flex; justify-content: flex-end; gap: 12px">
      <NButton @click="handleClose">关闭</NButton>
    </div>
  </NModal>
</template>

<style scoped>
.n-modal {
  max-width: 90vw;
}
</style>
