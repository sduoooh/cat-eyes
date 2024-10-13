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
/* const generateFocusPointTargetX = () => {
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
} */

const focusPointX = ref(finalValidFocusPoint.value.x);
const focusPointY = ref(finalValidFocusPoint.value.y);

const pupilXOffset = ref(0);
const pupilYOffset = ref(0);

const lastMark = ref({
  time: 0,
  reason: null
})
const markWork = ref(() => { })
const markInterval = 3000;

const pupilFocusDistance = computed(() => eyeSpacing.value / 2 + eyeWidth.value / 2);

const blinkProbability = 0.999;
const blinkSpeed = 0.05;

let animationFrameId = null;

const controlPoints = ref([]);

const putInRange = (value, min, max) => {
  return Math.max(min, Math.min(max, value))
};

const generateFocusPointTarget = () => {
  const widthBorder = safeBorderSize.value + eyeWidth.value + eyeSpacing.value / 2;
  const heightBorder = safeBorderSize.value + eyeHeight.value / 2;

  if (!x.value || !y.value) {
    if (finalValidFocusPoint.value.x === initialFocusPoint.value.x && finalValidFocusPoint.value.y === initialFocusPoint.value.y) return initialFocusPoint.value;
    if (lastMark.value.reason != 'overflow') {
      lastMark.value.reason = 'overflow';
      lastMark.value.time = Date.now();
      markWork.value = () => {
        finalValidFocusPoint.value.x = initialFocusPoint.value.x;
        finalValidFocusPoint.value.y = initialFocusPoint.value.y;
        isFocusLost.value = true;
      }
    }
    return finalValidFocusPoint.value;
  }
  if ((x.value - widthBorder) <= 0 || (canvasWidth.value - widthBorder - x.value) <= 0 || (y.value - heightBorder) <= 0 || (canvasHeight.value - heightBorder - y.value) <= 0) {
    if (lastMark.value.reason != 'overflow') {
      lastMark.value.reason = 'overflow';
      lastMark.value.time = Date.now();
      markWork.value = () => {
        finalValidFocusPoint.value.x = initialFocusPoint.value.x;
        finalValidFocusPoint.value.y = initialFocusPoint.value.y;
        isFocusLost.value = true;
      }
    }
    const isYMove = putInRange(x.value, widthBorder, canvasWidth.value - widthBorder) === focusPointX.value;
    const isXMove = putInRange(y.value, heightBorder, canvasHeight.value - heightBorder) === focusPointY.value;
    const isTop = y.value < focusPointY.value;
    const isRight = x.value > focusPointX.value;
    if (isYMove || isXMove || (isRight && x.value + widthBorder >= canvasWidth.value - 5) || (!isRight && x.value <= widthBorder + 5)) {
      finalValidFocusPoint.value.y = putInRange(y.value, heightBorder, canvasHeight.value - heightBorder);
      finalValidFocusPoint.value.x = putInRange(x.value, widthBorder, canvasWidth.value - widthBorder);
      return finalValidFocusPoint.value;
    }
    const traceSlope = Math.abs((y.value - focusPointY.value) / (x.value - focusPointX.value));
    const safeSlope = Math.abs((isTop ? heightBorder - focusPointY.value : canvasHeight.value - heightBorder - focusPointY.value) / (isRight ? canvasWidth.value - widthBorder - focusPointX.value : widthBorder - focusPointX.value));
    if (traceSlope === safeSlope) {
      finalValidFocusPoint.value.x = putInRange(x.value, widthBorder, canvasWidth.value - widthBorder);
      finalValidFocusPoint.value.y = putInRange(y.value, heightBorder, canvasHeight.value - heightBorder);
    } else if (isTop) {
      if (traceSlope > safeSlope) {
        finalValidFocusPoint.value.y = heightBorder;
        finalValidFocusPoint.value.x = focusPointX.value + (focusPointY.value - finalValidFocusPoint.value.y) * (isRight ? 1 : -1) / traceSlope;
      } else {
        finalValidFocusPoint.value.x = isRight ? canvasWidth.value - widthBorder : widthBorder;
        finalValidFocusPoint.value.y = focusPointY.value - (finalValidFocusPoint.value.x - focusPointX.value) * (isRight ? 1 : -1) * traceSlope;
      }
    } else {
      if (traceSlope > safeSlope) {
        finalValidFocusPoint.value.y = canvasHeight.value - heightBorder;
        finalValidFocusPoint.value.x = focusPointX.value + (finalValidFocusPoint.value.y - focusPointY.value) * (isRight ? 1 : -1) / traceSlope;
      } else {
        finalValidFocusPoint.value.x = isRight ? canvasWidth.value - widthBorder : widthBorder;
        finalValidFocusPoint.value.y = focusPointY.value + (finalValidFocusPoint.value.x - focusPointX.value) * (isRight ? 1 : -1) * traceSlope;
      }
    }
    return finalValidFocusPoint.value;
  }
  finalValidFocusPoint.value.x = x.value;
  finalValidFocusPoint.value.y = y.value;
  if (lastMark.value.reason === 'overflow') {
    lastMark.value = {
      time: 0,
      reason: null
    };
    isFocusLost.value = false;
  }
  return finalValidFocusPoint.value;
}

const pupilWidth = computed(() => eyeWidth.value * 0.16);
const pupilHeight = computed(() => eyeHeight.value * 0.8);
const pupilOffsetLimitX = [0.9, -0.9];
const pupilOffsetLimitY = [0.08, -0.08];
const pupilOffsetXDefault = -0.5;
const pupilOffsetYDefault = -0.02;
const pupilOffsetX = ref(pupilOffsetXDefault);
const pupilOffsetY = ref(pupilOffsetYDefault);

const eyeTraceCount = ref(0)
const eyeTraceInterval = 8000;
const lastFocusPoint = ref({ x: 0, y: 0 });
const getNextFocusPoint = (targetFocusPoint) => {
  let res = { x: targetFocusPoint.x, y: targetFocusPoint.y };

  const xDistance = targetFocusPoint.x - focusPointX.value;
  const yDistance = targetFocusPoint.y - focusPointY.value;
  const xDistanceFactor = xDistance / pupilWidth.value;
  const yDistanceFactor = yDistance / eyeHeight.value;
  const isPupilMatchX = putInRange(xDistanceFactor, pupilOffsetLimitX[0], pupilOffsetLimitX[1]) === pupilOffsetX.value;
  const isPupilMatchY = putInRange(yDistanceFactor, pupilOffsetLimitY[0], pupilOffsetLimitY[1]) === pupilOffsetY.value;
  const isXOverMatch = xDistanceFactor > pupilOffsetLimitX[1] || xDistanceFactor < pupilOffsetLimitX[0];
  const isYOverMatch = yDistanceFactor > pupilOffsetLimitY[1] || yDistanceFactor < pupilOffsetLimitY[0];

  console.log({
    isPupilMatchX,
    isPupilMatchY,
    isXOverMatch,
    isYOverMatch
  });

  if (targetFocusPoint.x !== lastFocusPoint.value.x || targetFocusPoint.y !== lastFocusPoint.value.y) {
    lastFocusPoint.value = targetFocusPoint;
    eyeTraceCount.value = 0;
  }

  if (xDistance === 0 && yDistance === 0) {
    pupilOffsetX.value = pupilOffsetXDefault;
    pupilOffsetY.value = pupilOffsetYDefault;
    return res;
  }

  if (isPupilMatchX && isPupilMatchY) {
    if (isXOverMatch) {
      res.x = focusPointX.value + xDistance//(xDistance > 0 ? 1 : -1) * Math.max(Math.abs(xDistance), eyeWidth.value / (20 * pupilTraceTime));
    }
    if (isYOverMatch) {
      res.y = focusPointY.value + yDistance//(yDistance > 0 ? 1 : -1) * Math.max(Math.abs(yDistance), eyeHeight.value / (20 * pupilTraceTime));
    }
    if (!isXOverMatch && !isYOverMatch) {
      if (eyeTraceCount.value >= eyeTraceInterval) {
        eyeTraceCount.value = 0;
        const addX = xDistance//(xDistance > 0 ? 1 : -1) * Math.max(Math.abs(xDistance), eyeWidth.value / (20 * pupilTraceTime));
        const addY = yDistance//(yDistance > 0 ? 1 : -1) * Math.max(Math.abs(yDistance), eyeHeight.value / (20 * pupilTraceTime));
        res.x = focusPointX.value + addX;
        res.y = focusPointY.value + addY;
        pupilOffsetX.value -= addX / pupilWidth.value;
        pupilOffsetY.value -= addY / eyeHeight.value;
      } else {
        eyeTraceCount.value++;
      }
    }
  } else {
    if (!isPupilMatchX) {
      pupilOffsetX.value += (xDistance > 0 ? 1 : -1) * 0.005;
    }
    if (!isPupilMatchY) {
      pupilOffsetY.value += (yDistance > 0 ? 1 : -1) * 0.0005;
    }
  }
  return res;
}

const drawCatEyes = () => {
  if (!ctx.value) return;

  if (lastMark.value.time != 0 && Date.now() - lastMark.value.time >= markInterval) {
    markWork.value();
    lastMark.value = {
      time: 0,
      reason: null
    };
  };
  // 清空画布
  ctx.value.clearRect(0, 0, canvasWidth.value, canvasHeight.value);
  //const targetFocusPoint = generateFocusPointTarget();
  const targetFocusPoint = getNextFocusPoint(generateFocusPointTarget());
  focusPointX.value = targetFocusPoint.x;
  focusPointY.value = targetFocusPoint.y;

  drawEye(targetFocusPoint, true);
  drawEye(targetFocusPoint, false);
};

const drawEye = (targetFocusPoint, isLeftEye) => {
  ctx.value.save();
  if (isLeftEye) {
    ctx.value.translate(targetFocusPoint.x - pupilFocusDistance.value, targetFocusPoint.y);
    ctx.value.scale(-1, 1); // 左眼水平翻转
  } else {
    ctx.value.translate(targetFocusPoint.x + pupilFocusDistance.value, targetFocusPoint.y);
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

  ctx.value.fillStyle = '#acb582';
  ctx.value.globalCompositeOperation = 'source-atop';
  ctx.value.beginPath();
  ctx.value.ellipse(pupilWidth.value * pupilOffsetX.value, eyeHeight.value * pupilOffsetY.value, pupilWidth.value / 2, pupilHeight.value / 2, 0, 0, 2 * Math.PI);
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