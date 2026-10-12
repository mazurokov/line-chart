<template>
  <div class="chart-wrapper">
    <div
      ref="scrollContainer"
      class="scroll-wrapper"
      @pointerdown="startDragging"
      @pointermove="doDrag"
      @pointerup="stopDragging"
      @pointercancel="stopDragging"
      @pointerleave="stopDragging"
      @wheel="handleWheel"
    >
      <div
        class="mileage-chart-container"
        :style="{ width: containerWidth, height: chartHeightPx }"
      >
        <Line
          v-if="imagesLoaded && chartPoints.length"
          :data="chartData"
          :options="chartOptions"
          :plugins="chartPlugins"
          :width="chartWidth"
          :height="chartHeight"
          :style="chartCanvasStyle"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import {
  ref,
  computed,
  onMounted,
  onBeforeUnmount,
  nextTick,
  watch,
} from 'vue';
import { Line } from 'vue-chartjs';
import {
  Chart as ChartJS,
  LineElement,
  PointElement,
  LinearScale,
  CategoryScale,
  Tooltip,
  Legend,
} from 'chart.js';
ChartJS.register(LineElement, PointElement, LinearScale, CategoryScale, Tooltip, Legend);

const props = defineProps({
  templateData: {
    type: Object,
    default: () => ({}),
  },
});

const COLUMN_WIDTH = 192;
const COLUMN_GAP = 12;
const GRAPH_HEIGHT = 256;
const PADDING_BOTTOM = 14;
const MAX_CHARS = 25;
const POINT_RADIUS = 13;

const TITLE_HEIGHT = 20;
const DESC_LINE_HEIGHT = 14;

const chart = computed(() => props.templateData?.component?.chart || {});

const chartPoints = computed(() => (chart.value?.values || []).map((point) => ({
  xValue: point?.xValue || '',
  yValue: point?.yValue === null || point?.yValue === undefined ? null : Number(point.yValue),
  state: point?.state || 'default',
  description: point?.description || '',
  extra1: point?.additionalYValues?.[0]?.content || '',
  extra2: point?.additionalYValues?.[1]?.content || '',
})));

const yValues = computed(() => chartPoints.value.map((point) => point.yValue));
const numericYValues = computed(() => yValues.value.filter((value) => Number.isFinite(value)));
const yLabelSuffix = computed(() => chart.value?.yLabelSuffix || '');
const pointTitleSuffix = computed(() => chart.value?.pointTitleSuffix || yLabelSuffix.value);
const Y_AXIS_STEP = 50;
const Y_AXIS_SMALL_RANGE_LIMIT = 100;
const Y_AXIS_TARGET_TICKS = 8;
const TOP_LAYOUT_PADDING = 0;
const TOP_LABEL_GAP = 18;
const TOP_LABEL_POINT_GAP = 6;
const minYValue = computed(() => (numericYValues.value.length ? Math.min(...numericYValues.value) : 0));
const maxYValue = computed(() => (numericYValues.value.length ? Math.max(...numericYValues.value) : 0));
const yAxisRange = computed(() => maxYValue.value - minYValue.value);
const getNiceStep = (range, targetTicks = Y_AXIS_TARGET_TICKS) => {
  const safeRange = Math.max(range, 1);
  const rawStep = safeRange / Math.max(targetTicks - 1, 1);
  const exponent = Math.floor(Math.log10(rawStep));
  const magnitude = 10 ** exponent;
  const fraction = rawStep / magnitude;
  const niceFraction = [1, 2, 5, 10].find((x) => fraction <= x);

  return niceFraction * magnitude;
};
const yAxisStep = computed(() => {
  if (!numericYValues.value.length) return 1;
  if (yAxisRange.value <= 5) return 1;
  if (yAxisRange.value <= 10) return 2;
  if (yAxisRange.value <= 20) return 5;
  if (yAxisRange.value <= 50) return 10;
  if (yAxisRange.value <= Y_AXIS_SMALL_RANGE_LIMIT) return 10;

  return getNiceStep(yAxisRange.value);
});
const horizontalGuideStep = computed(() => {
  if (!numericYValues.value.length) return 1;
  if (yAxisRange.value <= 5) return 1;
  if (yAxisRange.value <= 10) return 1;
  if (yAxisRange.value <= 20) return 2;
  if (yAxisRange.value <= 50) return 5;

  return Math.max(10, Math.round(yAxisStep.value / 2));
});
const yAxisLabelStep = computed(() => Math.max(yAxisStep.value, horizontalGuideStep.value * 2));

const getLinesCount = (text) => {
  if (!text) return 1;

  const words = text.split(/\s+/);
  const result = words.reduce((acc, word) => {
      const maxLines = 1;
    if (acc.lines >= maxLines) return acc;

    const wordLength = word.length;
    const addedLength = acc.currentLength === 0 ? wordLength : acc.currentLength + 1 + wordLength;

    if (addedLength <= MAX_CHARS) {
      return { ...acc, currentLength: addedLength };
    }

    return { lines: acc.lines + 1, currentLength: wordLength };
  }, { lines: 1, currentLength: 0 });

  const maxLines = 1;
  return Math.min(result.lines, maxLines);
};

const bottomSpace = computed(() => {
  if (!chartPoints.value.length) return TITLE_HEIGHT + PADDING_BOTTOM;
  const maxLines = Math.max(...chartPoints.value.map((item) => getLinesCount(item.description)));
  return TITLE_HEIGHT + (maxLines * DESC_LINE_HEIGHT) + PADDING_BOTTOM;
});

const splitTextLines = (text) => {
  const maxLines = 1;
  const words = (text || '').split(/\s+/).filter(Boolean);
  const lines = Array(maxLines).fill('');
  let curr = 0;

  words.forEach((word) => {
    if (curr < maxLines && (lines[curr] + word).length <= MAX_CHARS) {
      lines[curr] += (lines[curr] ? ' ' : '') + word;
    } else if (curr < maxLines - 1) {
      curr += 1;
      if ((lines[curr] + word).length <= MAX_CHARS - 2) {
        lines[curr] += (lines[curr] ? ' ' : '') + word;
      } else if (!lines[curr].endsWith('...')) {
        lines[curr] += '...';
      }
    } else if (!lines[curr].endsWith('...')) {
      lines[curr] += '...';
    }
  });

  return lines.filter((line) => line !== '');
};

const bottomLabelBgHeight = computed(() => {
  if (!chartPoints.value.length) return 0;

  const maxDescriptionLines = Math.max(
    ...chartPoints.value.map((item) => splitTextLines(item.description).length),
  );

  return 20 + (maxDescriptionLines * 14) + 8;
});

const canvasTopPadding = computed(() => TOP_LAYOUT_PADDING + 4);
const canvasBottomPadding = computed(() => bottomSpace.value + 4);
const yAxisMin = computed(() => {
  if (!numericYValues.value.length) return 0;
  if (yAxisRange.value <= Y_AXIS_SMALL_RANGE_LIMIT) {
    return Math.max(0, minYValue.value - Math.max(1, Math.ceil(yAxisRange.value * 0.1)));
  }

  return Math.max(0, Math.floor(minYValue.value / yAxisStep.value) * yAxisStep.value);
});
const yAxisMax = computed(() => {
  if (!numericYValues.value.length) return Y_AXIS_STEP;
  if (yAxisRange.value <= Y_AXIS_SMALL_RANGE_LIMIT) {
    return maxYValue.value + Math.max(1, Math.ceil(yAxisRange.value * 0.1));
  }

  return Math.ceil(maxYValue.value / yAxisStep.value) * yAxisStep.value;
});

const activeIndex = ref(null);
const imagesLoaded = ref(false);
const normalImg = ref(null);
const warningImg = ref(null);
const errorImg = ref(null);

const normalSvg = '<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><g clip-path="url(#clip0_45399_4079)"><rect width="24" height="24" rx="12" fill="white"/><path d="M12 18C15.3137 18 18 15.3137 18 12C18 8.68629 15.3137 6 12 6C8.68629 6 6 8.68629 6 12C6 15.3137 8.68629 18 12 18Z" fill="#1F2024"/><path fill-rule="evenodd" clip-rule="evenodd" d="M24 12C24 18.6274 18.6274 24 12 24C5.37258 24 0 18.6274 0 12C0 5.37258 5.37258 0 12 0C18.6274 0 24 5.37258 24 12ZM23 12C23 18.0751 18.0751 23 12 23C5.92487 23 1 18.0751 1 12C1 5.92487 5.92487 1 12 1C18.0751 1 23 5.92487 23 12Z" fill="#1F2024"/></g><defs><clipPath id="clip0_45399_4079"><rect width="24" height="24" rx="12" fill="white"/></clipPath></defs></svg>';
const warningSvg = '<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><rect width="24" height="24" rx="12" fill="white"/><circle cx="12" cy="12" r="10" fill="white"/><path fill-rule="evenodd" clip-rule="evenodd" d="M24 12C24 18.6274 18.6274 24 12 24C5.37258 24 0 18.6274 0 12C0 5.37258 5.37258 0 12 0C18.6274 0 24 5.37258 24 12ZM10.5 6.5C10.5 5.67157 11.1716 5 12 5C12.8284 5 13.5 5.67157 13.5 6.5V13.5C13.5 14.3284 12.8284 15 12 15C11.1716 15 10.5 14.3284 10.5 13.5V6.5ZM12 16C11.1716 16 10.5 16.6716 10.5 17.5C10.5 18.3284 11.1716 19 12 19C12.8284 19 13.5 18.3284 13.5 17.5C13.5 16.6716 12.8284 16 12 16Z" fill="#FE9800"/></svg>';
const errorSvg = '<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><circle cx="12" cy="12" r="10" fill="white"/><path fill-rule="evenodd" clip-rule="evenodd" d="M24 12C24 18.6274 18.6274 24 12 24C5.37258 24 0 18.6274 0 12C0 5.37258 5.37258 0 12 0C18.6274 0 24 5.37258 24 12ZM10.5 6.5C10.5 5.67157 11.1716 5 12 5C12.8284 5 13.5 5.67157 13.5 6.5V13.5C13.5 14.3284 12.8284 15 12 15C11.1716 15 10.5 14.3284 10.5 13.5V6.5ZM12 16C11.1716 16 10.5 16.6716 10.5 17.5C10.5 18.3284 11.1716 19 12 19C12.8284 19 13.5 18.3284 13.5 17.5C13.5 16.6716 12.8284 16 12 16Z" fill="#FF3333"/></svg>';

const pointLayouts = computed(() => {
  let cursor = 0;

  return chartPoints.value.map((_, index) => {
    const width = COLUMN_WIDTH;
    const start = cursor;
    cursor += width + (index < chartPoints.value.length - 1 ? COLUMN_GAP : 0);

    return {
      start,
      width,
      center: start + (width / 2),
    };
  });
});

const pointLayoutsTotalWidth = computed(() => (
  pointLayouts.value.length
    ? (() => {
      const lastLayout = pointLayouts.value[pointLayouts.value.length - 1];
      return lastLayout.start + lastLayout.width;
    })()
    : 0
));

const getLayoutPixels = (scale, layout) => {
  const left = scale.getPixelForValue(layout.start);
  const right = scale.getPixelForValue(layout.start + layout.width);

  return {
    left,
    right,
    width: right - left,
    center: left + ((right - left) / 2),
  };
};

const getYAxisLabelWidth = () => {
  if (import.meta.env.SSR || typeof document === 'undefined') return 0;

  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  if (!ctx) return 0;

  ctx.font = '12px "Roboto2", sans-serif';

  const labels = [];
  for (let { value } = yAxisMin; value <= yAxisMax.value; value += yAxisStep.value) {
    labels.push(`${Math.round(value)} ${yLabelSuffix.value}`);
  }

  return Math.ceil(Math.max(...labels.map((label) => ctx.measureText(label).width), 0)) + 12;
};

async function createImg(svg) {
  return new Promise((res) => {
    const img = new Image();
    img.src = `data:image/svg+xml;base64,${btoa(unescape(encodeURIComponent(svg)))}`;
    img.onload = () => res(img);
  });
}

const horizontalGuidePlugin = {
  id: 'horizontalGuidePlugin',
  beforeDatasetsDraw(chartInstance) {
    const {
      ctx,
      scales: { y },
      chartArea,
      width,
    } = chartInstance;

    if (!Number.isFinite(yAxisMin.value) || !Number.isFinite(yAxisMax.value)) return;

    ctx.save();
    ctx.strokeStyle = '#E5E7EB';
    ctx.lineWidth = 1;

    const start = Math.ceil(yAxisMin.value / horizontalGuideStep.value) * horizontalGuideStep.value;
    for (let value = start; value <= yAxisMax.value; value += horizontalGuideStep.value) {
      const yPos = y.getPixelForValue(value);
      if (yPos >= chartArea.top && yPos <= chartArea.bottom) {
        ctx.beginPath();
        ctx.moveTo(chartArea.left, yPos);
        ctx.lineTo(width, yPos);
        ctx.stroke();
      }
    }

    ctx.restore();
  },
};

const chartBorderPlugin = {
  id: 'chartBorderPlugin',
  beforeDatasetsDraw(chartInstance) {
    const {
      ctx,
      chartArea,
      width,
    } = chartInstance;

    ctx.save();
    ctx.strokeStyle = '#E5E7EB';
    ctx.lineWidth = 1;

    ctx.beginPath();
    ctx.moveTo(chartArea.left, chartArea.top);
    ctx.lineTo(width, chartArea.top);
    ctx.moveTo(width, chartArea.top);
    ctx.lineTo(width, chartArea.bottom);
    ctx.moveTo(chartArea.left, chartArea.bottom);
    ctx.lineTo(width, chartArea.bottom);
    ctx.stroke();

    ctx.restore();
  },
};

const highlightPlugin = {
  id: 'highlightPlugin',
  beforeDraw(chartInstance) {
    if (activeIndex.value !== null) {
      const { ctx, scales: { x }, chartArea: { top, bottom } } = chartInstance;
      const layout = pointLayouts.value[activeIndex.value];
      if (!layout) return;
      const { left, width } = getLayoutPixels(x, layout);
      ctx.save();
      ctx.fillStyle = 'rgba(232, 244, 253, 0.5)';
      ctx.fillRect(left, top, width, bottom - top);
      ctx.restore();
    }
  },
};

const customTopLabels = {
  id: 'customTopLabels',
  afterDraw(chartInstance) {
    if (!chartPoints.value.length) return;

    const { ctx, scales: { y }, chartArea } = chartInstance;

    ctx.save();
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';

    yValues.value.forEach((val, i) => {
      if (!Number.isFinite(val)) return;

      const layout = pointLayouts.value[i];
      if (!layout) return;
      const { center } = getLayoutPixels(chartInstance.scales.x, layout);
      const label = chartPoints.value[i];
      const hasExtraText = label?.extra1 || label?.extra2;
      const boxHeight = hasExtraText ? 76 : 24;
      const pointYPos = y.getPixelForValue(val);
      const minDistanceFromPoint = POINT_RADIUS + TOP_LABEL_POINT_GAP;
      const yPosAbove = pointYPos - (boxHeight + TOP_LABEL_GAP);
      const yPosBelow = pointYPos + minDistanceFromPoint;
      const canPlaceAbove = yPosAbove >= TOP_LAYOUT_PADDING;
      const canPlaceBelow = yPosBelow + boxHeight <= chartArea.bottom;
      const isPointAtTopEdge = !canPlaceAbove;
      const isPointAtBottomEdge = !canPlaceBelow;

      let yPos;

      if (isPointAtBottomEdge && canPlaceAbove) {
        yPos = yPosAbove;
      } else if (isPointAtTopEdge && canPlaceBelow) {
        yPos = yPosBelow;
      } else if (canPlaceAbove) {
        yPos = yPosAbove;
      } else {
        yPos = yPosBelow;
      }

      ctx.save();
      ctx.shadowColor = 'rgba(31, 32, 36, 0.1)';
      ctx.shadowBlur = 4;
      ctx.shadowOffsetY = 1;
      ctx.fillStyle = '#FFFFFF';

      const title = `${val} ${pointTitleSuffix.value}`;
      ctx.font = '700 14px "Roboto2", sans-serif';

      const contentWidth = Math.max(
        ctx.measureText(title).width,
        label?.extra1 ? ctx.measureText(label.extra1).width : 0,
        label?.extra2 ? ctx.measureText(label.extra2).width : 0,
      );
      const boxWidth = Math.min(Math.ceil(contentWidth) + 32, COLUMN_WIDTH - COLUMN_GAP);
      const boxX = center - (boxWidth / 2);

      ctx.beginPath();
      ctx.roundRect(boxX, yPos, boxWidth, boxHeight, 8);
      ctx.fill();
      ctx.restore();

      ctx.font = '700 14px "Roboto2", sans-serif';
      const pointState = chartPoints.value[i]?.state;
      // eslint-disable-next-line no-nested-ternary
      ctx.fillStyle = pointState === 'warning' ? '#FE9800' : (pointState === 'error' ? '#FF3333' : '#1F2024');
      const titleY = hasExtraText ? yPos + 18 : yPos + boxHeight / 2;
      ctx.fillText(title, center, titleY);

      if (label?.extra1) {
        ctx.font = '400 12px "Roboto2", sans-serif';
        ctx.fillStyle = '#79797C';
        ctx.fillText(label.extra1, center, yPos + 40);
      }

      if (label?.extra2) {
        ctx.font = '400 12px "Roboto2", sans-serif';
        ctx.fillStyle = '#79797C';
        ctx.fillText(label.extra2, center, yPos + 58);
      }
    });

    ctx.restore();
  },
};

const customBottomLabels = {
  id: 'customBottomLabels',
  afterDraw(chartInstance) {
    const { ctx, scales: { x } } = chartInstance;

    const maxLines = 1;
    ctx.save();
    ctx.textAlign = 'center';
    ctx.textBaseline = 'top';

    chartPoints.value.forEach((item, index) => {
      const layout = pointLayouts.value[index];
      if (!layout) return;
      const yPos = x.bottom + 12;
      const { left, width, center } = getLayoutPixels(x, layout);
      ctx.save();
      ctx.shadowColor = 'rgba(31, 32, 36, 0.1)';
      ctx.shadowBlur = 4;
      ctx.shadowOffsetY = 1;
      ctx.fillStyle = '#FFFFFF';

      const descriptionLines = splitTextLines(item.description);
      const bgHeight = bottomLabelBgHeight.value;
      const boxX = left;

      ctx.beginPath();
      ctx.roundRect(boxX, yPos - 8, width, bgHeight, 4);
      ctx.fill();
      ctx.restore();

      // eslint-disable-next-line no-nested-ternary
      const bottomLabelColor = item.state === 'warning' ? '#FE9800' : (item.state === 'error' ? '#FF3333' : '#1F2024');

      ctx.font = '700 14px "Roboto2", sans-serif';
      ctx.fillStyle = bottomLabelColor;
      ctx.fillText(item.xValue, center, yPos);

      ctx.font = '400 12px "Roboto2", sans-serif';
      ctx.fillStyle = bottomLabelColor;
      descriptionLines.forEach((line, lineIndex) => {
        if (lineIndex < maxLines) {
          ctx.fillText(line, center, yPos + 18 + (lineIndex * 12));
        }
      });
    });

    ctx.restore();
  },
};

const chartPlugins = [
  horizontalGuidePlugin,
  chartBorderPlugin,
  highlightPlugin,
  customTopLabels,
  customBottomLabels,
];

const scrollContainer = ref(null);
const scrollViewportWidth = ref(0);
const isDragging = ref(false);
const startX = ref(0);
const scrollLeftStart = ref(0);
const activePointerId = ref(null);
let resizeObserver;

const updateScrollViewportWidth = () => {
  if (!scrollContainer.value) return;
  const containerWidth = scrollContainer.value.getBoundingClientRect().width;
  const viewportWidth = window.visualViewport?.width || window.innerWidth;
  scrollViewportWidth.value = Math.min(containerWidth, viewportWidth);
};

const syncScrollPositionToStart = async () => {
  await nextTick();
  requestAnimationFrame(() => {
    if (!scrollContainer.value) return;
    scrollContainer.value.scrollLeft = 0;
  });
};

const startDragging = (event) => {
  if (!scrollContainer.value) return;

  const { pointerType } = event;
  if (pointerType === 'touch') {
    event.preventDefault();
  }

  activePointerId.value = event.pointerId;
  isDragging.value = true;
  startX.value = event.pageX - scrollContainer.value.offsetLeft;
  scrollLeftStart.value = scrollContainer.value.scrollLeft;
  scrollContainer.value.style.cursor = 'grabbing';
  scrollContainer.value.setPointerCapture?.(event.pointerId);
};

const doDrag = (event) => {
  if (!isDragging.value || !scrollContainer.value) return;
  if (activePointerId.value !== null && event.pointerId !== activePointerId.value) return;

  const x = event.pageX - scrollContainer.value.offsetLeft;
  scrollContainer.value.scrollLeft = scrollLeftStart.value - (x - startX.value) * 1.5;
};

const stopDragging = (event) => {
  if (activePointerId.value !== null && event?.pointerId !== undefined && event.pointerId !== activePointerId.value) return;

  if (scrollContainer.value && activePointerId.value !== null) {
    scrollContainer.value.releasePointerCapture?.(activePointerId.value);
  }

  isDragging.value = false;
  activePointerId.value = null;
  if (scrollContainer.value) {
    scrollContainer.value.style.cursor = 'grab';
  }
};

const chartWidth = computed(() => {
  const contentWidth = pointLayoutsTotalWidth.value;
  const yLabelsWidth = getYAxisLabelWidth();
  const baseWidth = contentWidth + yLabelsWidth;
  const availableWidth = scrollViewportWidth.value || baseWidth;

  return Math.max(baseWidth, availableWidth);
});

const xAxisRightPadding = computed(() => {
  const totalContentWidth = pointLayoutsTotalWidth.value + getYAxisLabelWidth();
  const viewportWidth = scrollViewportWidth.value || totalContentWidth;

  return Math.max(0, viewportWidth - totalContentWidth);
});

const chartHeight = computed(() => (
  GRAPH_HEIGHT
  + canvasTopPadding.value
  + canvasBottomPadding.value
));
const chartHeightPx = computed(() => `${chartHeight.value}px`);
const containerWidth = computed(() => `${chartWidth.value}px`);
const chartCanvasStyle = computed(() => ({
  display: 'block',
  width: `${chartWidth.value}px`,
  height: `${chartHeight.value}px`,
  minWidth: `${chartWidth.value}px`,
}));
const hasHorizontalOverflow = computed(() => chartWidth.value > scrollViewportWidth.value);

const handleWheel = (event) => {
  if (!scrollContainer.value || !hasHorizontalOverflow.value) return;

  const delta = Math.abs(event.deltaX) > Math.abs(event.deltaY)
    ? event.deltaX
    : event.deltaY;

  if (!delta) return;

  scrollContainer.value.scrollLeft += delta;
  event.preventDefault();
};

onMounted(async () => {
  updateScrollViewportWidth();
  resizeObserver = new ResizeObserver(() => {
    updateScrollViewportWidth();
  });

  if (scrollContainer.value) {
    resizeObserver.observe(scrollContainer.value);
  }

  [normalImg.value, warningImg.value, errorImg.value] = await Promise.all([
    createImg(normalSvg),
    createImg(warningSvg),
    createImg(errorSvg),
  ]);
  imagesLoaded.value = true;
  if (scrollContainer.value) {
    scrollContainer.value.style.cursor = 'grab';
  }
  await syncScrollPositionToStart();
});

onBeforeUnmount(() => {
  resizeObserver?.disconnect();
});

const getPointStyle = (state) => {
  if (state === 'error') return errorImg.value;
  if (state === 'warning') return warningImg.value;
  return normalImg.value;
};

watch(
  [scrollViewportWidth, chartWidth, imagesLoaded],
  async () => {
    if (!isDragging.value && imagesLoaded.value) {
      await syncScrollPositionToStart();
    }
  },
  { flush: 'post' },
);

const chartData = computed(() => ({
  datasets: [{
    data: chartPoints.value.map((point, index) => ({
      x: pointLayouts.value[index]?.center || 0,
      y: yValues.value[index],
    })),
    borderColor: '#1F2024',
    borderWidth: 2,
    tension: 0,
    clip: false,
    pointRadius: POINT_RADIUS,
    pointHoverRadius: POINT_RADIUS,
    spanGaps: true,
    pointStyle: chartPoints.value.map((point) => getPointStyle(point.state)),
  }],
}));

const chartOptions = computed(() => ({
  responsive: false,
  maintainAspectRatio: false,
  interaction: {
    mode: 'nearest',
    intersect: false,
  },
  onHover: (event, elements) => {
    activeIndex.value = (elements && elements.length > 0) ? elements[0].index : null;
  },
  layout: {
    padding: {
      bottom: canvasBottomPadding.value,
      top: canvasTopPadding.value,
      left: 0,
      right: xAxisRightPadding.value,
    },
  },
  plugins: {
    legend: { display: false },
    tooltip: { enabled: false },
  },
  scales: {
    y: {
      min: yAxisMin.value,
      max: yAxisMax.value,
      grid: {
        display: true,
        color: '#E5E7EB',
        lineWidth: 1.5,
        drawBorder: false,
        tickLength: 0,
      },
      ticks: {
        stepSize: yAxisStep.value,
        precision: 0,
        color: '#79797C',
        font: { size: 12 },
        callback: (value) => {
          const numericValue = Number(value);
          const relativeValue = numericValue - yAxisMin.value;

          if (relativeValue !== 0 && relativeValue % yAxisLabelStep.value !== 0) {
            return '';
          }

          return `${Math.round(numericValue)} ${yLabelSuffix.value}`;
        },
      },
    },
    x: {
      type: 'linear',
      min: 0,
      max: pointLayoutsTotalWidth.value,
      grid: {
        display: false,
      },
      ticks: { display: false },
    },
  },
}));
</script>

<style scoped lang="sass">
@mixin scroll($size: 6px, $barColor: var(--whiteSmokeCarbon), $thumbColor: var(--disabled))
  &::-webkit-scrollbar
    width: $size
    height: $size
    background-color: $barColor
    border-radius: $size

  &::-webkit-scrollbar-thumb
    background-color: $thumbColor
    border-radius: 4px

  @-moz-document url-prefix()
    scrollbar-color: $thumbColor transparent
    scrollbar-width: auto

//%scroll
//    @include scroll

@mixin scrollNotVisible
  scrollbar-width: none
  &::-webkit-scrollbar
    width: 0
    height: 0

//%scrollNotVisible
//    @include scrollNotVisible


.chart-wrapper
  display: grid

.scroll-wrapper
  display: block
  width: 100%
  overflow-x: auto
  overflow-y: hidden
  cursor: grab
  overscroll-behavior-x: contain
  user-select: none


.mileage-chart-container
  position: relative
  width: 100%
</style>
