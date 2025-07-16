<template>
  <div>
    <h2>Modbus 命令生成器</h2>
    <div class="proto-group">
      <label v-for="p in protocols" :key="p.value" class="proto-radio">
        <input type="radio" v-model="protocol" :value="p.value" /> {{ p.label }}
      </label>
    </div>
    <form @submit.prevent="generateCommand">
      <label>从站地址：<input v-model="slave" type="number" min="0" max="255" required /></label>
      <label>功能码：
        <select v-model="func" required>
          <option value="1">01 - 读线圈</option>
          <option value="2">02 - 读离散输入</option>
          <option value="3">03 - 读保持寄存器</option>
          <option value="4">04 - 读输入寄存器</option>
          <option value="5">05 - 写线圈</option>
          <option value="6">06 - 写保持寄存器</option>
          <option value="15">0F - 写多个线圈</option>
          <option value="16">10 - 写多个保持寄存器</option>
        </select>
      </label>
      <label>起始地址：<input v-model="startAddr" type="number" min="0" max="65535" required /></label>
      <label v-if="func != 5 && func != 6">数量：<input v-model="quantity" type="number" min="1" :max="func == 15 || func == 16 ? 123 : 125" required /></label>
      <template v-if="func == 5">
        <label>写入值（0=关，1=开）：<input v-model="writeValue" type="number" min="0" max="1" required /></label>
      </template>
      <template v-else-if="func == 6">
        <label>写入值（0~65535）：<input v-model="writeValue" type="number" min="0" max="65535" required /></label>
      </template>
      <template v-else-if="func == 15">
        <label>写入线圈值（0/1，用空格或逗号分隔）：<input v-model="writeMulti" placeholder="如 1 0 1 1" required /></label>
      </template>
      <template v-else-if="func == 16">
        <label>写入寄存器值（0~65535，用空格或逗号分隔）：<input v-model="writeMulti" placeholder="如 123 456 789" required /></label>
      </template>
      <button type="submit">生成命令</button>
    </form>
    <div v-if="result.rtu">
      <h3>Modbus RTU 命令：</h3>
      <div class="result">{{ result.rtu }}</div>
    </div>
    <div v-if="result.ascii">
      <h3>Modbus ASCII 命令：</h3>
      <div class="result">{{ result.ascii }}</div>
    </div>
    <div v-if="result.tcp">
      <h3>Modbus TCP 命令：</h3>
      <div class="result">{{ result.tcp }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const protocols = [
  { value: 'rtu', label: 'Modbus RTU' },
  { value: 'ascii', label: 'Modbus ASCII' },
  { value: 'tcp', label: 'Modbus TCP' }
]
const protocol = ref('rtu')
const slave = ref(1)
const func = ref(3)
const startAddr = ref(0)
const quantity = ref(2)
const command = ref('')
const writeValue = ref('')
const writeMulti = ref('')
const result = ref({ rtu: '', ascii: '', tcp: '' })

function toHex(num, len = 2) {
  return num.toString(16).toUpperCase().padStart(len, '0')
}

function crc16(buffer) {
  let crc = 0xFFFF
  for (let pos = 0; pos < buffer.length; pos++) {
    crc ^= buffer[pos]
    for (let i = 0; i < 8; i++) {
      if ((crc & 0x0001) !== 0) {
        crc >>= 1
        crc ^= 0xA001
      } else {
        crc >>= 1
      }
    }
  }
  return crc
}

function lrc(bytes) {
  // LRC校验，所有字节相加取反加1，低字节
  let sum = bytes.reduce((a, b) => a + b, 0)
  return ((~sum + 1) & 0xFF)
}

function buildPDU() {
  let bytes = [slave.value, Number(func.value), startAddr.value >> 8, startAddr.value & 0xFF]
  if (func.value == 5) {
    let val = Number(writeValue.value) ? 0xFF00 : 0x0000
    bytes.push(val >> 8, val & 0xFF)
  } else if (func.value == 6) {
    let val = Number(writeValue.value)
    bytes.push(val >> 8, val & 0xFF)
  } else if (func.value == 15) {
    let n = Number(quantity.value)
    let arr = writeMulti.value.split(/[\s,]+/).filter(Boolean).map(Number)
    if (arr.length !== n) return null
    bytes.push(n >> 8, n & 0xFF)
    let byteCount = Math.ceil(n / 8)
    bytes.push(byteCount)
    for (let i = 0; i < byteCount; i++) {
      let b = 0
      for (let j = 0; j < 8; j++) {
        let idx = i * 8 + j
        if (idx < arr.length && arr[idx]) b |= (1 << j)
      }
      bytes.push(b)
    }
  } else if (func.value == 16) {
    let n = Number(quantity.value)
    let arr = writeMulti.value.split(/[\s,]+/).filter(Boolean).map(Number)
    if (arr.length !== n) return null
    bytes.push(n >> 8, n & 0xFF)
    let byteCount = n * 2
    bytes.push(byteCount)
    for (let i = 0; i < n; i++) {
      let v = arr[i]
      bytes.push((v >> 8) & 0xFF, v & 0xFF)
    }
  } else {
    bytes.push(quantity.value >> 8, quantity.value & 0xFF)
  }
  return bytes
}

function generateCommand() {
  const pdu = buildPDU()
  if (!pdu) {
    result.value = { rtu: '', ascii: '', tcp: '' }
    command.value = '写入值数量与数量参数不符！'
    return
  }
  // RTU
  if (protocol.value === 'rtu') {
    const crc = crc16(pdu)
    const crcLo = crc & 0xFF
    const crcHi = (crc >> 8) & 0xFF
    result.value.rtu = pdu.map(b => toHex(b)).join(' ') + ' ' + toHex(crcLo) + ' ' + toHex(crcHi)
    result.value.ascii = ''
    result.value.tcp = ''
  } else if (protocol.value === 'ascii') {
    // Modbus ASCII: : + 数据区(每字节2位HEX) + LRC + CRLF
    const lrcVal = lrc(pdu)
    const asciiData = pdu.map(b => toHex(b)).join('') + toHex(lrcVal)
    result.value.ascii = ':' + asciiData + '\r\n'
    result.value.rtu = ''
    result.value.tcp = ''
  } else if (protocol.value === 'tcp') {
    // Modbus TCP: MBAP(7字节) + PDU
    // MBAP: 事务ID(2) 协议ID(2) 长度(2) 单元标识符(1)
    const tid = 1, pid = 0, uid = slave.value
    const len = pdu.length + 1 // PDU+UID
    const mbap = [tid >> 8, tid & 0xFF, pid >> 8, pid & 0xFF, len >> 8, len & 0xFF, uid]
    const tcpFrame = [...mbap, ...pdu.slice(1)] // pdu[0]是UID，已在mbap
    result.value.tcp = tcpFrame.map(b => toHex(b)).join(' ')
    result.value.rtu = ''
    result.value.ascii = ''
  }
}
</script>

<style scoped>
.proto-group {
  display: flex;
  gap: 24px;
  margin-bottom: 18px;
  margin-top: 8px;
}
.proto-radio {
  font-size: 15px;
  color: #1976d2;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 4px;
}
form {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 16px;
}
label {
  display: flex;
  flex-direction: column;
  font-size: 14px;
}
.result {
  font-family: monospace;
  background: #f6f8fa;
  padding: 8px 12px;
  border-radius: 4px;
  margin-top: 8px;
}
</style> 