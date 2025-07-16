<template>
  <div>
    <h2>Modbus RTU报文解析器</h2>
    <div style="margin-bottom: 12px;">
      <textarea v-model="inputHex" rows="5" placeholder="输入数据，每字节用空格分隔，支持多行" style="width:100%;font-size:16px;padding:10px 12px;border-radius:6px;border:1px solid #d0d7de;"></textarea>
    </div>
    <div style="display:flex;gap:12px;margin-bottom:10px;">
      <button @click="parse" :disabled="!inputHex.trim()">解析</button>
      <button @click="clearAll" type="button">清除</button>
      <button @click="loadDemo" type="button">示例数据</button>
    </div>
    <div v-if="parsed" class="parse-summary">
      <span>从站地址: <b>{{ parsed.slave }}</b> ({{ parsed.slaveHex }})</span>
      <span>功能码: <b>{{ parsed.func }}</b> ({{ parsed.funcHex }})</span>
      <span>数据长度: <b>{{ parsed.dataLen }}</b> ({{ parsed.dataLenHex }})</span>
      <span>CRC校验: <b>{{ parsed.crcHex }}</b></span>
      <span>校验结果: <b :style="{color: parsed.crcOk ? '#43a047':'#e53935'}">{{ parsed.crcOk ? '正确' : '错误' }}</b></span>
    </div>
    <div v-if="parsed" style="margin:8px 0 18px 0;">
      <span style="color:#888">数据区HEX：</span>
      <span style="font-family:monospace;font-size:15px;background:#f6f8fa;padding:4px 10px;border-radius:4px;">{{ parsed.dataHex }}</span>
    </div>
    <div v-if="parsed">
      <h4 style="margin:18px 0 8px 0;">解析规则配置</h4>
      <table class="rule-table">
        <thead>
          <tr>
            <th>起始位置</th>
            <th>数据类型</th>
            <th>原始数据</th>
            <th>字节序</th>
            <th>排序数据</th>
            <th>解析结果</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(rule, idx) in rules" :key="idx">
            <td><input v-model.number="rule.offset" type="number" min="0" style="width:48px;"></td>
            <td>
              <select v-model="rule.type">
                <option value="uint16">UINT16 - 16位无符号</option>
                <option value="int16">INT16 - 16位有符号</option>
                <option value="uint32">UINT32 - 32位无符号</option>
                <option value="int32">INT32 - 32位有符号</option>
                <option value="float32">FLOAT32 - 32位浮点</option>
              </select>
            </td>
            <td>{{ getRawHex(rule) }}</td>
            <td>
              <select v-model="rule.order">
                <option value="ABCD">ABCD（大端序-高字节在前）</option>
                <option value="DCBA">DCBA（小端序-低字节在前）</option>
                <option value="BADC">BADC（字节交换）</option>
                <option value="CDAB">CDAB（字节交换）</option>
              </select>
            </td>
            <td>{{ getSortedHex(rule) }}</td>
            <td><b>{{ getParsedValue(rule) }}</b></td>
            <td><span style="color:#e53935;cursor:pointer;" @click="removeRule(idx)">删除</span></td>
          </tr>
        </tbody>
      </table>
      <button @click="addRule" type="button" style="margin-top:8px;">+ 添加规则</button>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'

const inputHex = ref('')
const parsed = ref(null)
const rules = reactive([
  { offset: 0, type: 'uint16', order: 'ABCD' }
])

function parse() {
  const arr = inputHex.value.trim().replace(/\s+/g, ' ').split(' ').map(h => parseInt(h, 16))
  if (arr.length < 6) {
    parsed.value = null
    return
  }
  const slave = arr[0]
  const func = arr[1]
  const dataLen = arr[2]
  const data = arr.slice(3, 3 + dataLen)
  const crc = arr.slice(-2)
  const crcCalc = calcCrc16(arr.slice(0, -2))
  const crcOk = crc[0] === (crcCalc & 0xFF) && crc[1] === ((crcCalc >> 8) & 0xFF)
  parsed.value = {
    slave,
    slaveHex: toHex(slave),
    func,
    funcHex: toHex(func),
    dataLen,
    dataLenHex: toHex(dataLen),
    data,
    dataHex: data.map(b => toHex(b)).join(' '),
    crcHex: crc.map(b => toHex(b)).join(' '),
    crcOk,
    rawArr: data
  }
}

function clearAll() {
  inputHex.value = ''
  parsed.value = null
  rules.splice(0, rules.length, { offset: 0, type: 'uint16', order: 'ABCD' })
}

function loadDemo() {
  inputHex.value = '01 03 04 01 14 03 15 7A F4'
  parse()
}

function addRule() {
  let offset = 0
  if (rules.length > 0) {
    const last = rules[rules.length - 1]
    offset = last.offset + getTypeLen(last.type)
  }
  rules.push({ offset, type: 'uint16', order: 'ABCD' })
}
function removeRule(idx) {
  if (rules.length > 1) rules.splice(idx, 1)
}

function getRawHex(rule) {
  if (!parsed.value) return ''
  const len = getTypeLen(rule.type)
  const arr = parsed.value.rawArr.slice(rule.offset, rule.offset + len)
  return arr.map(toHex).join(' ')
}
function getSortedHex(rule) {
  if (!parsed.value) return ''
  const len = getTypeLen(rule.type)
  let arr = parsed.value.rawArr.slice(rule.offset, rule.offset + len)
  arr = applyOrder(arr, rule.order)
  return arr.map(toHex).join(' ')
}
function getParsedValue(rule) {
  if (!parsed.value) return ''
  const len = getTypeLen(rule.type)
  let arr = parsed.value.rawArr.slice(rule.offset, rule.offset + len)
  arr = applyOrder(arr, rule.order)
  return parseByType(arr, rule.type)
}
function getTypeLen(type) {
  if (type === 'uint16' || type === 'int16') return 2
  return 4
}
function applyOrder(arr, order) {
  if (arr.length === 2) {
    if (order === 'ABCD') return arr
    if (order === 'DCBA') return arr.slice().reverse()
    if (order === 'BADC') return [arr[1], arr[0]]
    if (order === 'CDAB') return [arr[1], arr[0]]
  } else if (arr.length === 4) {
    if (order === 'ABCD') return arr
    if (order === 'DCBA') return arr.slice().reverse()
    if (order === 'BADC') return [arr[1], arr[0], arr[3], arr[2]]
    if (order === 'CDAB') return [arr[2], arr[3], arr[0], arr[1]]
  }
  return arr
}
function parseByType(arr, type) {
  if (arr.length === 2) {
    const val = (arr[0] << 8) | arr[1]
    if (type === 'uint16') return val
    if (type === 'int16') return val > 0x7FFF ? val - 0x10000 : val
  } else if (arr.length === 4) {
    const val = (arr[0] << 24) | (arr[1] << 16) | (arr[2] << 8) | arr[3]
    if (type === 'uint32') return val >>> 0
    if (type === 'int32') return val > 0x7FFFFFFF ? val - 0x100000000 : val
    if (type === 'float32') {
      const buf = new ArrayBuffer(4)
      const view = new DataView(buf)
      arr.forEach((b, i) => view.setUint8(i, b))
      return view.getFloat32(0)
    }
  }
  return ''
}
function toHex(num) {
  return num.toString(16).toUpperCase().padStart(2, '0')
}
function calcCrc16(buffer) {
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
</script>

<style scoped>
.parse-summary {
  display: flex;
  gap: 24px;
  background: #f6f8fa;
  border-radius: 6px;
  padding: 10px 18px;
  font-size: 15px;
  margin-bottom: 4px;
  flex-wrap: wrap;
}
.rule-table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 8px;
  background: #fff;
  box-shadow: 0 1px 6px rgba(25,118,210,0.06);
  border-radius: 8px;
  overflow: hidden;
}
.rule-table th, .rule-table td {
  border: 1px solid #e3eaf5;
  padding: 6px 10px;
  text-align: center;
  font-size: 15px;
}
.rule-table th {
  background: #f6faff;
  font-weight: 500;
}
.rule-table td {
  background: #fff;
}
.rule-table input, .rule-table select {
  font-size: 15px;
  padding: 2px 6px;
  border-radius: 4px;
  border: 1px solid #d0d7de;
}
button {
  padding: 8px 22px;
  font-size: 15px;
  border-radius: 4px;
  border: none;
  background: #1976d2;
  color: #fff;
  cursor: pointer;
  transition: background 0.2s;
}
button:hover {
  background: #1565c0;
}
</style> 