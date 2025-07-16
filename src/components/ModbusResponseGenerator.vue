<template>
  <div>
    <h2>Modbus RTU 响应报文生成器</h2>
    <form @submit.prevent="generateResponse">
      <label>从站地址：<input v-model="slave" type="number" min="0" max="255" required /></label>
      <label>功能码：<input v-model="func" type="number" min="1" max="127" required /></label>
      <label>数据（十六进制，用空格分隔）：<input v-model="dataHex" required /></label>
      <button type="submit">生成响应报文</button>
    </form>
    <div v-if="response">
      <h3>生成的响应报文：</h3>
      <div class="result">{{ response }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const slave = ref(1)
const func = ref(3)
const dataHex = ref('00 01 00 02')
const response = ref('')

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

function generateResponse() {
  const dataArr = dataHex.value.split(/\s+/).map(h => parseInt(h, 16))
  const bytes = [slave.value, func.value, dataArr.length, ...dataArr]
  const crc = crc16(bytes)
  const crcLo = crc & 0xFF
  const crcHi = (crc >> 8) & 0xFF
  response.value = bytes.map(b => toHex(b)).join(' ') + ' ' + toHex(crcLo) + ' ' + toHex(crcHi)
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