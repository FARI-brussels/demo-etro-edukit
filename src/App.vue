<script setup>
import { ref, reactive, computed } from 'vue'
import ThreeSignalCanvas from './components/ThreeSignalCanvas.vue'
import { WebSerialService } from './services/webserial'

const service = new WebSerialService()

const connected = ref(false)
const statusMsg = ref('')

const debug = ref(false)
let lastLogMs = 0
function dlog(...args) {
  if (!debug.value) return
  const now = performance.now()
  if (now - lastLogMs < 250) return
  lastLogMs = now
  console.log(...args)
}

const selectedSensor = ref('S1')
const selectedSignal = ref('TIA')

// Sensor on/off toggles
const s1Enabled = ref(false)
const s2Enabled = ref(false)

const s1Options = ['TIA', 'HPF', 'LPF', 'AMP', 'HR', 'SPO2', 'TEMP']
const s2Options = ['IR', 'RED', 'GREEN', 'HR', 'SPO2']

const maxPoints = 500
const labels = ref([])
const s1 = reactive({ TIA: [], HPF: [], LPF: [], AMP: [], HR: [], SPO2: [], TEMP: [] })
const s2 = reactive({ IR: [], RED: [], GREEN: [], HR: [], SPO2: [] })

function pushSample(arr, v) {
  const num = Number.isFinite(v) ? v : NaN
  if (!Number.isFinite(num)) return
  if (arr.length < maxPoints) {
    arr.push(num)
    if (arr.length === maxPoints) {
      arr._rbIndex = 0
      arr._rbFull = true
    }
    return
  }
  if (typeof arr._rbIndex !== 'number') arr._rbIndex = 0
  arr[arr._rbIndex] = num
  arr._rbIndex = (arr._rbIndex + 1) % maxPoints
  arr._rbFull = true
}

service.setOnLine((raw) => {
  const line = String(raw || '').trim()
  if (!line) return
  dlog('[RAW]', line)
  const low = line.toLowerCase()
  try {
    if (low.startsWith('s1,')) {
      const parts = line.split(',')
      const vals = parts.slice(1).map(x => parseFloat(x))
      if (vals.length >= 7) {
        pushSample(s1.TIA, vals[0])
        pushSample(s1.HPF, vals[1])
        pushSample(s1.LPF, vals[2])
        pushSample(s1.AMP, vals[3])
        pushSample(s1.HR, vals[4])
        pushSample(s1.SPO2, vals[5])
        pushSample(s1.TEMP, vals[6])
        if (debug.value) console.log('[S1]', { TIA: vals[0], HPF: vals[1], LPF: vals[2], AMP: vals[3], HR: vals[4], SPO2: vals[5], TEMP: vals[6] })
        statusMsg.value = ''
      }
      return
    }
    if (low.startsWith('s2,')) {
      const parts = line.split(',')
      const vals = parts.slice(1).map(x => parseFloat(x))
      if (vals.length >= 5) {
        pushSample(s2.IR, vals[0])
        pushSample(s2.RED, vals[1])
        pushSample(s2.GREEN, vals[2])
        pushSample(s2.HR, vals[3])
        pushSample(s2.SPO2, vals[4])
        if (debug.value) console.log('[S2]', { IR: vals[0], RED: vals[1], GREEN: vals[2], HR: vals[3], SPO2: vals[4] })
        statusMsg.value = ''
      }
      return
    }
    if (low.startsWith('i,')) {
      if (debug.value) console.log('[INFO]', line)
      statusMsg.value = line
      return
    }
  } catch (e) {
    console.warn('Parse error for line', line, e)
  }
})

async function onConnect() {
  try {
    await service.connect({ baudRate: 115200 })
    connected.value = true
    statusMsg.value = 'Connected'
  } catch (e) {
    statusMsg.value = String(e)
  }
}

async function onDisconnect() {
  try {
    await service.disconnect()
    connected.value = false
    // reset toggles when disconnected
    s1Enabled.value = false
    s2Enabled.value = false
    statusMsg.value = 'Disconnected'
  } catch (e) {
    statusMsg.value = String(e)
  }
}

async function toggleSensor(sensor, enable) {
  try {
    await service.setSensorEnabled(sensor, enable)
  } catch (e) {
    statusMsg.value = String(e)
  }
}

const ledS1 = ref('red')
const ledS2 = ref('red')

async function setLed(sensor, mode) {
  try {
    await service.setLedMode(sensor, mode)
  } catch (e) {
    statusMsg.value = String(e)
  }
}

const visibleData = computed(() => {
  const sensor = selectedSensor.value
  const sig = selectedSignal.value
  if (sensor === 'S1') return s1[sig] || []
  if (sensor === 'S2') return s2[sig] || []
  return []
})

const colorMap = {
  S1: {
    TIA: '#00e5ff', HPF: '#6a5acd', LPF: '#1de9b6', AMP: '#ff8a65', HR: '#ffd54f', SPO2: '#a5d6a7', TEMP: '#ff5252'
  },
  S2: {
    IR: '#ff1744', RED: '#ff5252', GREEN: '#00e676', HR: '#ffd740', SPO2: '#b39ddb'
  }
}

const canvasColor = computed(() => {
  const s = selectedSensor.value
  const k = selectedSignal.value
  return (colorMap[s] && colorMap[s][k]) || '#00e5ff'
})

function onSensorChange() {
  if (selectedSensor.value === 'S1' && !s1Options.includes(selectedSignal.value)) selectedSignal.value = 'TIA'
  if (selectedSensor.value === 'S2' && !s2Options.includes(selectedSignal.value)) selectedSignal.value = 'IR'
}
</script>

<template>
  <div style="max-width: 1100px; margin: 0 auto; padding: 16px;">
    <h2>PPG EduKit Web (Web Serial)</h2>

    <div style="display:flex; gap:8px; align-items:center; flex-wrap: wrap; margin-bottom: 8px;">
      <button @click="onConnect" :disabled="connected">Connect</button>
      <button @click="onDisconnect" :disabled="!connected">Disconnect</button>
      <span :style="{ color: connected ? 'green' : 'red', marginLeft: '8px' }">{{ connected ? 'Connected' : 'Disconnected' }}</span>
      <label style="margin-left:12px; display:flex; align-items:center; gap:6px;">
        <input type="checkbox" v-model="debug" /> Debug logs
      </label>
    </div>

    <div style="display:flex; gap:16px; flex-wrap: wrap; margin: 8px 0; align-items: center;">
      <div>
        <div style="font-weight:600; margin-bottom: 4px;">Sensor Control</div>
        <div style="display:flex; gap:16px; align-items:center;">
          <label style="display:flex; align-items:center; gap:6px;">
            <span>S1</span>
            <input type="checkbox" v-model="s1Enabled" @change="toggleSensor('S1', s1Enabled)" :disabled="!connected" />
          </label>
          <label style="display:flex; align-items:center; gap:6px;">
            <span>S2</span>
            <input type="checkbox" v-model="s2Enabled" @change="toggleSensor('S2', s2Enabled)" :disabled="!connected" />
          </label>
        </div>
      </div>
      <div>
        <div style="font-weight:600; margin-bottom: 4px;">LED S1</div>
        <div role="radiogroup" style="display:flex; gap:10px; align-items:center;">
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS1" value="red" v-model="ledS1" @change="setLed('S1', ledS1)" :disabled="!connected" /> Red
          </label>
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS1" value="green" v-model="ledS1" @change="setLed('S1', ledS1)" :disabled="!connected" /> Green
          </label>
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS1" value="infrared" v-model="ledS1" @change="setLed('S1', ledS1)" :disabled="!connected" /> Infrared
          </label>
        </div>
      </div>
      <div>
        <div style="font-weight:600; margin-bottom: 4px;">LED S2</div>
        <div role="radiogroup" style="display:flex; gap:10px; align-items:center;">
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS2" value="red" v-model="ledS2" @change="setLed('S2', ledS2)" :disabled="!connected" /> Red
          </label>
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS2" value="green" v-model="ledS2" @change="setLed('S2', ledS2)" :disabled="!connected" /> Green
          </label>
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS2" value="blue" v-model="ledS2" @change="setLed('S2', ledS2)" :disabled="!connected" /> Blue
          </label>
          <label style="display:flex; align-items:center; gap:4px;">
            <input type="radio" name="ledS2" value="infrared" v-model="ledS2" @change="setLed('S2', ledS2)" :disabled="!connected" /> Infrared
          </label>
        </div>
      </div>
    </div>

    <div style="display:flex; gap:16px; align-items:center; flex-wrap: wrap;">
      <div>
        <label>Sensor:</label>
        <select v-model="selectedSensor" @change="onSensorChange">
          <option value="S1">S1 (Custom)</option>
          <option value="S2">S2 (Commercial)</option>
        </select>
      </div>
      <div>
        <label>Signal:</label>
        <select v-model="selectedSignal">
          <template v-if="selectedSensor==='S1'">
            <option v-for="k in s1Options" :key="k" :value="k">{{ k }}</option>
          </template>
          <template v-else>
            <option v-for="k in s2Options" :key="k" :value="k">{{ k }}</option>
          </template>
        </select>
      </div>
    </div>

    <div style="margin-top: 14px;">
      <ThreeSignalCanvas :data="visibleData" :color="canvasColor" background="#0b1020" />
    </div>

    <div v-if="statusMsg" style="margin-top: 8px; color: #666;">{{ statusMsg }}</div>

    <p style="margin-top:12px; color:#888; font-size: 12px;">
      Tip: try switching signals; colors adapt to make each channel distinct.
    </p>
  </div>
</template>

<style>
body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif; }
button { padding: 6px 10px; }
select { padding: 4px 6px; }
</style>
