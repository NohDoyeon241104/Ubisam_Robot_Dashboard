<script setup>
import HelloWorld from './components/HelloWorld.vue'

// SensorChart 컴포넌트 임포트 추가하기
import SensorChart from './components/SensorChart.vue'

// TurtleMap 컴포넌트 임포트 추가하기
// import TurtleMap from './components/TurtleMap.vue'
// TurtleMapKeyAdd 컴포넌트 임포트 추가하기
import TurtleMapKeyAdd from './components/TurtleMapKeyAdd.vue'
// FlowEditor 컴포넌트 임포트 추가하기
import FlowEditor from './components/FlowEditor.vue'

//  ProductRegistry 컴포넌트 임포트 추가하기 (장비 추가 및 노드 추가)
import ProductRegistry from './components/ProductRegistry.vue'



// ROS2 웹소켓 연결 및 데이터 수신을 위한 코드
import {ref, onMounted} from 'vue'
import {Ros,Topic} from 'roslib'

// 탭 상태 관리
const activeTab = ref('dashboard')
// 연결 상태와 수신된 메시지를 저장할 반응형 변수
const status = ref('연결 중...')
const message = ref('(없음)')
// 선택된 장비
const selectedProduct = ref(null)

onMounted(() =>{
  const ros = new Ros({
    url: 'ws://localhost:9090' // ROS2 웹소켓 서버 주소
  })

  // ROS2 연결 이벤트 핸들러
  ros.on('connection', () =>{status.value = '연결됨'})
  ros.on('error', () =>{status.value = '오류'})
  ros.on('close', () =>{status.value = '연결 끊김'})
  // ROS2 토픽 구독 설정
  const listener = new Topic({
    ros: ros,
    name: '/chatter', // 구독할 ROS2 토픽 이름
    messageType: 'std_msgs/String' // 토픽 메시지 타입
  })
  // 토픽 메시지 수신 이벤트 핸들러
  listener.subscribe((msg) => {
    message.value = msg.data
  })

})

//
function onProductSelect(product) {
  selectedProduct.value = product
  activeTab.value = 'flow'
}

</script>

<template>
  <div class="app">
    <div class="tab-bar">
      <span class="app-title">🤖 Ubisam Robot Dashboard</span>
      <button :class="['tab', activeTab === 'dashboard' && 'active']"
        @click="activeTab = 'dashboard'">📊 대시보드</button>
      <button :class="['tab', activeTab === 'registry' && 'active']"
        @click="activeTab = 'registry'">🔧 장비 관리</button>
      <button :class="['tab', activeTab === 'flow' && 'active']"
        @click="activeTab = 'flow'">
        🔀 플로우 에디터
        <span v-if="selectedProduct" class="selected-badge">
          {{ selectedProduct.product }}
        </span>
      </button>
      <span class="status">{{ status }}</span>
    </div>

    <div v-if="activeTab === 'dashboard'" class="tab-content">
      <p style="padding: 0 20px; color: #aaa;">마지막 메시지: {{ message }}</p>
      <SensorChart />
      <TurtleMap />
    </div>

    <div v-if="activeTab === 'registry'" class="tab-content">
      <ProductRegistry @select="onProductSelect" />
    </div>

    <div v-if="activeTab === 'flow'" class="tab-content flow-tab">
      <FlowEditor :product="selectedProduct" />
    </div>
  </div>
  <HelloWorld msg="Ubisam Robot Dashboard" />
</template>

<style scoped>
/* 추가 */
* { box-sizing: border-box; margin: 0; padding: 0; }
body { background: #0d1117; color: #fff; font-family: Arial, sans-serif; }
.app { display: flex; flex-direction: column; height: 100vh; }
.tab-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 16px;
  background: #161b22;
  border-bottom: 1px solid #333;
  height: 52px;
  flex-shrink: 0;
}
.app-title { color: #42b883; font-weight: bold; flex: 1; }
.tab {
  background: none;
  border: none;
  color: #888;
  padding: 6px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
}
.tab.active { background: #42b883; color: #fff; font-weight: bold; }
.status { color: #42b883; font-size: 13px; margin-left: 8px; }
.tab-content { flex: 1; overflow-y: auto; }
.flow-tab { overflow: hidden; }
/* 기존 */


.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
