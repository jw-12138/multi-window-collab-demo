<template>
  <div
    style="position: absolute; top: 0; left: 0; padding-left: 20px">
    <pre style="width: 100%">screenX: {{ screenX }}
screenY: {{ screenY }}
outerWidth: {{ outerWidth }}
outerHeight: {{ outerHeight }}
browserTopNotch: {{ browserTopNotch }}
    </pre>
    <button @click="newWindow" v-show="screenPermissionGranted">new window</button>
    <button @click="enableScreenPermission" v-show="!screenPermissionGranted">enable screen permission</button>
  </div>
  <div class="container" v-if="screenPermissionGranted">
    <!-- 所有窗口之间的连线 -->
    <div class="line" v-for="line in lines" :key="line.id" :style="{
      left: line.startX - smoothScreen.x - smoothScreen.leftNotch + 'px',
      top: line.startY - smoothScreen.y - smoothScreen.topNotch + 'px',
      height: line.height + 'px',
      transform: 'rotate(' + line.angle + 'deg)',
    }">
    </div>
    <!-- 当前窗口的圆 -->
    <div class="circle" style="transform: translate(-50%, -50%); top: 50%; left: 50%">
    </div>
    <!-- 其他窗口的圆 -->
    <div class="circle" v-for="w in Object.keys(smoothWindowPositions)" :key="w" :style="{
      left: smoothWindowPositions[w].ballCenter[0] - smoothScreen.x - smoothScreen.leftNotch + 'px',
      top: smoothWindowPositions[w].ballCenter[1] - smoothScreen.y - smoothScreen.topNotch + 'px'
    }">
    </div>
  </div>
</template>

<style>
body {
  overflow: hidden;
  padding: 0;
  margin: 0;
}

.container {
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.circle {
  position: absolute;
  height: 200px;
  width: 200px;
  border-radius: 200px;
  border: 1px solid #000;
}

.line {
  position: absolute;
  width: 1px;
  height: 1px;
  background: #000;
  transform-origin: top;
}
</style>

<script setup>
import {onMounted, ref, watch} from 'vue'

let currentScreenDetails = ref({})

let screenX = ref(0)
let screenY = ref(0)
let outerWidth = ref(0)
let outerHeight = ref(0)
let ballCenter = ref([0, 0])

let windowMap = ref({})
let topMap = ref({})
let smoothPositions = {} // 内部存储平滑位置（非响应式，用于计算）
let smoothWindowPositions = ref({}) // 响应式，用于模板渲染
let smoothBallCenter = [0, 0] // 当前窗口自己的平滑位置
let smoothScreen = ref({ x: 0, y: 0, topNotch: 0, leftNotch: 0 }) // 平滑后的屏幕位置（响应式）
const LERP_FACTOR = 0.2 // 插值系数，越小越平滑

let windowID = location.search.replace('?id=', '')

// 线性插值函数
function lerp(start, end, factor) {
  return start + (end - start) * factor
}

// 生成随机6位ID
function generateID() {
  return Math.random().toString(36).substring(2, 8)
}

if (!windowID) {
  windowID = generateID()
}

window.addEventListener('beforeunload', function (e) {
  localStorage.removeItem('window_' + windowID)
})

let lines = ref([])

onMounted(() => {
  for (let i = 0; i < 360; i++) {
    topMap.value[i] = 0
  }

  // 监听 storage 事件，实时响应其他窗口的变化
  window.addEventListener('storage', (e) => {
    if (e.key?.startsWith('window_') && e.newValue === null) {
      // 某个窗口被移除了，清理对应数据
      delete windowMap.value[e.key]
      delete smoothPositions[e.key]
      delete smoothWindowPositions.value[e.key]
      // lines 会在下一帧 run() 中自动重新计算
    }
  })

  enableScreenPermission()
})

let browserTopNotch = 0
let browserLeftNotch = 0


window.requestAnimationFrame(run)

function run(timestamp) {
  screenX.value = window.screenX
  screenY.value = window.screenY
  outerWidth.value = window.outerWidth
  outerHeight.value = window.outerHeight

  // calculate circle position related to the actual screen
  // 浏览器边框：上边距 = outerHeight - innerHeight，左边距 = outerWidth - innerWidth（假设左右对称，取一半）
  browserTopNotch = window.outerHeight - window.innerHeight
  browserLeftNotch = (window.outerWidth - window.innerWidth) / 2

  let ballCenterX = (window.innerWidth / 2 - 100) + screenX.value + browserLeftNotch
  let ballCenterY = (window.innerHeight / 2 - 100) + screenY.value + browserTopNotch

  // 当前窗口的原始位置（用于写入 localStorage）
  ballCenter.value = [ballCenterX, ballCenterY]

  // 对当前窗口自己的位置也做插值
  if (smoothBallCenter[0] === 0 && smoothBallCenter[1] === 0) {
    // 初始化
    smoothBallCenter = [ballCenterX, ballCenterY]
    smoothScreen.value = { x: screenX.value, y: screenY.value, topNotch: browserTopNotch, leftNotch: browserLeftNotch }
  } else {
    smoothBallCenter[0] = lerp(smoothBallCenter[0], ballCenterX, LERP_FACTOR)
    smoothBallCenter[1] = lerp(smoothBallCenter[1], ballCenterY, LERP_FACTOR)
    smoothScreen.value = {
      x: lerp(smoothScreen.value.x, screenX.value, LERP_FACTOR),
      y: lerp(smoothScreen.value.y, screenY.value, LERP_FACTOR),
      topNotch: lerp(smoothScreen.value.topNotch, browserTopNotch, LERP_FACTOR),
      leftNotch: lerp(smoothScreen.value.leftNotch, browserLeftNotch, LERP_FACTOR)
    }
  }

  localStorage.setItem('window_' + windowID, JSON.stringify({
    screenX: screenX.value,
    screenY: screenY.value,
    outerWidth: outerWidth.value,
    outerHeight: outerHeight.value,
    updateTime: Date.now(),
    ballCenter: ballCenter.value,
    browserTopNotch: browserTopNotch
  }))

  let localKeys = Object.keys(localStorage)

  // 收集所有有效窗口（包括当前窗口，使用平滑后的位置）
  let allWindows = {}
  allWindows['window_' + windowID] = {
    ballCenter: smoothBallCenter
  }

  localKeys.forEach((key) => {
    if (!key.startsWith('window_')) {
      return
    }

    if (key === 'window_' + windowID) {
      return
    }

    let windowObject = JSON.parse(localStorage.getItem(key))

    // remove window if not updated for 200ms
    if (Date.now() - windowObject.updateTime > 200) {
      // 精确移除失效窗口，不影响其他窗口
      localStorage.removeItem(key)
      delete windowMap.value[key]
      delete smoothPositions[key]
      delete smoothWindowPositions.value[key]
      return
    }

    windowMap.value[key] = windowObject

    // 初始化或更新平滑位置
    if (!smoothPositions[key]) {
      // 首次出现，直接使用目标位置
      smoothPositions[key] = {
        ballCenter: [...windowObject.ballCenter]
      }
    } else {
      // 插值平滑
      smoothPositions[key].ballCenter[0] = lerp(
        smoothPositions[key].ballCenter[0],
        windowObject.ballCenter[0],
        LERP_FACTOR
      )
      smoothPositions[key].ballCenter[1] = lerp(
        smoothPositions[key].ballCenter[1],
        windowObject.ballCenter[1],
        LERP_FACTOR
      )
    }

    // 更新响应式对象用于模板渲染
    smoothWindowPositions.value[key] = {
      ballCenter: [...smoothPositions[key].ballCenter]
    }

    allWindows[key] = {
      ballCenter: smoothPositions[key].ballCenter
    }
  })

  // 计算所有窗口两两之间的连线
  let newLines = []
  let windowKeys = Object.keys(allWindows)

  for (let i = 0; i < windowKeys.length; i++) {
    for (let j = i + 1; j < windowKeys.length; j++) {
      let w1 = allWindows[windowKeys[i]]
      let w2 = allWindows[windowKeys[j]]

      // ballCenter 已经是圆的左上角位置，圆心需要 +100
      let startX = w1.ballCenter[0] + 100
      let startY = w1.ballCenter[1] + 100
      let endX = w2.ballCenter[0] + 100
      let endY = w2.ballCenter[1] + 100

      let lineLength = Math.sqrt(Math.pow(endX - startX, 2) + Math.pow(endY - startY, 2))
      // 原来的角度计算方式
      let lineAngle = Math.atan2(startY - endY, startX - endX) * 180 / Math.PI + 90

      newLines.push({
        id: windowKeys[i] + '_' + windowKeys[j],
        startX: startX,
        startY: startY,
        height: lineLength,
        angle: lineAngle
      })
    }
  }

  lines.value = newLines

  window.requestAnimationFrame(run)
}

let screenPermissionGranted = ref(false)

async function enableScreenPermission() {
  let screenDetails

  try {
    screenDetails = await window.getScreenDetails()
    screenPermissionGranted.value = true
    currentScreenDetails.value = screenDetails.currentScreen
  } catch (e) {
    return false
  }
}

function newWindow() {
  window.open(location.origin + '/?id=' + generateID(), '_blank', 'width=400,height=400')
}


</script>
