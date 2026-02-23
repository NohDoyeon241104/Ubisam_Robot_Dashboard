<!-- 장비 등록 Vue 컴포넌트 추가 -->
<script setup>
import { ref, onMounted } from 'vue'
import { Ros, Service } from 'roslib'

// Rosbridge 연결
const ros = new Ros({ url: 'ws://localhost:9090' })

const emit = defineEmits(['select'])

const products = ref([])
const showForm = ref(false)
const editingProduct = ref(null)


// 토픽 목록
const topicList = ref([])
const loadingTopics = ref(false)

// 새 장비 폼 기본 구조
const emptyProduct = () => ({
  product: '',
  connection: {
    host: '192.168.1.1',
    port: 9090
  },
  nodes: []
})

const emptyNode = () => ({
  type: '',
  label: '',
  color: '#42b883',
  topic: '',
  messageType: '',
  fields: {}
})

const form = ref(emptyProduct())
const newNode = ref(emptyNode())
const newFieldKey = ref('')
const newFieldType = ref('float')

// localStorage에서 불러오기
onMounted(() => {
  const saved = localStorage.getItem('ros2-products')
  if (saved) {
    products.value = JSON.parse(saved)
  } else {
    // 기본 Jazzy Demo 장비 자동 등록
    const defaultProduct = {
      product: 'ROS2 Jazzy Demo',
      nodes: [
        { type: 'talker', label: '📢 Talker', color: '#42b883',
          topic: '/chatter', messageType: 'std_msgs/msg/String',
          fields: { data: { type: 'string', default: 'Hello from Flow!' } } },
        { type: 'turtle_move', label: '🐢 거북이 이동', color: '#2E75B6',
          topic: '/turtle1/cmd_vel', messageType: 'geometry_msgs/msg/Twist',
          fields: {
            'linear.x':  { type: 'float', default: 1.0, min: -2.0, max: 2.0 },
            'angular.z': { type: 'float', default: 0.0, min: -2.0, max: 2.0 }
          } },
        { type: 'turtle_rotate', label: '🔄 거북이 회전', color: '#9C27B0',
          topic: '/turtle1/cmd_vel', messageType: 'geometry_msgs/msg/Twist',
          fields: { 'angular.z': { type: 'float', default: 1.5, min: -3.0, max: 3.0 } } },
        { type: 'turtle_stop', label: '⛔ 정지', color: '#f44336',
          topic: '/turtle1/cmd_vel', messageType: 'geometry_msgs/msg/Twist',
          fields: {} },
        { type: 'wait', label: '⏱ 대기', color: '#FF9800',
          topic: null, messageType: null,
          fields: { duration: { type: 'float', default: 1.0, min: 0.1, max: 10.0, unit: '초' } } }
      ]
    }
    products.value = [defaultProduct]
    save()
  }
})

function save() {
  localStorage.setItem('ros2-products', JSON.stringify(products.value))
}

// 노드에 필드 추가
function addField() {
  if (!newFieldKey.value) return
  newNode.value.fields[newFieldKey.value] = {
    type: newFieldType.value,
    default: newFieldType.value === 'float' ? 0.0 : ''
  }
  newFieldKey.value = ''
}

function removeField(key) {
  delete newNode.value.fields[key]
  newNode.value = { ...newNode.value }
}

// 장비에 노드 추가
function addNodeToForm() {
  if (!newNode.value.type || !newNode.value.label) return
  form.value.nodes.push({ ...newNode.value, fields: { ...newNode.value.fields } })
  newNode.value = emptyNode()
}

function removeNode(index) {
  form.value.nodes.splice(index, 1)
}

// 장비 저장
function saveProduct() {
  if (!form.value.product) return
  if (editingProduct.value !== null) {
    products.value[editingProduct.value] = { ...form.value }
  } else {
    products.value.push({ ...form.value })
  }
  save()
  cancelForm()
}

// 장비 편집
function editProduct(index) {
  editingProduct.value = index
  form.value = JSON.parse(JSON.stringify(products.value[index]))
  showForm.value = true
}

// 장비 삭제
function deleteProduct(index) {
  products.value.splice(index, 1)
  save()
}

function cancelForm() {
  showForm.value = false
  editingProduct.value = null
  form.value = emptyProduct()
  newNode.value = emptyNode()
}

function selectProduct(product) {
  emit('select', product)
}

// 추가 함수
// 토픽 목록 조회
function fetchTopics() {
  loadingTopics.value = true
  const service = new Service({
    ros,
    name: '/rosapi/topics',
    serviceType: 'rosapi/Topics'
  })
  service.callService({}, (result) => {
    topicList.value = result.topics.sort()
    loadingTopics.value = false
  })
}

// 토픽 선택 시 messageType 자동 조회
function onTopicSelect(topic) {
  newNode.value.topic = topic
  if (!topic) return

  const service = new Service({
    ros,
    name: '/rosapi/topic_type',
    serviceType: 'rosapi/TopicType'
  })
  service.callService({ topic }, (result) => {
    newNode.value.messageType = result.type
  })
}

//연결 테스트 함수 추가 
const connectionStatus = ref('')

function testConnection() {
  connectionStatus.value = '⏳ 연결 중...'
  const testRos = new Ros({
    url: `ws://${form.value.connection.host}:${form.value.connection.port}`
  })
  testRos.on('connection', () => {
    connectionStatus.value = '✅ 연결 성공!'
    testRos.close()
  })
  testRos.on('error', () => {
    connectionStatus.value = '❌ 연결 실패 — IP/포트를 확인하세요'
  })
  setTimeout(() => {
    if (connectionStatus.value === '⏳ 연결 중...') {
      connectionStatus.value = '⏰ 연결 시간 초과'
    }
  }, 5000)
}

</script>

<template>
  <div class="registry">
    <div class="registry-header">
      <h2>🔧 장비 등록 관리</h2>
      <button class="btn-add" @click="showForm = true">+ 장비 등록</button>
    </div>

    <!-- 장비 목록 -->
    <div class="product-list">
      <div v-if="products.length === 0" class="empty">
        등록된 장비가 없습니다
      </div>
      <div v-for="(product, index) in products" :key="index" class="product-card">
        <div class="product-info">
          <span class="product-name">📦 {{ product.product }}</span>
          <span class="node-count">노드 {{ product.nodes.length }}개</span>
          <span class="product-ip">
            🔌 {{ product.connection?.host }}:{{ product.connection?.port }}
          </span>
        </div>
        <div class="node-tags">
          <span v-for="node in product.nodes" :key="node.type"
            class="node-tag" :style="{ background: node.color }">
            {{ node.label }}
          </span>
        </div>
        <div class="product-actions">
          <button class="btn-select" @click="selectProduct(product)">
            🔀 플로우에서 사용
          </button>
          <button class="btn-edit" @click="editProduct(index)">✏️ 편집</button>
          <button class="btn-delete" @click="deleteProduct(index)">🗑️</button>
        </div>
      </div>
    </div>

    <!-- 장비 등록/편집 폼 -->
    <div v-if="showForm" class="modal-overlay">
      <div class="modal">
        <div class="modal-header">
          <h3>{{ editingProduct !== null ? '장비 편집' : '새 장비 등록' }}</h3>
          <button @click="cancelForm">✕</button>
        </div>

        <!-- 장비명 -->
        <div class="form-section">
          <label>장비명</label>
          <input v-model="form.product" placeholder="예: RoboArm-X1" />
        </div>
        <!-- 연결 설정 추가 -->
        <div class="form-section">
        <label>연결 설정</label>
        <div class="connection-row">
            <input
            v-model="form.connection.host"
            placeholder="IP 주소 (예: 192.168.1.101)"
            style="flex: 2"
            />
            <span style="color: #888; padding: 0 6px;">:</span>
            <input
            v-model.number="form.connection.port"
            placeholder="포트"
            type="number"
            style="flex: 1"
            />
            <button class="btn-test" @click="testConnection">🔌 테스트</button>
        </div>
        <div v-if="connectionStatus" class="connection-status">
            {{ connectionStatus }}
        </div>
        </div>



        <!-- 등록된 노드 목록 -->
        <div class="form-section">
          <label>노드 목록 ({{ form.nodes.length }}개)</label>
          <div v-for="(node, i) in form.nodes" :key="i" class="added-node">
            <span :style="{ color: node.color }">{{ node.label }}</span>
            <span class="node-topic">{{ node.topic || '토픽 없음' }}</span>
            <button @click="removeNode(i)">✕</button>
          </div>
        </div>

        <!-- 노드 추가 -->
        <div class="form-section node-add-section">
          <label>노드 추가</label>
          <div class="node-form">
            <input v-model="newNode.type" placeholder="type (예: move)" />
            <input v-model="newNode.label" placeholder="label (예: 🐢 이동)" />
            <input v-model="newNode.color" type="color" />
            <!-- 토픽 선택 또는 직접 입력 -->
            <div class="topic-select-row">
            <select v-model="newNode.topic" @change="onTopicSelect(newNode.topic)">
                <option value="">-- 토픽 선택 --</option>
                <option v-for="t in topicList" :key="t" :value="t">{{ t }}</option>
            </select>
            <button class="btn-fetch" @click="fetchTopics" :disabled="loadingTopics">
                {{ loadingTopics ? '⏳' : '🔍 조회' }}
            </button>
            </div>

            <!-- 직접 입력 (조회에 없을 때) -->
            <input
            v-model="newNode.topic"
            placeholder="또는 직접 입력 (예: /roarm/gripper_cmd)"
            />
            <input
            v-model="newNode.messageType"
            placeholder="messageType (예: std_msgs/msg/Float64)"
            />
            <!-- 필드 추가 -->
            <div class="field-add">
              <input v-model="newFieldKey" placeholder="필드명 (예: linear.x)" />
              <select v-model="newFieldType">
                    <option value="float">float (소수)</option>
                    <option value="int">int (정수)</option>
                    <option value="string">string (문자열)</option>
                    <option value="bool">bool (참/거짓)</option>
                    <option value="select">select (선택지)</option>
              </select>
              <button @click="addField">+ 필드</button>
            </div>

            <!-- 추가된 필드 목록 -->
            <div v-for="(meta, key) in newNode.fields" :key="key" class="field-item">
              <span>{{ key }} ({{ meta.type }})</span>
              <button @click="removeField(key)">✕</button>
            </div>

            <button class="btn-add-node" @click="addNodeToForm">
              + 노드 추가
            </button>
          </div>
        </div>

        <div class="modal-footer">
          <button class="btn-cancel" @click="cancelForm">취소</button>
          <button class="btn-save" @click="saveProduct">💾 저장</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.connection-row {
  display: flex;
  align-items: center;
  gap: 4px;
}
.connection-row input { margin: 0; }
.btn-test {
  white-space: nowrap;
  background: #1F4E79;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 6px 12px;
  cursor: pointer;
  font-size: 13px;
}
.connection-status {
  margin-top: 6px;
  font-size: 13px;
  color: #42b883;
}
.product-ip {
  color: #2E75B6;
  font-size: 12px;
}


.topic-select-row {
  display: flex;
  gap: 6px;
  align-items: center;
  margin-bottom: 6px;
}
.topic-select-row select { flex: 1; margin: 0; }
.btn-fetch {
  white-space: nowrap;
  background: #2E75B6;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 6px 12px;
  cursor: pointer;
  font-size: 13px;
}
.btn-fetch:disabled { background: #555; cursor: not-allowed; }


.registry { padding: 24px; background: #0d1117; min-height: 100%; color: #fff; }
.registry-header {
  display: flex; justify-content: space-between;
  align-items: center; margin-bottom: 24px;
}
h2 { color: #42b883; }
.btn-add {
  background: #42b883; color: #fff; border: none;
  border-radius: 6px; padding: 8px 18px; cursor: pointer; font-weight: bold;
}
.product-list { display: flex; flex-direction: column; gap: 12px; }
.empty { color: #555; text-align: center; padding: 40px; }
.product-card {
  background: #161b22; border: 1px solid #333;
  border-radius: 10px; padding: 16px;
}
.product-info {
  display: flex; align-items: center;
  gap: 12px; margin-bottom: 10px;
}
.product-name { color: #fff; font-weight: bold; font-size: 15px; }
.node-count { color: #888; font-size: 12px; }
.node-tags { display: flex; flex-wrap: wrap; gap: 6px; margin-bottom: 12px; }
.node-tag {
  padding: 3px 10px; border-radius: 12px;
  font-size: 11px; color: #fff; font-weight: bold;
}
.product-actions { display: flex; gap: 8px; }
.btn-select {
  background: #2E75B6; color: #fff; border: none;
  border-radius: 6px; padding: 6px 14px; cursor: pointer; font-size: 13px;
}
.btn-edit {
  background: #333; color: #aaa; border: none;
  border-radius: 6px; padding: 6px 12px; cursor: pointer; font-size: 13px;
}
.btn-delete {
  background: #3a1a1a; color: #f44336; border: none;
  border-radius: 6px; padding: 6px 10px; cursor: pointer;
}
/* 모달 */
.modal-overlay {
  position: fixed; inset: 0; background: rgba(0,0,0,0.8);
  display: flex; align-items: center; justify-content: center; z-index: 1000;
}
.modal {
  background: #161b22; border: 1px solid #333; border-radius: 12px;
  width: 560px; max-height: 85vh; overflow-y: auto; padding: 24px;
}
.modal-header {
  display: flex; justify-content: space-between;
  align-items: center; margin-bottom: 20px;
}
.modal-header h3 { color: #42b883; }
.modal-header button { background: none; border: none; color: #888; font-size: 18px; cursor: pointer; }
.form-section { margin-bottom: 20px; }
label { display: block; color: #aaa; font-size: 12px; margin-bottom: 6px; }
input, select {
  width: 100%; background: #0d1117; border: 1px solid #444;
  border-radius: 6px; color: #fff; padding: 8px 10px;
  font-size: 13px; margin-bottom: 6px;
}
input[type="color"] { width: 48px; padding: 2px; cursor: pointer; }
.added-node {
  display: flex; align-items: center; gap: 10px;
  background: #0d1117; border-radius: 6px; padding: 6px 10px; margin-bottom: 4px;
}
.added-node button { margin-left: auto; background: none; border: none; color: #f44336; cursor: pointer; }
.node-topic { color: #555; font-size: 11px; }
.node-form { background: #0d1117; border-radius: 8px; padding: 12px; }
.field-add { display: flex; gap: 6px; align-items: center; margin: 8px 0; }
.field-add input { margin: 0; }
.field-add select { width: auto; margin: 0; }
.field-add button {
  white-space: nowrap; background: #333; color: #aaa;
  border: none; border-radius: 4px; padding: 6px 10px; cursor: pointer;
}
.field-item {
  display: flex; justify-content: space-between; align-items: center;
  color: #42b883; font-size: 12px; padding: 4px 0;
}
.field-item button { background: none; border: none; color: #f44336; cursor: pointer; }
.btn-add-node {
  width: 100%; background: #1F4E79; color: #fff; border: none;
  border-radius: 6px; padding: 8px; cursor: pointer; margin-top: 8px;
}
.modal-footer { display: flex; justify-content: flex-end; gap: 8px; margin-top: 20px; }
.btn-cancel { background: #333; color: #aaa; border: none; border-radius: 6px; padding: 8px 18px; cursor: pointer; }
.btn-save { background: #42b883; color: #fff; border: none; border-radius: 6px; padding: 8px 18px; cursor: pointer; font-weight: bold; }
</style>