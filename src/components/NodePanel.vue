<!-- 좌측 노드 목록 (신규생성) -->
 <script setup>
defineProps({
  nodes: { type: Array, default: () => [] }
})

function onDragStart(event, node) {
  event.dataTransfer.setData('nodeType', JSON.stringify(node))
  event.dataTransfer.effectAllowed = 'move'
}
</script>

<template>
  <div class="panel">
    <h3>📦 노드 목록</h3>
    <p class="hint">캔버스로 드래그하세요</p>
    <div
      v-for="node in nodes"
      :key="node.type"
      class="node-item"
      :style="{ borderLeft: `4px solid ${node.color}` }"
      draggable="true"
      @dragstart="onDragStart($event, node)"
    >
      {{ node.label }}
    </div>
  </div>
</template>

<style scoped>
.panel {
  width: 180px;
  background: #1a1a2e;
  border-right: 1px solid #333;
  padding: 16px;
  flex-shrink: 0;
}
h3 { color: #42b883; margin: 0 0 4px; font-size: 14px; }
.hint { color: #888; font-size: 11px; margin: 0 0 16px; }
.node-item {
  background: #16213e;
  color: #fff;
  padding: 10px 12px;
  margin-bottom: 8px;
  border-radius: 6px;
  cursor: grab;
  font-size: 13px;
  user-select: none;
  transition:  0.2s;
}
.node-item:hover { background: #0f3460; }
</style>