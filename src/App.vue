<template>
  <div class="cat-eyes-container">
    <canvas ref="catEyesCanvas" :width="canvasWidth" :height="canvasHeight" style="background: #222626;"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, shallowRef } from 'vue';
import { useMouse } from '@vueuse/core';

const catEyesCanvas = ref(null);
const ctx = shallowRef(null);

const canvasWidth = ref(window.innerWidth);
const canvasHeight = ref(window.innerHeight);
const eyeOpenness = ref(1);
const isEyeOpen = ref(false);
const isBlinking = ref(false);

const safeBorderSize = computed(() => canvasWidth.value * 0.01);

const eyeHeight = computed(() => canvasHeight.value * 0.42);
const eyeWidth = computed(() => eyeHeight.value * 1.14);
const eyeSpacing = computed(() => eyeWidth.value * 0.81);
const slope = computed(() => -0.13 * eyeHeight.value / eyeWidth.value);
const offset = computed(() => -0.065 * eyeHeight.value);

const { x, y } = useMouse({ touch: false });

const initialFocusPoint = computed(() => ({
  x: canvasWidth.value / 2,
  y: canvasHeight.value / 2
}));

const finalValidFocusPoint = ref({
  x: initialFocusPoint.value.x,
  y: initialFocusPoint.value.y
});

const isFocusLost = ref(false);
const generateFocusPointTargetX = () => {
  const border = safeBorderSize.value + eyeWidth.value + eyeSpacing.value / 2;
  if (!x.value) {
    if (finalValidFocusPoint.value.x === initialFocusPoint.value.x) return initialFocusPoint.value.x;
    if (isFocusLost.value) return finalValidFocusPoint.value.x;
    if (lastMark.value.reason != 'overflow-x' && lastMark.value.reason != 'overflow-y' && lastMark.value.reason != 'overflow') {
      lastMark.value.reason = 'overflow';
      lastMark.value.time = Date.now();
      markWork.value = () => {
        finalValidFocusPoint.value.x = initialFocusPoint.value.x;
        finalValidFocusPoint.value.y = initialFocusPoint.value.y;
        isFocusLost.value = true;
      }
    }
    return finalValidFocusPoint.value.x;
  }
  if ((border - x.value) * (x.value + border - canvasWidth.value) <= 0) {
    if (isFocusLost.value) return finalValidFocusPoint.value.x;
    if (lastMark.value.reason != 'overflow-x' && lastMark.value.reason != 'overflow-y' && lastMark.value.reason != 'overflow') {
      lastMark.value.reason = 'overflow-x';
      lastMark.value.time = Date.now();
      markWork.value = () => {
        finalValidFocusPoint.value.x = initialFocusPoint.value.x;
        finalValidFocusPoint.value.y = initialFocusPoint.value.y;
        isFocusLost.value = true;
      }
    }
    return finalValidFocusPoint.value.x;
  }
  if (lastMark.value.reason === 'overflow-x' || lastMark.value.reason === 'overflow') {
    lastMark.value = {
      time: 0,
      reason: null
    }
    isFocusLost.value = false;
  }
  finalValidFocusPoint.value.x = x.value;
  return x.value;
}
const generateFocusPointTargetY = () => {
  const border = safeBorderSize.value + eyeHeight.value / 2;
  if (!y.value) {
    if (finalValidFocusPoint.value.y === initialFocusPoint.value.y) return initialFocusPoint.value.y;
    if (isFocusLost.value) return finalValidFocusPoint.value.y;
    if (lastMark.value.reason != 'overflow-x' && lastMark.value.reason != 'overflow-y' && lastMark.value.reason != 'overflow') {
      lastMark.value.reason = 'overflow';
      lastMark.value.time = Date.now();
      markWork.value = () => {
        finalValidFocusPoint.value.x = initialFocusPoint.value.x;
        finalValidFocusPoint.value.y = initialFocusPoint.value.y;
        isFocusLost.value = true;
      }
    }
    return finalValidFocusPoint.value.y;
  }
  if ((border - y.value) * (y.value + border - canvasHeight.value) <= 0) {
    if (isFocusLost.value) return finalValidFocusPoint.value.y;
    if (lastMark.value.reason != 'overflow-x' && lastMark.value.reason != 'overflow-y' && lastMark.value.reason != 'overflow') {
      lastMark.value.reason = 'overflow-y';
      lastMark.value.time = Date.now();
      markWork.value = () => {
        finalValidFocusPoint.value.x = initialFocusPoint.value.x;
        finalValidFocusPoint.value.y = initialFocusPoint.value.y;
        isFocusLost.value = true;
      }
    }
    return finalValidFocusPoint.value.y
  }
  if (lastMark.value.reason === 'overflow-y' || lastMark.value.reason === 'overflow') {
    lastMark.value = {
      time: 0,
      reason: null
    }
    isFocusLost.value = false;
  }
  finalValidFocusPoint.value.y = y.value;
  return y.value;
}

/* // 目标焦点位置
const focusPointTargetX = computed(generateFocusPointTargetX);
const focusPointTargetY = computed(generateFocusPointTargetY);

// 当前焦点位置
const focusPointX = ref(initialFocusPoint.value.x);
const focusPointY = ref(initialFocusPoint.value.y); */

// 测试
const focusPointX = computed(() => x.value || finalValidFocusPoint.value.x);
const focusPointY = computed(() => y.value || finalValidFocusPoint.value.y);

const lastMark = ref({
  time: 0,
  reason: null
})
const markWork = ref(() => { })
const markInterval = 3000;

const pupilFocusDistance = computed(() => eyeSpacing.value / 2 + eyeWidth.value / 2);
const leftEyeX = computed(() => focusPointX.value - pupilFocusDistance.value);
const rightEyeX = computed(() => focusPointX.value + pupilFocusDistance.value);

const blinkProbability = 0.999;
const blinkSpeed = 0.05;

let animationFrameId = null;

const drawCatEyes = () => {
  if (!ctx.value) return;

  if (lastMark.value.time != 0 &&  Date.now() - lastMark.value.time >= markInterval) {
    markWork.value();
    lastMark.value = {
      time: 0,
      reason: null
    };
  };
  // 清空画布
  ctx.value.clearRect(0, 0, canvasWidth.value, canvasHeight.value);

  // 绘制眼睛
  drawEye(true);
  drawEye(false);
};

const controlPoints = ref([]);

const drawEye = (isLeftEye) => {
  ctx.value.save();
  if (isLeftEye) {
    ctx.value.translate(leftEyeX.value, focusPointY.value);
    ctx.value.scale(-1, 1); // 右眼水平翻转
  } else {
    ctx.value.translate(rightEyeX.value, focusPointY.value);
  }

  const srcControlPoints = [
    { x: eyeWidth.value * -0.4, y: eyeHeight.value * -0.6 },
    { x: eyeWidth.value * 0.3, y: eyeHeight.value * -0.6 },
    { x: eyeWidth.value * 0.3, y: eyeHeight.value * 0.6 },
    { x: eyeWidth.value * -0.3, y: eyeHeight.value * 0.6 }
  ];

  controlPoints.value = srcControlPoints.map(
    p => {
      let closeY = p.x * slope.value + offset.value;
      return {
        x: p.x,
        y: closeY + (p.y - closeY) * eyeOpenness.value,
        closeY
      }
    }
  )

  // 眼白
  ctx.value.fillStyle = '#fcfeee';
  ctx.value.beginPath();
  ctx.value.moveTo(eyeWidth.value * -0.5, 0);

  ctx.value.bezierCurveTo(
    controlPoints.value[0].x, controlPoints.value[0].y,
    controlPoints.value[1].x, controlPoints.value[1].y,
    eyeWidth.value * 0.5, eyeHeight.value * -0.14
  );
  ctx.value.bezierCurveTo(
    controlPoints.value[2].x, controlPoints.value[2].y,
    controlPoints.value[3].x, controlPoints.value[3].y,
    eyeWidth.value * -0.5, 0
  );

  ctx.value.closePath();
  ctx.value.fill();

  // 瞳孔（竖瞳）
  const pupilWidth = eyeWidth.value * 0.16;
  const pupilHeight = eyeHeight.value * 0.8;
  const pupilX = -pupilWidth / 2;
  const pupilY = -eyeHeight.value * 0.02;
  ctx.value.fillStyle = '#acb582';
  ctx.value.globalCompositeOperation = 'source-atop';
  ctx.value.beginPath();
  ctx.value.ellipse(pupilX, pupilY, pupilWidth / 2, pupilHeight / 2, 0, 0, 2 * Math.PI);
  ctx.value.fill();
  ctx.value.globalCompositeOperation = 'source-over';


  /*   ctx.fillStyle = 'red'; // 使用红色使控制点更加明显
    ctx.beginPath();
    controlPoints.value.forEach(point => {
      ctx.moveTo(point.x, point.y);
      ctx.arc(point.x, point.y, 4, 0, 2 * Math.PI);
    });
    ctx.fill();
  
    ctx.fillStyle = 'blue'; // 使用红色使控制点更加明显
    ctx.beginPath();
    srcControlPoints.forEach(point => {
      ctx.moveTo(point.x, point.y);
      ctx.arc(point.x, point.y, 4, 0, 2 * Math.PI);
    });
    ctx.fill(); */

  ctx.value.restore();
};

const animate = () => {
  canvasWidth.value = window.innerWidth;
  canvasHeight.value = window.innerHeight;
  if (!isBlinking.value) {
    isBlinking.value = Math.random() >= blinkProbability;
    eyeOpenness.value = 1;
    if (isBlinking.value) eyeOpenness.value = 1 - blinkSpeed;
  } else {
    if (eyeOpenness.value === 1) {
      isBlinking.value = isEyeOpen.value;
    }
    if (eyeOpenness.value > 1) {
      isEyeOpen.value = false;
    }
    if (eyeOpenness.value <= 0) {
      isEyeOpen.value = true;
    }
    eyeOpenness.value += (isEyeOpen.value ? blinkSpeed : -blinkSpeed);
  }
  drawCatEyes();
  animationFrameId = requestAnimationFrame(animate);
}

onMounted(() => {
  if (catEyesCanvas.value) ctx.value = catEyesCanvas.value.getContext('2d');
  animate();
});

onUnmounted(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});
</script>

<style scoped>
.cat-eyes-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #f0f0f0;
}
</style>