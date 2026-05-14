<template>
  <div class="birthday-container">
    <!-- 给蛋糕容器加渐变消失类 -->
    <div
      class="cake-wrapper"
      v-if="!showLottery"
      :class="{ fadeOut: candleOut }"
    >
      <canvas ref="cakeCanvas" class="cake-canvas"></canvas>
      <canvas ref="particleCanvas" class="particle-canvas"></canvas>
      <canvas ref="textCanvas" class="text-canvas"></canvas>
    </div>

    <button class="blow-btn" @click="handleClick" v-if="btnText">
      {{ btnText }}
    </button>

    <div class="blessing-box" v-if="showBlessing">
      <!-- 顶部短句 依次掉落 -->
      <div
        class="blessing-item"
        v-for="(item, index) in blessings"
        :key="index"
        :style="item.style"
      >
        {{ item.text }}
      </div>

      <!-- 底部书信 + 打字效果 -->
      <div class="letter-box" v-if="showLetter">
        <div class="letter-content">
          {{ typedText }}
        </div>
      </div>
    </div>

    <div class="lottery" v-if="showLottery">
      <button
        class="blow-btn"
        @click="startLottery"
        v-if="showBlessing && !showCards"
      >
        下一步
      </button>

      <div class="lottery-wrap" v-if="showCards">
        <!-- 剩余次数 -->
        <div class="lottery-count">剩余抽奖次数：{{ remainingTimes }}</div>

        <!-- 8个明牌卡牌 整齐方形布局 -->
        <div class="card-list" v-if="cards">
          <div
            class="card"
            :class="{
              active: currentIndex === 0,
              winner: winIndex === 0,
              disabled: usedIndexes.includes(0) && winIndex !== 0,
            }"
          >
            {{ lotteryItems[0] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 1,
              winner: winIndex === 1,
              disabled: usedIndexes.includes(1) && winIndex !== 1,
            }"
          >
            {{ lotteryItems[1] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 2,
              winner: winIndex === 2,
              disabled: usedIndexes.includes(2) && winIndex !== 2,
            }"
          >
            {{ lotteryItems[2] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 3,
              winner: winIndex === 3,
              disabled: usedIndexes.includes(3) && winIndex !== 3,
            }"
          >
            {{ lotteryItems[3] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 4,
              winner: winIndex === 4,
              disabled: usedIndexes.includes(4) && winIndex !== 4,
            }"
          >
            {{ lotteryItems[4] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 5,
              winner: winIndex === 5,
              disabled: usedIndexes.includes(5) && winIndex !== 5,
            }"
          >
            {{ lotteryItems[5] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 6,
              winner: winIndex === 6,
              disabled: usedIndexes.includes(6) && winIndex !== 6,
            }"
          >
            {{ lotteryItems[6] }}
          </div>
          <div
            class="card"
            :class="{
              active: currentIndex === 7,
              winner: winIndex === 7,
              disabled: usedIndexes.includes(7) && winIndex !== 7,
            }"
          >
            {{ lotteryItems[7] }}
          </div>
        </div>

        <!-- 最终中奖结果展示 方便截图 -->
        <div class="final-result" v-if="showResult">
          <div class="result-card">
            <div class="result-title">🎉 好事成双 🎉</div>
            <div class="result-prize">{{ prizeList[0] }}</div>
            <div class="result-prize">{{ prizeList[1] }}</div>
            <div class="result-tip">截图发给 小的 领取</div>
          </div>
        </div>

        <!-- 中间开始按钮 -->
        <button
          class="start-btn"
          v-if="!showResult"
          @click="runLottery"
          :disabled="isRunning || remainingTimes <= 0"
        >
          {{
            isRunning
              ? "抽奖中..."
              : remainingTimes <= 0
                ? "展示奖励"
                : "点击开始"
          }}
        </button>

        <!-- 两次抽完：展示奖励按钮 -->
        <button
          class="start-btn"
          v-if="remainingTimes <= 0 && !showResult"
          @click="
            showResult = true;
            cards = false;
          "
        >
          展示奖励
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader.js";
import screenfull from "screenfull";

// ======================================
// 🔥 移动端全局开关：自动优化性能
// ======================================
const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
const MOBILE_PERF = isMobile; // 手机自动开性能模式

const candleOut = ref(false);
const btnText = ref("吹蜡烛");
const showBlessing = ref(false);
const showLottery = ref(false);
const showCards = ref(false);

const remainingTimes = ref(2);
const isRunning = ref(false);
const currentIndex = ref(-1);
const winIndex = ref(-1);
let rotateTimer = null;
const totalCards = 8;
const usedIndexes = ref([]);

const showResult = ref(false);
const prizeList = ref([]);
const cards = ref(true);

const lotteryItems = ref([
  "奶茶一杯",
  "谢谢惠顾",
  "三好学生 奖状一张",
  "肥皂一块",
  "酱油一瓶",
  "岁数红包一个",
  "金佛山门票一张",
  "洗衣粉一包",
]);

const cakeCanvas = ref(null);
const particleCanvas = ref(null);
const textCanvas = ref(null);
let scene, camera, renderer, controls, cakeModel;
let particleAnimationId = null;
let textAnimationId = null;

const rawBlessings = [
  "生日快乐！天天开心！",
  "岁岁常欢愉，年年皆胜意",
  "平安喜乐，万事顺遂",
  "新的一岁，闪闪发光",
  "永远被空气包围",
  "把把儿都吃鸡",
  "日子滚烫，快乐至上",
];

const longBlessing =
  "虽然认识得有点草率，但这句生日快乐还是说声，愿你新的一岁，平安喜乐，万事顺意，所求皆如愿，所行皆坦途。";

const blessings = ref([]);
const typedText = ref("");
const showLetter = ref(false);

const generateSafeBlessings = () => {
  blessings.value = rawBlessings.map((text, index) => ({
    text,
    style: {
      top: `${10 + index * 8}%`,
      left: "50%",
      transform: "translateX(-50%)",
      fontSize: MOBILE_PERF ? "16px" : "18px",
      color: `hsl(${Math.random() * 360}, 100%, 75%)`,
      animationDelay: `${index * 0.2}s`,
      whiteSpace: "nowrap",
    },
  }));
};

const startTypeWriter = () => {
  showLetter.value = true;
  let i = 0;
  const speed = MOBILE_PERF ? 80 : 60;
  const timer = setInterval(() => {
    if (i < longBlessing.length) {
      typedText.value += longBlessing[i];
      i++;
    } else clearInterval(timer);
  }, speed);
};

// ======================================
// 按钮逻辑（优化：切换阶段自动销毁无用canvas）
// ======================================
const handleClick = () => {
  if (btnText.value === "吹蜡烛") {
    candleOut.value = true;
    cakeCanvas.value.classList.add("fade-out");
    startTextAnimation();
    btnText.value = "查看祝福";
    if (screenfull.isEnabled) screenfull.request();
  }
  else if (btnText.value === "查看祝福") {
    // 🔥 关键：销毁所有粒子/蛋糕，只保留UI，手机瞬间不卡
    stopTextAnimation();
    destroyParticles();
    destroyCake();

    showLottery.value = true;
    generateSafeBlessings();
    showBlessing.value = true;
    btnText.value = "";
    startTypeWriter();
  }
};

const startLottery = () => {
  showBlessing.value = false;
  showCards.value = true;
};

const runLottery = () => {
  if (isRunning.value || remainingTimes.value <= 0) return;
  const availableCards = Array.from({ length: totalCards }, (_, i) => i).filter(
    (i) => !usedIndexes.value.includes(i)
  );
  if (availableCards.length === 0) {
    alert("所有奖品已抽完！");
    isRunning.value = false;
    return;
  }

  isRunning.value = true;
  winIndex.value = -1;
  currentIndex.value = -1;
  const minSpeed = 40;
  const maxSpeed = MOBILE_PERF ? 200 : 320;
  const stayTime = 1000;
  const baseRound = 6;
  const randomExtra = Math.floor(Math.random() * 8);
  const totalSteps = baseRound * totalCards + randomExtra;
  const finalWinIndex = availableCards[Math.floor(Math.random() * availableCards.length)];

  let step = 0;
  const rotate = () => {
    currentIndex.value = (currentIndex.value + 1) % totalCards;
    step++;
    if (step === totalSteps) currentIndex.value = finalWinIndex;
    const progress = Math.min(step / totalSteps, 1);
    const curSpeed = minSpeed + (maxSpeed - minSpeed) * (progress ** 2);

    if (step >= totalSteps) {
      setTimeout(() => {
        winIndex.value = finalWinIndex;
        usedIndexes.value.push(finalWinIndex);
        prizeList.value.push(lotteryItems.value[finalWinIndex]);
        isRunning.value = false;
        remainingTimes.value -= 1;
      }, stayTime);
      return;
    }
    rotateTimer = setTimeout(rotate, curSpeed);
  };

  clearTimeout(rotateTimer);
  rotate();
};

// ======================================
// 文字粒子动画（手机：大幅减少粒子）
// ======================================
const startTextAnimation = () => {
  const canvas = textCanvas.value;
  if (!canvas) return;
  const ctx = canvas.getContext("2d");
  const FIX_W = 1000;
  const FIX_H = 600;
  canvas.width = FIX_W;
  canvas.height = FIX_H;

  let text, BASE_WIDTH, BASE_HEIGHT;
  if (isMobile) {
    text = ["孙文纹", "生日快乐"];
    BASE_WIDTH = 600;
    BASE_HEIGHT = 240;
  } else {
    text = "孙文纹  生日快乐";
    BASE_WIDTH = 600;
    BASE_HEIGHT = 150;
  }

  const particles = [];
  const tempCanvas = document.createElement("canvas");
  const tempCtx = tempCanvas.getContext("2d");
  tempCanvas.width = BASE_WIDTH;
  tempCanvas.height = BASE_HEIGHT;
  tempCtx.fillStyle = "#fff";

  if (isMobile) {
    tempCtx.font = "bold 100px Microsoft YaHei";
    tempCtx.textAlign = "center";
    tempCtx.textBaseline = "middle";
    tempCtx.fillText(text[0], BASE_WIDTH / 2, BASE_HEIGHT / 2 - 60);
    tempCtx.fillText(text[1], BASE_WIDTH / 2, BASE_HEIGHT / 2 + 60);
  } else {
    tempCtx.font = "bold 70px Microsoft YaHei";
    tempCtx.textAlign = "center";
    tempCtx.textBaseline = "middle";
    tempCtx.fillText(text, BASE_WIDTH / 2, BASE_HEIGHT / 2 - 20);
  }

  const imageData = tempCtx.getImageData(0, 0, BASE_WIDTH, BASE_HEIGHT);
  const data = imageData.data;

  // 🔥 手机：采样间隔变大，粒子数量减少60%
  const step = MOBILE_PERF ? 5 : 3;

  for (let y = 0; y < BASE_HEIGHT; y += step) {
    for (let x = 0; x < BASE_WIDTH; x += step) {
      const idx = (y * BASE_WIDTH + x) * 4;
      if (data[idx + 3] > 128) {
        particles.push({
          x: Math.random() * FIX_W,
          y: Math.random() * FIX_H,
          tx: (FIX_W - BASE_WIDTH) / 2 + x,
          ty: (FIX_H - BASE_HEIGHT) / 2 + y,
          xRate: x / BASE_WIDTH,
          size: Math.random() * 0.8 + 1.0,
          speed: Math.random() * 2 + 1.8,
        });
      }
    }
  }

  const animate = () => {
    ctx.clearRect(0, 0, FIX_W, FIX_H);
    particles.forEach((p) => {
      p.x += (p.tx - p.x) / p.speed;
      p.y += (p.ty - p.y) / p.speed;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
      ctx.fillStyle = getGradientColor(p.xRate);
      ctx.fill();
    });
    textAnimationId = requestAnimationFrame(animate);
  };
  animate();
};

const getGradientColor = (rate) => {
  if (rate < 0.16) return "#ff88bb";
  if (rate < 0.32) return "#ffaa77";
  if (rate < 0.48) return "#ffdd55";
  if (rate < 0.64) return "#88ee99";
  if (rate < 0.8) return "#66ccff";
  return "#cc88ff";
};

const stopTextAnimation = () => {
  if (textAnimationId) cancelAnimationFrame(textAnimationId);
  textAnimationId = null;
  if (textCanvas.value) {
    const ctx = textCanvas.value.getContext("2d");
    ctx?.clearRect(0, 0, textCanvas.value.width, textCanvas.value.height);
  }
};

// ======================================
// 礼花粒子（手机：减少爆炸数量）
// ======================================
const initParticles = () => {
  const canvas = particleCanvas.value;
  if (!canvas) return;
  const ctx = canvas.getContext("2d");

  const resizeCanvas = () => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
  };
  resizeCanvas();
  window.addEventListener("resize", resizeCanvas);

  const rockets = [];
  const explosions = [];
  const colors = ["#ff3d3d","#ff9c3d","#ffe03d","#3dff83","#3db9ff","#9c3dff","#ff3de8","#ffffff","#ff5e88","#ffdf00","#7fff3d"];

  class Rocket {
    constructor() {
      this.x = Math.random() * canvas.width;
      this.y = canvas.height;
      this.speed = Math.random() * 2 + 2;
      this.targetY = Math.random() * (canvas.height * 0.4) + canvas.height * 0.3;
      this.color = colors[Math.floor(Math.random() * colors.length)];
      this.size = 3;
    }
    update() { this.y -= this.speed; }
    draw() {
      ctx.beginPath();
      ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
      ctx.fillStyle = this.color;
      ctx.globalAlpha = 1;
      ctx.fill();
    }
  }

  class Particle {
    constructor(x, y, color) {
      this.x = x; this.y = y;
      this.vx = (Math.random() - 0.5) * 4.5;
      this.vy = (Math.random() - 0.5) * 4.5;
      this.color = color;
      this.alpha = 1;
      this.decay = 0.004;
      this.gravity = 0.012;
    }
    update() {
      this.x += this.vx; this.y += this.vy;
      this.vy += this.gravity;
      this.alpha -= this.decay;
    }
    draw() {
      ctx.globalAlpha = Math.max(this.alpha, 0);
      ctx.fillStyle = this.color;
      ctx.beginPath();
      ctx.arc(this.x, this.y, 2.2, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  let lastLaunch = 0;
  const animate = () => {
    ctx.fillStyle = "rgba(0,0,0,0.15)";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.globalAlpha = 1;

    const now = Date.now();
    if (now - lastLaunch > (MOBILE_PERF ? 3500 : 2500) + Math.random() * 1000) {
      rockets.push(new Rocket());
      lastLaunch = now;
    }

    for (let i = rockets.length - 1; i >= 0; i--) {
      const r = rockets[i];
      r.update(); r.draw();
      if (r.y <= r.targetY) {
        // 🔥 手机：爆炸粒子减半
        const count = MOBILE_PERF ? 40 : 80;
        for (let j = 0; j < count; j++) explosions.push(new Particle(r.x, r.y, r.color));
        rockets.splice(i, 1);
      }
    }

    for (let i = explosions.length - 1; i >= 0; i--) {
      const p = explosions[i];
      p.update();
      if (p.alpha <= 0.01) { explosions.splice(i, 1); continue; }
      p.draw();
    }
    particleAnimationId = requestAnimationFrame(animate);
  };
  animate();
};

// ======================================
// 销毁礼花（关键！）
// ======================================
const destroyParticles = () => {
  if (particleAnimationId) cancelAnimationFrame(particleAnimationId);
  if (particleCanvas.value) {
    const ctx = particleCanvas.value.getContext("2d");
    ctx.clearRect(0, 0, particleCanvas.value.width, particleCanvas.value.height);
  }
};

// ======================================
// 销毁蛋糕Three.js（关键！）
// ======================================
const destroyCake = () => {
  if (!renderer) return;
  renderer.dispose();
  if (cakeCanvas.value) cakeCanvas.value.remove();
  renderer = null;
  scene = null;
  camera = null;
  controls = null;
};

// ======================================
// 蛋糕初始化（手机：极致性能模式）
// ======================================
onMounted(() => {
  if (!cakeCanvas.value) return;

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(45, 1, 0.1, 1000);
  camera.position.set(1.5, 1.5, 4);

  // 🔥 手机：关闭抗锯齿 + 低像素比 + 高性能
  renderer = new THREE.WebGLRenderer({
    canvas: cakeCanvas.value,
    antialias: !MOBILE_PERF,
    alpha: true,
    powerPreference: "high-performance",
  });

  renderer.setSize(400, 400);
  renderer.setPixelRatio(MOBILE_PERF ? 1 : Math.min(window.devicePixelRatio, 1.2));
  renderer.outputColorSpace = THREE.SRGBColorSpace;

  const ambientLight = new THREE.AmbientLight(0xffffff, 1.8);
  scene.add(ambientLight);
  const directionalLight = new THREE.DirectionalLight(0xffffff, 2.2);
  directionalLight.position.set(5, 8, 5);
  scene.add(directionalLight);
  const fillLight = new THREE.DirectionalLight(0xffc8e6, 1.2);
  fillLight.position.set(-5, 3, -3);
  scene.add(fillLight);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.08;
  controls.minDistance = 2;
  controls.maxDistance = 6;
  controls.enablePan = false;

  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath("/wen/draco/");
  const loader = new GLTFLoader();
  loader.setDRACOLoader(dracoLoader);

  loader.load(
    "cake.glb",
    (gltf) => {
      cakeModel = gltf.scene;
      cakeModel.traverse((child) => {
        if (child.isMesh && child.material) {
          child.material.needsUpdate = true;
          child.material.emissive = new THREE.Color(0x222222);
          child.material.emissiveIntensity = 0.4;
          child.material.metalness = 0.1;
        }
      });
      const box = new THREE.Box3().setFromObject(cakeModel);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3());
      const maxDim = Math.max(size.x, size.y, size.z);
      const scale = 3 / maxDim;
      const pivot = new THREE.Group();
      scene.add(pivot);
      pivot.add(cakeModel);
      cakeModel.scale.setScalar(scale);
      cakeModel.position.sub(center);
      cakeModel.position.x = 0.23;
      cakeModel.position.y = -1;
      cakeModel.position.z = 0;
      cakeModel = pivot;
    },
    (progress) => {},
    (err) => {
      console.error("模型加载失败，自动创建备用蛋糕", err);
      const geo = new THREE.CylinderGeometry(0.8, 0.8, 1.2, 16);
      const mat = new THREE.MeshStandardMaterial({ color: 0xff99cc });
      const fallback = new THREE.Mesh(geo, mat);
      fallback.position.y = -0.5;
      const pivot = new THREE.Group();
      scene.add(pivot);
      pivot.add(fallback);
      cakeModel = pivot;
    }
  );

  // 手机延迟渲染，避免黑屏
  setTimeout(() => {
    const animate = () => {
      const id = requestAnimationFrame(animate);
      if (!renderer) { cancelAnimationFrame(id); return; }
      if (cakeModel) cakeModel.rotation.y += 0.003;
      controls.update();
      renderer.render(scene, camera);
    };
    animate();
  }, MOBILE_PERF ? 400 : 0);

  initParticles();
});

onUnmounted(() => {
  destroyCake();
  destroyParticles();
  stopTextAnimation();
  if (rotateTimer) clearTimeout(rotateTimer);
});
</script>

<style scoped>
.birthday-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: #000;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 1;
}

.cake-wrapper {
  position: relative;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 400px;
  height: 400px;
  z-index: 2;
}

.cake-canvas {
  width: 400px;
  height: 400px;
  display: block;
  cursor: grab;
  z-index: 2;
  /* 加上过渡动画 */
  transition: all 1.5s ease-out;
}
.cake-canvas:active {
  cursor: grabbing;
}

/* 吹蜡烛后蛋糕透明+缩小消失 */
.cake-canvas.fade-out {
  opacity: 0;
  transform: scale(0.6);
}

.text-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
  pointer-events: none;
  z-index: 9998;
}

.particle-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  z-index: 1;
}

.blow-btn {
  position: fixed;
  left: 50%;
  bottom: 8%;
  transform: translateX(-50%);
  padding: 14px 42px;
  font-size: 18px;
  font-weight: bold;
  background: rgba(23, 105, 255, 0.2);
  color: #9cc8ff;
  border: 1px solid rgba(100, 150, 255, 0.5);
  border-radius: 50px;
  cursor: pointer;
  backdrop-filter: blur(10px);
  box-shadow: 0 0 20px rgba(100, 150, 255, 0.3);
  transition: all 0.3s ease;
  z-index: 9999 !important;
}
.blow-btn:hover {
  background: rgba(23, 105, 255, 0.35);
  color: #fff;
  box-shadow: 0 0 25px rgba(100, 150, 255, 0.6);
}

.blessing-box {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 9999 !important;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 短句祝福语 */
.blessing-item {
  position: absolute;
  font-weight: bold;
  text-shadow: 0 0 10px currentColor;
  animation:
    blessing 1.5s ease-out forwards,
    breathe 2.5s ease-in-out infinite;
  opacity: 0;
  padding: 6px 10px;
  text-align: center;
}

/* 书信框 */
.letter-box {
  position: absolute;
  bottom: 12%;
  left: 50%;
  transform: translateX(-50%);
  width: 88%;
  max-width: 500px;
  background: rgba(0, 0, 0, 0.5);
  border: 2px solid #ffccff;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 0 20px rgba(255, 180, 255, 0.3);
  animation: letterIn 1.2s ease forwards;
  opacity: 0;
}

.letter-content {
  color: #fff;
  font-size: 16px;
  line-height: 1.6;
  text-align: left;
  min-height: 120px;
}

/* 动画 */
@keyframes blessing {
  0% {
    transform: translate(-50%, -20px);
    opacity: 0;
  }
  100% {
    transform: translate(-50%, 0);
    opacity: 1;
  }
}

@keyframes breathe {
  0%,
  100% {
    transform: translateX(-50%) scale(1);
  }
  50% {
    transform: translateX(-50%) scale(1.08);
  }
}

@keyframes letterIn {
  0% {
    opacity: 0;
    transform: translate(-50%, 40px);
  }
  100% {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}

.lottery {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  z-index: 9999 !important;
  position: relative;
}

/* 抽奖布局样式 */
.lottery-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  position: relative;
}

.lottery-count {
  font-size: 22px;
  color: #fff;
  font-weight: bold;
  text-shadow: 0 0 12px #ffdf00;
  margin-bottom: 10px;
}

/* 整齐方形布局 8个卡片 */
.card-list {
  width: 460px;
  height: 460px;
  position: relative;
  margin: 0 auto;
}

.card {
  position: absolute;
  width: 140px;
  height: 170px;
  border-radius: 18px;
  background: linear-gradient(160deg, #ffcc00, #ff5e88);
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 16px;
  text-align: center;
  padding: 12px;
  box-shadow: 0 0 15px rgba(255, 165, 255, 0.4);
  transition: all 0.25s ease;
  border: 3px solid transparent;
}
.card.disabled {
  background: #666 !important;
  filter: grayscale(100%);
  opacity: 0.5 !important;
  box-shadow: none !important;
  border-color: #444 !important;
  pointer-events: none;
}
/* 转动发光边框 */
.card.active {
  border-color: #ffffff;
  box-shadow:
    0 0 30px #ffffff,
    0 0 50px #ffdf00;
  transform: scale(1.05);
}

/* 中奖超级高亮 */
.card.winner {
  border-color: #ffdf40 !important;
  box-shadow:
    0 0 50px #ffdf40,
    0 0 80px #ff9500 !important;
  animation: winPulse 0.7s infinite alternate;
  transform: scale(1.12);
  z-index: 5;
}

@keyframes winPulse {
  0% {
    transform: scale(1.12);
  }
  100% {
    transform: scale(1.18);
  }
}

/* 8个卡片 整齐顺时针排列 */
.card:nth-child(1) {
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
}
.card:nth-child(2) {
  top: 20px;
  right: 20px;
}
.card:nth-child(3) {
  top: 50%;
  right: 20px;
  transform: translateY(-50%);
}
.card:nth-child(4) {
  bottom: 20px;
  right: 20px;
}
.card:nth-child(5) {
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
}
.card:nth-child(6) {
  bottom: 20px;
  left: 20px;
}
.card:nth-child(7) {
  top: 50%;
  left: 20px;
  transform: translateY(-50%);
}
.card:nth-child(8) {
  top: 20px;
  left: 20px;
}

/* 中间开始按钮 */
.start-btn {
  position: absolute;
  bottom: -25%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 22px 50px;
  font-size: 24px;
  background: linear-gradient(90deg, #ffdf00, #ff2e63);
  color: #fff;
  border: none;
  border-radius: 60px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 0 35px rgba(255, 223, 0, 0.8);
  transition: all 0.3s ease;
  z-index: 10;
}

.start-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  box-shadow: none;
}

/* 最终中奖展示居中大字 方便截图 */
.final-result {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 20;
}
.result-card {
  background: linear-gradient(160deg, #ffdf00, #ff5888);
  padding: 50px 60px;
  border-radius: 24px;
  text-align: center;
  box-shadow: 0 0 40px rgba(255, 210, 0, 0.6);
  min-width: 380px;
}
.result-title {
  font-size: 30px;
  color: #fff;
  font-weight: bold;
  margin-bottom: 30px;
  text-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
}
.result-prize {
  font-size: 26px;
  color: #fff;
  font-weight: bold;
  margin: 16px 0;
  padding: 12px 20px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  text-shadow: 0 0 6px rgba(0, 0, 0, 0.2);
}
.result-tip {
  margin-top: 28px;
  font-size: 18px;
  color: #fff;
  opacity: 0.9;
}

/* 抽完后卡牌半透明弱化 */
.card-list {
  transition: all 0.5s ease;
}
.final-result ~ .card-list .card {
  opacity: 0.15 !important;
  filter: grayscale(100%);
  pointer-events: none;
}
</style>
