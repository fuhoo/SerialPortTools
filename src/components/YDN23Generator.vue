<template>
  <div>
    <h2>电总协议命令生成器</h2>
    <form @submit.prevent="generateCmd" class="dz-form">
      <label>协议版本：<input v-model="version" placeholder="如 33H" required /></label>
      <label>地址：<input v-model="addr" placeholder="如 000000000001" required /></label>
      <label>CID1：<input v-model="cid1" placeholder="如 01H" required /></label>
      <label>CID2：<input v-model="cid2" placeholder="如 02H" required /></label>
      <label>数据（HEX，可空）：<input v-model="data" placeholder="如 01 02 03" /></label>
      <button type="submit">生成命令</button>
    </form>
    <div v-if="result.ascii">
      <h3>ASCII命令：</h3>
      <div class="result-container">
        <div class="result">{{ result.ascii }}</div>
        <button @click="copyToClipboard(result.ascii)" class="copy-btn">复制</button>
      </div>
    </div>
    <div v-if="result.hex">
      <h3>HEX命令：</h3>
      <div class="result-container">
        <div class="result">{{ result.hex }}</div>
        <button @click="copyToClipboard(result.hex)" class="copy-btn">复制</button>
      </div>
    </div>
    <div class="dz-divider"></div>
    <div class="dz-card">
      <h2>电总协议报文解析器</h2>
      <form @submit.prevent="parseFrame" class="dz-form">
        <label>输入报文（HEX）：
         <textarea v-model="parseInput" placeholder="如 7E 33 01 60 42 ..." required rows="3" class="dz-textarea" />
        </label>
        <button type="submit">解析</button>
      </form>
      <div v-if="parseResult">
        <h3>解析结果：</h3>
        <div class="result">
          <div v-for="(v, k) in parseResult" :key="k" v-if="k !== 'ASC码'">{{ k }}: {{ v }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const version = ref('21')
const addr = ref('01')
const cid1 = ref('60')
const cid2 = ref('42')
const data = ref('')
const result = ref({ ascii: '', hex: '' })
const parseInput = ref('')
const parseResult = ref(null)

function copyToClipboard(text) {
  navigator.clipboard.writeText(text).then(() => {
    alert('已复制到剪贴板！')
  }).catch(err => {
    console.error('复制失败:', err)
    alert('复制失败，请手动复制')
  })
}

function toAsciiHex(byte) {
  // 1字节转2个ASCII码
  const hex = byte.toString(16).toUpperCase().padStart(2, '0')
  return [hex.charCodeAt(0), hex.charCodeAt(1)]
}

function hexStrToAsciiBytes(str) {
  // HEX字符串转ASCII码数组（每字节2个ASCII码）
  return str.split(/\s+/).filter(Boolean).flatMap(h => toAsciiHex(parseInt(h, 16)))
}

function calcLenLchksum(len) {
  // LENGTH共2个字节：LENID(高字节) + LCHKSUM(低字节)
  // LENID表示INFO的ASCII字节数
  // 传输时先传高字节，再传低字节，分四个ASCII码传送
  
  // 将len转为12位二进制，分成3组计算校验码
  const lenBin = len.toString(2).padStart(12, '0')
  const d11_8 = parseInt(lenBin.substr(0, 4), 2)  // D11D10D9D8
  const d7_4 = parseInt(lenBin.substr(4, 4), 2)   // D7D6D5D4  
  const d3_0 = parseInt(lenBin.substr(8, 4), 2)   // D3D2D1D0
  
  // 校验码计算：求和后模16余数取反加1
  let sum = d11_8 + d7_4 + d3_0
  let lchksum = (~(sum % 16) + 1) & 0xF
  
  // 组合LENGTH：LCHKSUM(4位) + LENID(12位) = 16位
  const length = (lchksum << 12) | len
  
  // 转为4个ASCII码：高字节2位 + 低字节2位
  const hi = ((length >> 8) & 0xFF).toString(16).toUpperCase().padStart(2, '0')
  const lo = (length & 0xFF).toString(16).toUpperCase().padStart(2, '0')
  
  return [hi.charCodeAt(0), hi.charCodeAt(1), lo.charCodeAt(0), lo.charCodeAt(1)]
}

function calcChksum(asciiArr) {
  let sum = asciiArr.reduce((a, b) => a + b, 0)

  let v = (~sum + 1) & 0xFFFF
  console.log(v)
  // 高字节、低字节分别转2位ASCII
  const hi = ((v >> 8) & 0xFF).toString(16).toUpperCase().padStart(2, '0')
  const lo = (v & 0xFF).toString(16).toUpperCase().padStart(2, '0')
  return [hi.charCodeAt(0), hi.charCodeAt(1), lo.charCodeAt(0), lo.charCodeAt(1)]
}

function buildFrame(infoArr) {
  // SOI(7E) + VER + ADR + CID1 + CID2 + LENGTH + INFO + CHKSUM + EOI(0D)
  const SOI = 0x7E, EOI = 0x0D
  const verArr = toAsciiHex(parseInt(version.value, 16))
  const addrArr = addr.value.padStart(2, '0').match(/.{1,2}/g).map(h => toAsciiHex(parseInt(h, 16))).flat()
  const cid1Arr = toAsciiHex(parseInt(cid1.value, 16))
  const cid2Arr = toAsciiHex(parseInt(cid2.value, 16))
  const lenArr = calcLenLchksum(infoArr.length)
  console.log("lenArr",lenArr)
  // 拼接除SOI/EOI/CHKSUM外所有ASCII码
  const frameArr = [...verArr, ...addrArr, ...cid1Arr, ...cid2Arr, ...lenArr,...infoArr]
 
  // 计算CHKSUM（不含SOI/EOI/CHKSUM）
  const chksumArr = calcChksum(frameArr)
  // 最终帧
  return [SOI, ...frameArr, ...chksumArr, EOI]
}

function arrToAsciiStr(arr) {
  return arr.map(b => String.fromCharCode(b)).join('')
}
function arrToHexStr(arr) {
  return arr.map(b => b.toString(16).toUpperCase().padStart(2, '0')).join(' ')
}

function generateCmd() {
 
  // INFO区始终按HEX字节流处理
  let infoAscii = hexStrToAsciiBytes(data.value)
  const asciiFrame = buildFrame(infoAscii)
  result.value.ascii = arrToAsciiStr(asciiFrame)
  result.value.hex = arrToHexStr(asciiFrame)
}

function parseFrame() {
  // 先将HEX流转为字节数组
  const arr = parseInput.value.trim().split(/\s+/).map(h => parseInt(h, 16))
  if (arr.length < 18 || arr[0] !== 0x7E || arr[arr.length-1] !== 0x0D) {
    parseResult.value = { 错误: '报文格式不正确，需以7E开头0D结尾' }
    return
  }
  // 转为ASCII字符串
  let asciiStr = ''
  for (let i = 1; i < arr.length - 1; i += 2) {
    asciiStr += String.fromCharCode(arr[i]) + String.fromCharCode(arr[i+1])
  }
  // 分段解析
  // SOI(1) VER(2) ADR(2) CID1(2) RTN(2) LENGTH(4) INFO(?) CHKSUM(4) EOI(1)
  const ver = asciiStr.slice(0, 2)
  const adr = asciiStr.slice(2, 4)
  const cid1 = asciiStr.slice(4, 6)
  const rtn = asciiStr.slice(6, 8)
  const lengthStr = asciiStr.slice(8, 12)
  const lengthVal = parseInt(lengthStr, 16)
  const lenid = lengthVal & 0x0FFF
  const lchksum = (lengthVal >> 12) & 0xF
  // 校验LCHKSUM算法
  const lenidBin = lenid.toString(2).padStart(12, '0')
  const d11_8 = parseInt(lenidBin.substr(0, 4), 2)
  const d7_4 = parseInt(lenidBin.substr(4, 4), 2)
  const d3_0 = parseInt(lenidBin.substr(8, 4), 2)
  let sum = d11_8 + d7_4 + d3_0
  let lchksumCalc = (~(sum % 16) + 1) & 0xF
  const lchksumOk = lchksum === lchksumCalc
  // 用LENID反推INFO区长度
  const infoLen = lenid
  const info = infoLen > 0 ? asciiStr.slice(12, 12 + infoLen) : ''
  const chksum = asciiStr.slice(12 + infoLen, 12 + infoLen + 4)

  // 用已有的calcChksum函数计算校验
  // 计算范围：VER+ADR+CID1+RTN+LENGTH+INFO（ASCII码数组）
  // arr: [SOI, VER1, VER2, ADR1, ADR2, CID1_1, CID1_2, ...]
  // 取1~(len-1-4)（去掉SOI、CHKSUM、EOI）
  const calcStart = 1;
  const calcEnd = arr.length - 1 - 4;
  const asciiArr = arr.slice(calcStart, calcEnd)
  const calcArr = [...asciiArr]
  const calcChksumArr = calcChksum(calcArr)
  const calcChksumAscii = String.fromCharCode(...calcChksumArr)
  const isChksumOk = chksum === calcChksumAscii

  parseResult.value = {
    '协议版本': ver,
    '地址': adr,
    'CID1': cid1,
    'RTN': rtn,
    'LENGTH': lengthStr,
    'LENID(ASCII字节数)': lenid,
    'LCHKSUM': lchksum.toString(16).toUpperCase(),
    'LCHKSUM校验': lchksumOk ? '✔ 正确' : '✘ 错误',
    'INFO区': info,
    'CHKSUM': chksum,
    '校验结果': isChksumOk ? '✔ 正确' : '✘ 错误',
    ...(isChksumOk ? {} : { '本地计算校验': calcChksumAscii })
  }
}
</script>

<style scoped>
.dz-form {
  display: flex;
  flex-wrap: wrap;
  gap: 18px;
  margin-bottom: 18px;
}
.dz-form label {
  display: flex;
  flex-direction: column;
  font-size: 15px;
  min-width: 0;
  width: 100%;
}
.dz-form input, .dz-form select {
  font-size: 16px;
  padding: 6px 10px;
  border-radius: 4px;
  border: 1px solid #d0d7de;
}
.dz-form button {
  padding: 10px 28px;
  font-size: 16px;
  border-radius: 4px;
  border: none;
  background: #1976d2;
  color: #fff;
  cursor: pointer;
}
.result-container {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  margin-top: 8px;
}
.result {
  flex: 1;
  font-family: monospace;
  background: #f6f8fa;
  padding: 10px 14px;
  border-radius: 4px;
  word-break: break-all;
}
.copy-btn {
  padding: 8px 16px;
  font-size: 14px;
  border-radius: 4px;
  border: 1px solid #d0d7de;
  background: #fff;
  color: #1976d2;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.2s;
}
.copy-btn:hover {
  background: #f0f7ff;
}
.dz-divider {
  height: 0;
  border-top: 2px dashed #e3eaf5;
  margin: 32px 0 32px 0;
}
.dz-card {
  background: #fff;
  border-radius: 14px;
  box-shadow: 0 2px 10px rgba(25,118,210,0.07);
  padding: 28px 0 18px 0;
  margin: 0 auto;
  max-width: 1200px;
}
.dz-textarea {
  font-size: 18px;
  min-height: 60px;
  padding: 10px 24px;
  border-radius: 4px;
  border: 1px solid #d0d7de;
  resize: vertical;
  font-family: 'Fira Mono', 'Consolas', 'Menlo', monospace;
  width: 100%;
  box-sizing: border-box;
  margin: 0;
}
</style> 