<template>
  <div class="serial-debugger-layout">
    <!-- 左侧参数区 -->
    <div class="sidebar">
      <div class="sidebar-content">
        <div class="card param-card">
          <h3>串口参数</h3>
          <div class="form-group">
            <label>波特率</label>
            <select v-model="baudRate" :disabled="isOpen">
              <option v-for="rate in baudRates" :key="rate" :value="rate">{{ rate }}</option>
            </select>
          </div>
          <div class="form-group">
            <label>数据位</label>
            <select v-model="dataBits" :disabled="isOpen">
              <option v-for="bit in dataBitsList" :key="bit" :value="bit">{{ bit }}</option>
            </select>
          </div>
          <div class="form-group">
            <label>停止位</label>
            <select v-model="stopBits" :disabled="isOpen">
              <option v-for="bit in stopBitsList" :key="bit" :value="bit">{{ bit }}</option>
            </select>
          </div>
          <div class="form-group">
            <label>校验位</label>
            <select v-model="parity" :disabled="isOpen">
              <option v-for="p in parityList" :key="p" :value="p">{{ p }}</option>
            </select>
          </div>
          <div class="form-group">
            <label>缓冲区</label>
            <select v-model="bufferSize" :disabled="isOpen">
              <option v-for="size in bufferSizes" :key="size" :value="size">{{ size }}</option>
            </select>
          </div>
          <div class="form-group">
            <label>流控</label>
            <select v-model="flowControl" :disabled="isOpen">
              <option v-for="f in flowControlList" :key="f" :value="f">{{ f }}</option>
            </select>
          </div>
          <div class="btn-group">
            <button @click="requestPort">选择串口</button>
            <button :disabled="!port" @click="togglePort" :class="isOpen ? 'open' : ''">{{ isOpen ? '关闭串口' : '打开串口' }}</button>
          </div>
          <div class="status-box">
            <span v-if="portName">已选择{{ portName }}</span>
            <span v-else>未选择串口</span>
          </div>
        </div>
        <div class="card setting-card">
          <div class="form-group">
            <label>接收设置：</label>
            <div class="radio-group">
              <label><input type="radio" v-model="recvMode" value="ascii" /> ASCII</label>
              <label><input type="radio" v-model="recvMode" value="hex" /> HEX</label>
            </div>
          </div>
        </div>
        <div class="card setting-card">
          <div class="form-group">
            <label>发送设置：</label>
            <div class="radio-group">
              <label><input type="radio" v-model="sendMode" value="ascii" /> ASCII</label>
              <label><input type="radio" v-model="sendMode" value="hex" /> HEX</label>
            </div>
          </div>
        </div>
      </div>
    </div>
    <!-- 右侧日志/收发区 -->
    <div class="main-area">
      <div class="log-toolbar">
        <span>串口日志</span>
        <button @click="clearLogs" style="margin-left:auto;">清空日志</button>
      </div>
      <div class="log-area">
        <div class="log-viewer" ref="logViewerRef">
          <div v-for="(log, idx) in logs" :key="idx" :class="['log-line', log.type === 'send' ? 'send-log' : 'recv-log']">
            {{ formatLogText(log) }}
          </div>
        </div>
      </div>
      <div class="send-bar">
        <textarea v-model="sendText" rows="3" :placeholder="sendMode === 'hex' ? '请输入HEX字节流，如 01 02 0A' : '请输入要发送的内容'" @keyup.enter="sendData"></textarea>
        <button :disabled="!isOpen" @click="sendData">发送</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, watch } from 'vue'

const port = ref(null)
const portName = ref('')
const isOpen = ref(false)
const baudRate = ref(115200)
const baudRates = [110, 300, 600, 1200, 2400, 4800, 9600, 14400, 19200, 38400, 57600, 115200, 230400, 460800]
const dataBits = ref(8)
const dataBitsList = [8, 7]
const stopBits = ref(1)
const stopBitsList = [1, 2]
const parity = ref('none')
const parityList = ['none', 'even', 'odd']
const bufferSize = ref(1024)
const bufferSizes = [255, 512, 1024, 2048, 4096, 8192]
const flowControl = ref('none')
const flowControlList = ['none', 'hardware']
const sendText = ref('')
const logs = ref([])
const logViewerRef = ref(null)
const recvMode = ref('ascii')
const sendMode = ref('ascii')
let reader = null
let writer = null

function getFullTimeStr() {
  const d = new Date()
  const pad = n => n.toString().padStart(2, '0')
  const date = `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}`
  const time = `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}.${d.getMilliseconds().toString().padStart(3, '0')}`
  return `${date} ${time}`
}

async function requestPort() {
  try {
    const selected = await navigator.serial.requestPort()
    port.value = selected
    const info = selected.getInfo ? selected.getInfo() : {}
    if (info.usbVendorId || info.usbProductId) {
      portName.value = `VID: ${info.usbVendorId || '--'} PID: ${info.usbProductId || '--'}`
    } else {
      portName.value = '串口'
    }
  } catch (e) {
    port.value = null
    portName.value = ''
  }
}

async function togglePort() {
  if (!isOpen.value) {
    try {
      await port.value.open({
        baudRate: Number(baudRate.value),
        dataBits: Number(dataBits.value),
        stopBits: Number(stopBits.value),
        parity: parity.value,
        flowControl: flowControl.value
      })
      isOpen.value = true
      startRead()
      writer = port.value.writable.getWriter()
    } catch (e) {
      alert('串口打开失败：' + e)
    }
  } else {
    try {
      if (reader) {
        await reader.cancel();
        // 等待startRead的finally释放锁
        await new Promise(resolve => setTimeout(resolve, 50));
      }
      if (writer) writer.releaseLock();
      await port.value.close();
      isOpen.value = false;
    } catch (e) {
      alert('串口关闭失败：' + e)
    }
  }
}

async function sendData() {
  if (!isOpen.value) {
    alert('请先打开串口！')
    return
  }
  if (!sendText.value.trim()) {
    alert('请输入发送内容')
    return
  }
  if (!writer) {
    alert('串口写入通道未就绪，请重新打开串口！')
    return
  }
  let data
  let logStr = ''
  const time = `[${getFullTimeStr()}]# SEND ${sendMode.value === 'hex' ? 'HEX' : 'ASCII'}> `
  if (sendMode.value === 'hex') {
    data = new Uint8Array(sendText.value.trim().split(/\s+/).map(h => parseInt(h, 16)))
    logStr = time + Array.from(data).map(b => b.toString(16).toUpperCase().padStart(2, '0')).join(' ')
  } else {
    data = new TextEncoder().encode(sendText.value)
    logStr = time + sendText.value
  }
  await writer.write(data)
  logs.value.push({ type: 'send', text: logStr, raw: data })
}

async function startRead() {
  reader = port.value.readable.getReader()
  try {
    while (isOpen.value) {
      const { value, done } = await reader.read()
      if (done) break
      if (value) {
        const time = `[${getFullTimeStr()}]# RECV ${recvMode.value === 'hex' ? 'HEX' : 'ASCII'}> `
        let logStr = ''
        if (recvMode.value === 'hex') {
          logStr = time + Array.from(value).map(b => b.toString(16).toUpperCase().padStart(2, '0')).join(' ')
        } else {
          logStr = time + new TextDecoder().decode(value)
        }
        logs.value.push({ type: 'recv', text: logStr, raw: value })
      }
    }
  } catch (e) {
    // 读取被取消
  } finally {
    reader.releaseLock()
    reader = null
  }
}

function clearLogs() {
  logs.value = []
}

function formatLogText(log) {
  // 只格式化显示，不影响原始数据
  if (log.type === 'recv') {
    if (recvMode.value === 'hex') {
      // 若当前为HEX显示，强制转HEX
      if (log.raw) {
        // 保留完整时间戳前缀（到毫秒）
        const prefix = log.text.match(/^\[[^\]]+\]/)?.[0] || ''
        return prefix + log.text.slice(prefix.length).replace(/^(# RECV ).*/, '# RECV HEX> ') + Array.from(log.raw).map(b => b.toString(16).toUpperCase().padStart(2, '0')).join(' ')
      }
    } else {
      // ASCII显示
      if (log.raw) {
        const prefix = log.text.match(/^\[[^\]]+\]/)?.[0] || ''
        return prefix + log.text.slice(prefix.length).replace(/^(# RECV ).*/, '# RECV ASCII> ') + new TextDecoder().decode(log.raw)
      }
    }
  }
  if (log.type === 'send') {
    if (sendMode.value === 'hex') {
      if (log.raw) {
        const prefix = log.text.match(/^\[[^\]]+\]/)?.[0] || ''
        return prefix + log.text.slice(prefix.length).replace(/^(# SEND ).*/, '# SEND HEX> ') + Array.from(log.raw).map(b => b.toString(16).toUpperCase().padStart(2, '0')).join(' ')
      }
    } else {
      if (log.raw) {
        const prefix = log.text.match(/^\[[^\]]+\]/)?.[0] || ''
        return prefix + log.text.slice(prefix.length).replace(/^(# SEND ).*/, '# SEND ASCII> ') + new TextDecoder().decode(log.raw)
      }
    }
  }
  return log.text
}

// 自动滚动到底部
watch(
  () => logs.value.length,
  async () => {
    await nextTick()
    if (logViewerRef.value) {
      logViewerRef.value.scrollTop = logViewerRef.value.scrollHeight
    }
  }
)
</script>

<style scoped>
.radio-group {
  display: flex;
  gap: 16px;
  margin-top: 4px;
}
.card {
  background: #fafdff;
  border-radius: 14px;
  box-shadow: 0 2px 10px rgba(25,118,210,0.07);
  padding: 18px 16px 12px 16px;
  margin-bottom: 18px;
}
.param-card {
  font-size: 14px;
  padding: 12px 12px 6px 12px;
}
.param-card h3 {
  font-size: 17px;
  margin-bottom: 12px;
}
.param-card .form-group label {
  font-size: 13px;
}
.param-card .form-group select {
  font-size: 14px;
  padding: 6px 8px;
  border-radius: 7px;
}
.param-card .form-group {
  margin-bottom: 7px;
  gap: 4px;
}
.setting-card {
  font-size: 15px;
  padding: 14px 14px 8px 14px;
}
.setting-card .form-group label {
  font-size: 14px;
}
.setting-card .radio-group label {
  font-size: 15px;
}
.serial-debugger-layout {
  display: flex;
  min-width: 980px;
  background: #f6faff;
  border-radius: 0 0 18px 18px;
  box-shadow: 0 6px 32px rgba(25, 118, 210, 0.10);
  overflow: hidden;
  margin-top: 12px;
}
.sidebar {
  width: 290px;
  background: linear-gradient(135deg, #e3f0ff 0%, #f6faff 100%);
  padding: 32px 20px;
  border-right: 1.5px solid #e0e7ef;
  display: flex;
  flex-direction: column;
  gap: 18px;
  box-shadow: 2px 0 8px rgba(25,118,210,0.03);
  height: 100%;
  overflow-y: auto;
}
.sidebar-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 18px;
}
.sidebar h3 {
  font-size: 22px;
  font-weight: bold;
  margin-bottom: 18px;
  color: #1976d2;
  letter-spacing: 1px;
}
.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 10px;
}
.form-group label {
  font-size: 15px;
  color: #1976d2;
  font-weight: 500;
}
.form-group select {
  padding: 8px 12px;
  border-radius: 8px;
  border: 1.5px solid #b6c8e6;
  font-size: 16px;
  background: #fafdff;
  transition: border 0.2s;
}
.form-group select:focus {
  border: 1.5px solid #1976d2;
}
.btn-group {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}
.btn-group button {
  flex: 1;
  padding: 12px 0;
  border: none;
  border-radius: 8px;
  background: linear-gradient(90deg, #1976d2 60%, #42a5f5 100%);
  color: #fff;
  font-size: 17px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(25,118,210,0.08);
  transition: background 0.2s, box-shadow 0.2s;
}
.btn-group button.open {
  background: linear-gradient(90deg, #e53935 60%, #ff7043 100%);
}
.btn-group button:hover {
  box-shadow: 0 4px 16px rgba(25,118,210,0.13);
  filter: brightness(1.08);
}
.status-box {
  margin-top: 16px;
  font-size: 14px;
  color: #1976d2;
  background: #e3f2fd;
  border-radius: 6px;
  padding: 8px 10px;
  min-height: 32px;
  font-weight: 500;
  letter-spacing: 0.5px;
}
.main-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 0 28px 0 28px;
  background: #fff;
  border-radius: 0 0 18px 0;
  box-shadow: 0 2px 12px rgba(25,118,210,0.04) inset;
  min-width: 0;
  min-height: 0;
}
.log-toolbar {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 10px;
  font-size: 18px;
  color: #1976d2;
  font-weight: bold;
  letter-spacing: 1px;
}
.log-area {
  flex: 1 1 0;
  min-height: 340px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  margin-bottom: 28px;
  min-width: 0;
  min-height: 0;
}
.log-area textarea {
  width: 100%;
  height: 100%;
  min-height: 340px;
  font-size: 16px;
  font-family: 'Fira Mono', 'Consolas', 'Menlo', monospace;
  border-radius: 8px;
  border: 1.5px solid #b6c8e6;
  padding: 12px;
  resize: vertical;
  background: #fafdff;
  color: #222;
  box-shadow: 0 1px 4px rgba(25,118,210,0.03) inset;
}
.send-bar {
  display: flex;
  align-items: flex-end;
  gap: 16px;
  padding-bottom: 24px;
  border-top: 2px solid #e3eaf5;
  background: #fff;
}
.send-bar textarea {
  flex: 1;
  min-height: 70px;
  max-height: 140px;
  font-size: 17px;
  padding: 10px;
  border-radius: 8px;
  border: 1.5px solid #b6c8e6;
  resize: vertical;
  background: #fafdff;
  font-family: 'Fira Mono', 'Consolas', 'Menlo', monospace;
}
.send-bar label {
  margin-bottom: 10px;
  font-size: 16px;
  white-space: nowrap;
  color: #1976d2;
  font-weight: 500;
}
.send-bar button {
  padding: 16px 36px;
  border: none;
  border-radius: 8px;
  background: linear-gradient(90deg, #1976d2 60%, #42a5f5 100%);
  color: #fff;
  font-size: 19px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(25,118,210,0.10);
  transition: background 0.2s, box-shadow 0.2s;
}
.send-bar button:hover {
  box-shadow: 0 4px 16px rgba(25,118,210,0.13);
  filter: brightness(1.08);
}
.log-viewer {
  width: 100%;
  height: 100%;
  flex: 1 1 0;
  overflow-y: auto;
  font-size: 16px;
  font-family: 'Fira Mono', 'Consolas', 'Menlo', monospace;
  background: #fafdff;
  border-radius: 8px;
  border: 1.5px solid #b6c8e6;
  padding: 12px;
  color: #222;
  box-shadow: 0 1px 4px rgba(25,118,210,0.03) inset;
  display: flex;
  flex-direction: column;
  gap: 2px;
  align-items: flex-start;
  text-align: left;
  min-width: 0;
  min-height: 0;
}
.log-line {
  white-space: pre-wrap;
  word-break: break-all;
  text-align: left;
}
.send-log {
  color: #219653;
  font-weight: bold;
}
.recv-log {
  color: #1976d2;
  font-weight: bold;
}
</style> 