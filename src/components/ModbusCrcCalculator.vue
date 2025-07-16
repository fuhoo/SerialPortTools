<template>
  <div>
    <h2>Modbus RTU CRC16 校验码计算器</h2>
    <form @submit.prevent="calcCrc">
      <label>输入报文（十六进制，用空格分隔）：<input v-model="inputHex" required /></label>
      <button type="submit">计算CRC</button>
    </form>
    <div v-if="crcResult">
      <h3>CRC16 校验码：</h3>
      <div class="result">{{ crcResult }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const inputHex = ref('01 03 00 00 00 02')
const crcResult = ref('')

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

function calcCrc() {
  const bytes = inputHex.value.split(/\s+/).map(h => parseInt(h, 16))
  const crc = crc16(bytes)
  const crcLo = crc & 0xFF
  const crcHi = (crc >> 8) & 0xFF
  crcResult.value = crcLo.toString(16).toUpperCase().padStart(2, '0') + ' ' + crcHi.toString(16).toUpperCase().padStart(2, '0')
}
</script>

<style scoped>
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