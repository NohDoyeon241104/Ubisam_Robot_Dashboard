<!-- Vue Flow 캔버스 (신규 생성) -->
<script setup>
import { ref, onMounted ,computed, watch} from 'vue'  // computed추가 computed는 props로 받은 product의 connection 정보에 따라 wsUrl을 동적으로 생성하기 위해 사용 //watch 추가 - product가 변경될 때마다 노드/엣지 초기화
import { VueFlow, useVueFlow } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import { Controls } from '@vue-flow/controls'
import '@vue-flow/core/dist/style.css'
import '@vue-flow/controls/dist/style.css'

import NodePanel from './NodePanel.vue'
import NodeConfigForm from './NodeConfigForm.vue'
import { Ros, Topic } from 'roslib'

// import nodeDefinitions from '../data/jazzy-nodes.json'
// 삭제 후 아래로 대체
const nodeDefinitions = computed(() => {
  return props.product || { product: 'ROS2 Jazzy Demo', nodes: [] }
})

const { addNodes, addEdges, getNodes, getEdges } = useVueFlow()

const nodes = ref([])
const edges = ref([])
const selectedNode = ref(null)
const isRunning = ref(false)
const currentStep = ref(-1)
const statusLog = ref([])



const props = defineProps({
  product: { type: Object, default: null }
})

// product가 있으면 해당 IP로, 없으면 localhost로 연결
const wsUrl = computed(() => {
  if (props.product?.connection) {
    return `ws://${props.product.connection.host}:${props.product.connection.port}`
  }
  return 'ws://localhost:9090'
})
// Rosbridge 연결
const ros = new Ros({ url: wsUrl.value })

// ✅ 장비 전환 시 노드/엣지 초기화
watch(() => props.product, () => {
  nodes.value = []
  edges.value = []
  selectedNode.value = null
  statusLog.value = []
})


// 캔버스에 드롭
function onDrop(event) {
  const raw = event.dataTransfer.getData('nodeType')
  if (!raw) return
  const def = JSON.parse(raw)

  const bounds = event.currentTarget.getBoundingClientRect()
  const position = {
    x: event.clientX - bounds.left - 80,
    y: event.clientY - bounds.top - 20,
  }

  // 각 노드의 fields 값을 복사해서 사용자 편집 가능하게
  const userFields = {}
  for (const [key, meta] of Object.entries(def.fields)) {
    userFields[key] = meta.default
  }

  const newNode = {
    id: `${def.type}-${Date.now()}`,
    type: 'default',
    position,
    label: def.label,
    data: {
      ...def,
      userFields,
    },
    style: {
      background: def.color,
      color: '#fff',
      border: 'none',
      borderRadius: '8px',
      padding: '10px 16px',
      fontWeight: 'bold',
      fontSize: '13px',
    }
  }

  nodes.value.push(newNode)
}

// 노드 클릭 시 설정 폼 표시
function onNodeClick({ node }) {
  selectedNode.value = JSON.parse(JSON.stringify(node.data))
  selectedNode.value._id = node.id
}

// 폼에서 값 변경 시 반영
function onFieldUpdate({ key, value }) {
  if (!selectedNode.value) return
  selectedNode.value.userFields[key] = value

  // nodes 배열에도 반영
  const target = nodes.value.find(n => n.id === selectedNode.value._id)
  if (target) target.data.userFields[key] = value
}

// 연결된 순서대로 노드 정렬
function getSortedNodes() {
  const allEdges = edges.value
  const allNodes = nodes.value

  if (allEdges.length === 0) return allNodes

  // 시작 노드 찾기 (들어오는 엣지 없는 노드)
  const targetIds = new Set(allEdges.map(e => e.target))
  const startNode = allNodes.find(n => !targetIds.has(n.id))
  if (!startNode) return allNodes

  // 연결 순서대로 정렬
  const sorted = []
  let current = startNode
  const visited = new Set()

  while (current && !visited.has(current.id)) {
    sorted.push(current)
    visited.add(current.id)
    const nextEdge = allEdges.find(e => e.source === current.id)
    current = nextEdge ? allNodes.find(n => n.id === nextEdge.target) : null
  }

  return sorted
}

// 시퀀스 실행
async function runSequence() {
  if (isRunning.value) return
  isRunning.value = true
  statusLog.value = []
  currentStep.value = 0

  const sequence = getSortedNodes()

  for (let i = 0; i < sequence.length; i++) {
    currentStep.value = i
    const node = sequence[i]
    const { type, topic, messageType, userFields, label } = node.data

    statusLog.value.push(`▶ [${i + 1}/${sequence.length}] ${label} 실행 중...`)

    if (type === 'wait') {
      // 대기 노드
      await new Promise(r => setTimeout(r, (userFields.duration || 1) * 1000))

    } else if (topic && messageType) {
      // ROS2 토픽 발행
      const pubTopic = new Topic({ ros, name: topic, messageType })

      let message = {}
      if (messageType === 'geometry_msgs/msg/Twist') {
        message = {
          linear:  { x: userFields['linear.x'] || 0, y: 0, z: 0 },
          angular: { x: 0, y: 0, z: userFields['angular.z'] || 0 }
        }
        } else if (messageType === 'std_msgs/msg/Float64') {
        message = { data: userFields['data'] || 0.0 }
        }

      pubTopic.publish(message)

      // 정지 노드가 아니면 1초 동작 후 정지
      if (type !== 'turtle_stop') {
        await new Promise(r => setTimeout(r, 1000))
        if (topic === '/turtle1/cmd_vel') {
          pubTopic.publish({ linear: { x: 0, y: 0, z: 0 }, angular: { x: 0, y: 0, z: 0 } })
        }
      }
    }

    statusLog.value[statusLog.value.length - 1] =
      `✅ [${i + 1}/${sequence.length}] ${label} 완료`
  }

  currentStep.value = -1
  isRunning.value = false
  statusLog.value.push('🎉 시퀀스 실행 완료!')
}

function resetFlow() {
  nodes.value = []
  edges.value = []
  selectedNode.value = null
  statusLog.value = []
  currentStep.value = -1
}
</script>

<template>
  <div class="flow-wrapper">

    <!-- 좌측: 노드 패널 -->
    <NodePanel :nodes="nodeDefinitions.nodes" />

    <!-- 중앙: Vue Flow 캔버스 -->
    <div class="canvas-area" @drop="onDrop" @dragover.prevent>
      <div class="toolbar">
        <span class="product-name">📦 {{ nodeDefinitions.product }}</span>
        <button class="btn-run" :disabled="isRunning" @click="runSequence">
          {{ isRunning ? '⏳ 실행 중...' : '▶ 시퀀스 실행' }}
        </button>
        <button class="btn-reset" @click="resetFlow">🔄 초기화</button>
      </div>

      <VueFlow
        v-model:nodes="nodes"
        v-model:edges="edges"
        :default-edge-options="{ animated: true, style: { stroke: '#42b883', strokeWidth: 3 } }"
        fit-view-on-init
        @node-click="onNodeClick"
        style="height: calc(100% - 48px);"
      >
        <Background pattern-color="#333" :gap="20" />
        <Controls />
      </VueFlow>

      <!-- 하단 실행 로그 -->
      <div class="log-area" v-if="statusLog.length > 0">
        <div v-for="(log, i) in statusLog" :key="i" class="log-line">
          {{ log }}
        </div>
      </div>
    </div>

    <!-- 우측: 노드 설정 폼 -->
    <NodeConfigForm
      :node="selectedNode"
      @update="onFieldUpdate"
      @close="selectedNode = null"
    />

  </div>
</template>

<style scoped>
.flow-wrapper {
  display: flex;
  height: 100%;
  background: #0d1117;
}
.canvas-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  position: relative;
}
.toolbar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 16px;
  background: #161b22;
  border-bottom: 1px solid #333;
  height: 48px;
  box-sizing: border-box;
}
.product-name {
  color: #42b883;
  font-weight: bold;
  font-size: 14px;
  flex: 1;
}
.btn-run {
  background: #42b883;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 6px 18px;
  font-size: 13px;
  font-weight: bold;
  cursor: pointer;
}
.btn-run:disabled { background: #555; cursor: not-allowed; }
.btn-reset {
  background: #333;
  color: #aaa;
  border: none;
  border-radius: 6px;
  padding: 6px 14px;
  font-size: 13px;
  cursor: pointer;
}
.log-area {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: rgba(0,0,0,0.85);
  padding: 10px 16px;
  max-height: 120px;
  overflow-y: auto;
  border-top: 1px solid #333;
}
.log-line {
  color: #42b883;
  font-size: 12px;
  font-family: monospace;
  line-height: 1.8;
}


/* Vue Flow 핸들 강제 표시 */
:deep(.vue-flow__handle) {
  width: 12px;
  height: 12px;
  background: #42b883;
  border: 2px solid #fff;
  border-radius: 50%;
  cursor: crosshair;
}

:deep(.vue-flow__handle:hover) {
  background: #fff;
  transform: scale(1.3);
}

:deep(.vue-flow__node) {
  cursor: grab;
}

:deep(.vue-flow__edge-path) {
  stroke: #42b883;
  stroke-width: 3;
}

:deep(.vue-flow__edge.animated .vue-flow__edge-path) {
  stroke: #42b883;
  stroke-dasharray: 8;
}

:deep(.vue-flow__arrowhead) {
  fill: #42b883;
}
</style>