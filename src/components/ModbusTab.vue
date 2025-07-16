<template>
  <div class="modbus-tab-layout">
    <div class="card"><ModbusCommandGenerator /></div>
    <div class="divider"></div>
    <div class="card"><ModbusCrcCalculator /></div>
    <div class="divider"></div>
    <div class="card"><ModbusResponseGenerator /></div>
    <div class="divider"></div>
    <div class="card"><ModbusMessageParser /></div>
  </div>
</template>

<script setup>
import ModbusCommandGenerator from './ModbusCommandGenerator.vue'
import ModbusCrcCalculator from './ModbusCrcCalculator.vue'
import ModbusResponseGenerator from './ModbusResponseGenerator.vue'
import ModbusMessageParser from './ModbusMessageParser.vue'
</script>

<style scoped>
.modbus-tab-layout {
  background: #f6faff;
  padding: 36px 32px 32px 32px;
  border-radius: 18px;
  min-width: 980px;
  box-shadow: 0 6px 32px rgba(25, 118, 210, 0.10);
  display: flex;
  flex-direction: column;
  gap: 28px;
}
.card {
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 2px 12px rgba(25,118,210,0.08);
  padding: 28px 28px 22px 28px;
  margin: 0 auto;
  max-width: 1100px;
  min-width: 600px;
}
.divider {
  height: 0;
  border-top: 2px dashed #e3eaf5;
  margin: 0 0 0 0;
}
/* 统一输入框和按钮风格，参照电总协议页面，使用:deep()确保作用于所有子组件 */
:deep(.card input),
:deep(.card select) {
  font-size: 16px;
  padding: 6px 10px;
  border-radius: 4px;
  border: 1px solid #d0d7de;
  box-sizing: border-box;
}
:deep(.card button),
:deep(.card .dz-form button) {
  padding: 10px 28px;
  font-size: 16px;
  border-radius: 4px;
  border: none;
  background: #1976d2;
  color: #fff;
  cursor: pointer;
  transition: background 0.2s;
}
:deep(.card button:hover),
:deep(.card .dz-form button:hover) {
  background: #1565c0;
}
:deep(.card label) {
  display: flex;
  flex-direction: column;
  font-size: 15px;
  min-width: 180px;
  margin-bottom: 8px;
}
:deep(.card .result-container) {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  margin-top: 8px;
}
:deep(.card .result) {
  flex: 1;
  font-family: monospace;
  background: #f6f8fa;
  padding: 10px 14px;
  border-radius: 4px;
  word-break: break-all;
}
:deep(.card .copy-btn) {
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
:deep(.card .copy-btn:hover) {
  background: #f0f7ff;
}
@media (max-width: 1200px) {
  .modbus-tab-layout {
    padding: 18px 6px 12px 6px;
    min-width: 0;
  }
  .card {
    max-width: 100%;
    min-width: 0;
    padding: 18px 8px 12px 8px;
  }
}
.icp-footer {
  text-align: center;
  color: #90a4ae;
  font-size: 14px;
  margin-top: 18px;
  margin-bottom: 2px;
  letter-spacing: 1px;
}
</style> 