<script setup>
import { ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import SerialDebugger from './components/SerialDebugger.vue'
import ModbusTab from './components/ModbusTab.vue'
import YDN23Generator from './components/YDN23Generator.vue'
import Disclaimer from './components/Disclaimer.vue'

const tabs = [
  { key: 'serial', label: '串口调试' },
  { key: 'modbus', label: 'Modbus' },
  { key: 'ydn23', label: '电总协议' }
]
const currentTab = ref('serial')
const tabComponentMap = {
  serial: SerialDebugger,
  modbus: ModbusTab,
  ydn23: YDN23Generator
}

const route = useRoute()
const router = useRouter()

// 监听路由变化，自动切换 tab
watch(
  () => route.params.tab,
  (tab) => {
    if (tabs.some(t => t.key === tab)) {
      currentTab.value = tab
    } else {
      currentTab.value = 'serial'
    }
  },
  { immediate: true }
)

// 切换 tab 时跳转路由
function switchTab(tab) {
  currentTab.value = tab
  router.push({ path: '/' + tab })
}

// 控制免责声明弹窗显示
const showDisclaimer = ref(false)
function goDisclaimer() {
  showDisclaimer.value = true
}
function closeDisclaimer() {
  showDisclaimer.value = false
}
</script>

<template>
  <div id="app">
    <div class="tab-bar">
      <button v-for="tab in tabs" :key="tab.key" :class="{active: currentTab === tab.key}" @click="switchTab(tab.key)">
        {{ tab.label }}
      </button>
    </div>
    <div class="container">
      <keep-alive>
        <component :is="tabComponentMap[currentTab]" class="card" />
      </keep-alive>
      <div v-if="showDisclaimer" class="disclaimer-modal">
        <div class="disclaimer-modal-content">
          <button class="disclaimer-close" @click="closeDisclaimer">×</button>
          <Disclaimer />
        </div>
        <div class="disclaimer-mask" @click="closeDisclaimer"></div>
      </div>
    </div>
    <footer class="icp-global-footer">
      <a href="https://beian.miit.gov.cn" target="_blank" rel="noopener" class="icp-link">粤ICP备2025443986号-1</a>
      <span class="footer-sep">|</span>
      <a href="javascript:void(0)" @click="goDisclaimer" class="icp-link">免责声明</a>
    </footer>
  </div>
</template>

<style scoped>
.bg-full {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  width: 100vw;
  height: 100vh;
  background: linear-gradient(120deg, #e0e7ff 0%, #f4f6fa 100%);
  z-index: 0;
  overflow-y: auto;
}
.tab-bar {
  display: flex;
  justify-content: center;
  gap: 32px;
  background: #e3f0ff;
  padding: 18px 0 8px 0;
  box-shadow: 0 2px 8px rgba(25,118,210,0.04);
  margin-bottom: 18px;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  z-index: 1000;
  min-height: 58px;
}
.tab-bar button {
  background: none;
  border: none;
  font-size: 20px;
  color: #1976d2;
  font-weight: bold;
  padding: 8px 32px;
  border-radius: 0;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
}
.tab-bar button.active {
  background: #fff;
  color: #1565c0;
  box-shadow: 0 2px 8px rgba(25,118,210,0.06);
}
.container {
  max-width: 1400px;
  margin: 0 auto;
  min-height: 600px;
  padding-top: 12px;
}
.card {
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.10);
  padding: 40px 48px;
  margin: 0 auto;
  width: 100%;
  max-width: 1200px;
}
.divider {
  height: 40px;
}

/* 让表单和字体更大 */
.card form {
  font-size: 18px;
}
.card input, .card button {
  font-size: 18px;
  padding: 10px 16px;
}
.card h2 {
  font-size: 2rem;
  margin-bottom: 24px;
}
.card h3 {
  font-size: 1.2rem;
  margin-top: 20px;
}
.card.placeholder {
  text-align: center;
  color: #888;
  min-height: 300px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.icp-global-footer {
  width: 100%;
  text-align: center;
  color: #90a4ae;
  font-size: 14px;
  margin-top: 32px;
  margin-bottom: 8px;
  letter-spacing: 1px;
}
.icp-link {
  color: #90a4ae;
  text-decoration: none;
  transition: color 0.2s;
  margin: 0 4px;
}
.icp-link:hover {
  color: #1976d2;
  text-decoration: underline;
}
.footer-sep {
  color: #b0bec5;
  margin: 0 6px;
}
.disclaimer-modal {
  position: fixed;
  left: 0; top: 0; right: 0; bottom: 0;
  z-index: 2000;
  display: flex;
  align-items: center;
  justify-content: center;
}
.disclaimer-modal-content::-webkit-scrollbar {
  width: 10px;
  background: #f3f6fa;
  border-radius: 8px;
}
.disclaimer-modal-content::-webkit-scrollbar-thumb {
  background: #b6c8e6;
  border-radius: 8px;
  min-height: 40px;
}
.disclaimer-modal-content::-webkit-scrollbar-thumb:hover {
  background: #1976d2;
}
.disclaimer-modal-content {
  scrollbar-width: thin;
  scrollbar-color: #b6c8e6 #f3f6fa;
}
.disclaimer-modal-content {
  position: relative;
  z-index: 2001;
  background: none;
  max-width: 1300px;
  min-width: 900px;
  width: 100%;
  max-height: 95vh;
  overflow-y: auto;
  box-shadow: 0 8px 32px rgba(25,118,210,0.18);
  border-radius: 18px;
  padding: 0;
}
.disclaimer-close {
  position: absolute;
  top: 18px;
  right: 28px;
  font-size: 2.2rem;
  color: #1976d2;
  background: none;
  border: none;
  cursor: pointer;
  z-index: 10;
  font-weight: bold;
  line-height: 1;
}
.disclaimer-mask {
  position: fixed;
  left: 0; top: 0; right: 0; bottom: 0;
  background: rgba(33, 33, 33, 0.18);
  z-index: 2000;
}
</style>
