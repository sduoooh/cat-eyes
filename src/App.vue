<template>
  <div class="cat-eyes-container">
    <video ref="videoElement" autoplay muted style="display: none;"></video>
    <canvas ref="catEyesCanvas" :width="canvasWidth" :height="canvasHeight" style="background: #222626;"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, shallowRef } from 'vue';

// 全局相关
const animationFrameId = ref(null);
const catEyesCanvas = ref(null);
const ctx = shallowRef(null);
const canvasWidth = ref(window.innerWidth);
const canvasHeight = ref(window.innerHeight);

// 静态渲染参数
const eyeHeight = computed(() => canvasHeight.value * 0.42);
const eyeWidth = computed(() => eyeHeight.value * 1.14);
const eyeSpacing = computed(() => eyeWidth.value * 0.81);
const pupilWidth = computed(() => eyeWidth.value * 0.16);
const pupilHeight = computed(() => eyeHeight.value * 0.8);
const pupilFocusDistance = computed(() => eyeSpacing.value / 2 + eyeWidth.value / 2);
const contrlPointParams = {
  upPoint1X: -0.4, upPoint1Y: -0.6,
  upPoint2X: 0.3, upPoint2Y: -0.6,
  downPoint1X: 0.3, downPoint1Y: 0.6,
  downPoint2X: -0.3, downPoint2Y: 0.6,
}
const controlPoints = computed(() => {
  return [
    { x: eyeWidth.value * contrlPointParams.upPoint1X, y: (eyeWidth.value * contrlPointParams.upPoint1X * slope.value + offset.value) + (eyeHeight.value * contrlPointParams.upPoint1Y - (eyeWidth.value * contrlPointParams.upPoint1X * slope.value + offset.value)) * eyeOpenness.value },
    { x: eyeWidth.value * contrlPointParams.upPoint2X, y: (eyeWidth.value * contrlPointParams.upPoint2X * slope.value + offset.value) + (eyeHeight.value * contrlPointParams.upPoint2Y - (eyeWidth.value * contrlPointParams.upPoint2X * slope.value + offset.value)) * eyeOpenness.value },
    { x: eyeWidth.value * contrlPointParams.downPoint1X, y: (eyeWidth.value * contrlPointParams.downPoint1X * slope.value + offset.value) + (eyeHeight.value * contrlPointParams.downPoint1Y - (eyeWidth.value * contrlPointParams.downPoint1X * slope.value + offset.value)) * eyeOpenness.value },
    { x: eyeWidth.value * contrlPointParams.downPoint2X, y: (eyeWidth.value * contrlPointParams.downPoint2X * slope.value + offset.value) + (eyeHeight.value * contrlPointParams.downPoint2Y - (eyeWidth.value * contrlPointParams.downPoint2X * slope.value + offset.value)) * eyeOpenness.value }
  ]
});


// 眨眼状态
const eyeOpenness = ref(1);
const isEyeOpen = ref(false);
const isBlinking = ref(false);
// 眨眼参数
const blinkProbability = 0.999;
const blinkSpeed = 0.05;
// 眨眼运动参数
const slope = computed(() => -0.13 * eyeHeight.value / eyeWidth.value);
const offset = computed(() => -0.065 * eyeHeight.value);


// 焦点
const initialFocusPoint = computed(() => ({
  x: canvasWidth.value / 2,
  y: canvasHeight.value / 2
}));
const finalValidFocusPoint = ref({
  x: initialFocusPoint.value.x,
  y: initialFocusPoint.value.y
});
const focusPointX = ref(finalValidFocusPoint.value.x);
const focusPointY = ref(finalValidFocusPoint.value.y);
const lastFocusPoint = ref({ x: 0, y: 0 });
// 瞳孔
const pupilOffsetLimitX = [-1.5, 1.5];
const pupilOffsetLimitY = [-0.13, 0.2];
const pupilOffsetXDefault = -0.5;
const pupilOffsetYDefault = -0.02;
const pupilOffsetDeltaX = ref(0);
const pupilOffsetDeltaY = ref(0);
const pupilOffsetXRight = computed(() => pupilOffsetXDefault + pupilOffsetDeltaX.value);
const pupilOffsetXLeft = computed(() => pupilOffsetXDefault - pupilOffsetDeltaX.value)
const pupilOffsetY = computed(() => pupilOffsetYDefault + pupilOffsetDeltaY.value);
// 追焦运动相关
const isFocusLost = ref(false);
const safeBorderSize = computed(() => canvasWidth.value * 0.01);
const eyeTraceTime = ref(0)
const eyeTraceInterval = 5000;
const eyeTraceFlag = ref(false);
const backTraceFlag = ref(false);
const eyeTracePlannedTime = 350;
const backTracePlannedTime = 1500;
const eyeTraceSpeed = computed(() => eyeWidth.value / eyeTracePlannedTime);
const backTraceSpeed = computed(() => eyeWidth.value / backTracePlannedTime);
//const testPupilRadius = ref()


// 焦点动态获取-鼠标
/* import { useMouse } from '@vueuse/core';
const { x, y } = useMouse(); */

// 焦点动态获取-手势
import { Hands as _Hands } from "@mediapipe/hands";
import { Camera as _Camera } from "@mediapipe/camera_utils";
const x = ref();
const y = ref();
const videoElement = ref(null);

// 计时器函数 类似于setInterval
const lastMark = ref({
  time: 0,
  reason: null
})
const markWork = ref(() => { })
const markInterval = 3000;


// 超限取限值，否则返回原值
const putInRange = (value, min, max) => {
  return Math.max(min, Math.min(max, value))
};

// 返回一个最接近目标点的有效路径终点
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
    markWork.value = () => { };
    isFocusLost.value = false;
  }
  return finalValidFocusPoint.value;
}

// 更新瞳孔焦点位置，生成本帧的焦点位置
const getNextFocusPoint = (targetFocusPoint) => {
  let res = { x: focusPointX.value, y: focusPointY.value };

  const xDistance = targetFocusPoint.x - focusPointX.value;
  const yDistance = targetFocusPoint.y - focusPointY.value;
  const xDistanceFactor = xDistance / pupilWidth.value;
  const yDistanceFactor = yDistance / eyeHeight.value;
  const isXOverMatch = xDistanceFactor > pupilOffsetLimitX[1] || xDistanceFactor < pupilOffsetLimitX[0];
  const isYOverMatch = yDistanceFactor > pupilOffsetLimitY[1] || yDistanceFactor < pupilOffsetLimitY[0];
  const eyeMinDistanceX = Math.min(Math.abs(xDistance), eyeTraceSpeed.value);
  const eyeMinDistanceY = Math.min(Math.abs(yDistance), eyeTraceSpeed.value);
  const backTraceMinDistanceX = Math.min(Math.abs(xDistance), backTraceSpeed.value);
  const backTraceMinDistanceY = Math.min(Math.abs(yDistance), backTraceSpeed.value);

  pupilOffsetDeltaX.value = putInRange(xDistanceFactor, pupilOffsetLimitX[0], pupilOffsetLimitX[1]);
  pupilOffsetDeltaY.value = putInRange(yDistanceFactor, pupilOffsetLimitY[0], pupilOffsetLimitY[1]);

  if (targetFocusPoint.x !== lastFocusPoint.value.x || targetFocusPoint.y !== lastFocusPoint.value.y) {
    lastFocusPoint.value = targetFocusPoint;
    eyeTraceFlag.value = false;
    eyeTraceTime.value = Date.now();
  }

  if (xDistance === 0 && yDistance === 0) {
    backTraceFlag.value = false;
    eyeTraceFlag.value = false;
    eyeTraceTime.value = 0;
    return res;
  }


  if (isXOverMatch) {
    backTraceFlag.value = true;
    eyeTraceTime.value = 0;
    res.x += eyeMinDistanceX * (xDistanceFactor > 0 ? 1 : -1);
  }
  if (isYOverMatch) {
    backTraceFlag.value = true;
    eyeTraceTime.value = 0;
    res.y += eyeMinDistanceY * (yDistanceFactor > 0 ? 1 : -1);
  }
  if (!isXOverMatch && !isYOverMatch) {
    if (eyeTraceTime.value === 0) {
      eyeTraceTime.value = Date.now();
      eyeTraceFlag.value = false;
    }
    if (Date.now() - eyeTraceTime.value >= eyeTraceInterval) {
      eyeTraceFlag.value = true;
    }
    if (eyeTraceFlag.value || backTraceFlag.value) {
      res.x += backTraceMinDistanceX * (xDistanceFactor > 0 ? 1 : -1);
      res.y += backTraceMinDistanceY * (yDistanceFactor > 0 ? 1 : -1);
    }
  }

  return res;
}

const drawCatEyes = () => {
  if (!ctx.value) return;
  ctx.value.clearRect(0, 0, canvasWidth.value, canvasHeight.value);

  const targetFocusPoint = getNextFocusPoint(generateFocusPointTarget());

  drawEye(targetFocusPoint, true);
  drawEye(targetFocusPoint, false);

  focusPointX.value = targetFocusPoint.x;
  focusPointY.value = targetFocusPoint.y;
};


const drawEye = (targetFocusPoint, isLeftEye) => {
  ctx.value.save();

  if (isLeftEye) {
    ctx.value.translate(targetFocusPoint.x - pupilFocusDistance.value, targetFocusPoint.y);
    ctx.value.scale(-1, 1); // 左眼水平翻转
  } else {
    ctx.value.translate(targetFocusPoint.x + pupilFocusDistance.value, targetFocusPoint.y);
  }

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

  // 瞳孔
  ctx.value.fillStyle = '#acb582';
  ctx.value.globalCompositeOperation = 'source-atop';
  ctx.value.beginPath();
  ctx.value.ellipse(pupilWidth.value * (isLeftEye ? pupilOffsetXLeft.value : pupilOffsetXRight.value), eyeHeight.value * pupilOffsetY.value, pupilWidth.value / 2, pupilHeight.value / 2, 0, 0, 2 * Math.PI);
  ctx.value.fill();
  ctx.value.globalCompositeOperation = 'source-over';

  ctx.value.restore();
};

const updateCanvasSize = () => {
  canvasWidth.value = window.innerWidth;
  canvasHeight.value = window.innerHeight;
}

const updateBlinkStatus = () => {
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
}

const countCount = () => {
  if (lastMark.value.time != 0 && Date.now() - lastMark.value.time >= markInterval) {
    markWork.value();
    lastMark.value = {
      time: 0,
      reason: null
    };
    markWork.value = () => { };
  };
}

const onResults = (results) => {
  if (results.multiHandLandmarks.length === 0) {
    x.value = undefined;
    y.value = undefined;
    return;
  }
  const dectedPoints = results.multiHandLandmarks[0].reduce((pre, cur) => {
    return { x: pre.x + cur.x, y: pre.y + cur.y }
  }, { x: 0.5, y: 0.5 })
  x.value = dectedPoints.x / 21;
  y.value = dectedPoints.y / 21;
  x.value = initialFocusPoint.value.x - (x.value > 0.5 ? 1 : -1) * Math.abs(x.value - 0.5) * canvasWidth.value;
  y.value = initialFocusPoint.value.y + (y.value > 0.5 ? 1 : -1) * Math.abs(y.value - 0.5) * canvasHeight.value;
}


const updateTargetByHandInit = () => {
  const Hands = _Hands || window.Hands;
  const Camera = _Camera || window.Camera;
  const hands = new Hands({
    locateFile: (file) => {
      return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
    }
  });
  hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 1,
    minDetectionConfidence: 0.7,
    minTrackingConfidence: 0.7
  });
  hands.onResults(onResults);
  hands.initialize();
  const camera = new Camera(videoElement.value, {
    onFrame: async () => {
      await hands.send({ image: videoElement.value });
    },
  });
  camera.start()
}

const drawFocusPoint = () => {
  ctx.value.fillStyle = 'rgb(255, 0, 0)';
  ctx.value.beginPath();
  ctx.value.arc(x.value, y.value, 10, 0, 2 * Math.PI);
  ctx.value.fill();
  ctx.value.restore();
}

const animate = () => {
  countCount();
  updateCanvasSize();
  updateBlinkStatus();
  drawCatEyes();
  drawFocusPoint();
  animationFrameId.value = requestAnimationFrame(animate);
}


onMounted(() => {
  if (catEyesCanvas.value)
    ctx.value = catEyesCanvas.value.getContext('2d');
  if (videoElement.value) {
    updateTargetByHandInit();
  }
  animate();
});

onUnmounted(() => {
  if (animationFrameId.value)
    cancelAnimationFrame(animationFrameId.value);
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