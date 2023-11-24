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
    <div class="circle" style="transform: translate(-50%, -50%); top: 50%; left: 50%">
      <div class="line" v-for="line in lines" :style="{
        height: (line ? line.height : 0) + 'px',
        transform: 'rotate(' + (line ? line.angle : 0) + 'deg)',
      }">
      </div>
    </div>
    <div class="circle" v-for="window in Object.keys(windowMap)" :style="{
      transition: 'all 0.1s ease-in-out',
      left: windowMap[window].ballCenter[0] - screenX + 'px',
      top: windowMap[window].ballCenter[1] - screenY - browserTopNotch + 'px'
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
  top: 50%;
  left: 50%;
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
let windowID = location.search.replace('?id=', '')

if (!windowID) {
  windowID = '1'
}

window.addEventListener('beforeunload', function (e) {
  localStorage.removeItem('window_' + windowID)
})

let lines = ref([])

onMounted(() => {
  for (let i = 0; i < 360; i++) {
    topMap.value[i] = 0
  }

  enableScreenPermission()
})

let browserTopNotch = 0


window.requestAnimationFrame(run)

function run(timestamp) {
  screenX.value = window.screenX
  screenY.value = window.screenY
  outerWidth.value = window.outerWidth
  outerHeight.value = window.outerHeight

  // calculate circle position related to the actual screen
  browserTopNotch = window.outerHeight - window.innerHeight
  let ballCenterX = (window.innerWidth / 2 - 100) + screenX.value
  let ballCenterY = (window.innerHeight / 2 - 100) + screenY.value + browserTopNotch

  ballCenter.value = [ballCenterX, ballCenterY]

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

  localKeys.forEach((key, index) => {
    if (!key.startsWith('window_')) {
      return false
    }

    if (key === 'window_' + windowID) {
      return false
    }

    let windowObject = JSON.parse(localStorage.getItem(key))

    // remove window if not updated for 200ms
    if (Date.now() - windowObject.updateTime > 200) {
      try {
        // remove things sometimes not working
        // so we go brutal here

        localStorage.clear()
        window.location.reload()
      } catch (_) {
        // do nothing
      }
      return false
    }

    windowMap.value[key] = windowObject

    let lineLength = Math.sqrt(Math.pow(ballCenter.value[0] - windowObject.ballCenter[0], 2) + Math.pow(ballCenter.value[1] - windowObject.ballCenter[1], 2))
    let lineAngle = Math.atan2(ballCenter.value[1] - windowObject.ballCenter[1], ballCenter.value[0] - windowObject.ballCenter[0]) * 180 / Math.PI + 90

    lines.value[index] = {
      id: key,
      height: lineLength,
      angle: lineAngle
    }
  })

  window.requestAnimationFrame(run)
}

let mousePoint = ref([0, 0])

window.addEventListener('mousemove', (e) => {
  mousePoint.value = [e.pageX, e.pageY]
})

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
  window.open(location.origin + '/?id=' + (parseInt(windowID) + 1), '_blank', 'width=400,height=400')
}


</script>
