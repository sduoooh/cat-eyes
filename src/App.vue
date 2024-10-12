<template>
  <div class="cat-eyes-container">
    <canvas ref="catEyesCanvas" :width="canvasWidth" :height="canvasHeight" style="background: #222626;"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const catEyesCanvas = ref(null);
const canvasWidth = ref(window.innerWidth);
const canvasHeight = ref(window.innerHeight);
const eyeOpenness = ref(1);
const isEyeOpen = ref(false);
const isBlinking = ref(false);

const blinkProbability = 0.999;
const blinkSpeed = 0.05;

let animationFrameId = null;

const getCloseEyeXSlope = (width, height) => -0.13 * height / width;
const getCloseEyeXOffet = (height) => -0.065 * height;

const drawCatEyes = () => {
  const canvas = catEyesCanvas.value;
  const ctx = canvas.getContext('2d');

  // 清空画布
  ctx.clearRect(0, 0, canvasWidth.value, canvasHeight.value);

  // 定义眼睛参数
  const eyeHeight = canvasHeight.value * 0.42;
  const eyeWidth = eyeHeight * 1.14;
  const eyeSpacing = eyeWidth * 0.81;

  // 左眼位置
  const leftEyeX = canvasWidth.value / 2 - eyeSpacing / 2 - eyeWidth / 2;
  const leftEyeY = canvasHeight.value / 2;

  // 右眼位置
  const rightEyeX = canvasWidth.value / 2 + eyeSpacing / 2 + eyeWidth / 2;
  const rightEyeY = canvasHeight.value / 2;

  const slope = getCloseEyeXSlope(eyeWidth, eyeHeight);
  const offset = getCloseEyeXOffet(eyeHeight);

  // 绘制眼睛
  drawEye(ctx, leftEyeX, leftEyeY, eyeWidth, eyeHeight, true, slope, offset);
  drawEye(ctx, rightEyeX, rightEyeY, eyeWidth, eyeHeight, false, slope, offset);
};

const controlPoints = ref([]);

const drawEye = (ctx, centerX, centerY, width, height, isLeftEye, slope, offset) => {
  ctx.save();
  ctx.translate(centerX, centerY);
  if (isLeftEye) {
    ctx.scale(-1, 1); // 右眼水平翻转
  }

  const srcControlPoints = [
    { x: width * -0.4, y: height * -0.6 },
    { x: width * 0.3, y: height * -0.6 },
    { x: width * 0.3, y: height * 0.6 },
    { x: width * -0.3, y: height * 0.6 }
  ];

  controlPoints.value = srcControlPoints.map(
    p => {
      let closeY = p.x * slope + offset;
      return {
        x: p.x,
        y: closeY + (p.y - closeY) * eyeOpenness.value,
        closeY
      }
    }
  )

  // 眼白
  ctx.fillStyle = '#fcfeee';
  ctx.beginPath();
  ctx.moveTo(width * -0.5, 0);

  ctx.bezierCurveTo(
    controlPoints.value[0].x, controlPoints.value[0].y,
    controlPoints.value[1].x, controlPoints.value[1].y,
    width * 0.5, height * -0.14
  );
  ctx.bezierCurveTo(
    controlPoints.value[2].x, controlPoints.value[2].y,
    controlPoints.value[3].x, controlPoints.value[3].y,
    width * -0.5, 0
  );

  ctx.closePath();
  ctx.fill();

  // 瞳孔（竖瞳）
  const pupilWidth = width * 0.16;
  const pupilHeight = height * 0.8;
  const pupilX = -pupilWidth / 2;
  const pupilY = -height * 0.02;
  ctx.fillStyle = '#acb582';
  ctx.globalCompositeOperation = 'source-atop';
  ctx.beginPath();
  ctx.ellipse(pupilX, pupilY, pupilWidth / 2, pupilHeight / 2, 0, 0, 2 * Math.PI);
  ctx.fill();
  ctx.globalCompositeOperation = 'source-over';


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

  ctx.restore();
};

const animate = () => {
  canvasWidth.value = window.innerWidth;
  canvasHeight.value = window.innerHeight;
  if (!isBlinking.value) {
    isBlinking.value = Math.random() >= blinkProbability;
    eyeOpenness.value = 1;
    if (isBlinking.value) eyeOpenness.value = 1-blinkSpeed;
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