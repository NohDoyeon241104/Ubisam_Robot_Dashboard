<!-- 노드 값 편집 폼 (신규생성) -->

<script setup>
defineProps({
  node: { type: Object, default: null }
})
defineEmits(['update', 'close'])
</script>

<template>
  <div v-if="node" class="form-panel">
    <div class="form-header">
      <span>{{ node.label }} 설정</span>
      <button @click="$emit('close')">✕</button>
    </div>

    <div v-if="Object.keys(node.fields).length === 0" class="no-fields">
      설정값 없음 (즉시 실행)
    </div>

    <div v-for="(meta, key) in node.fields" :key="key" class="field">
      <label>{{ key }} <span v-if="meta.unit">({{ meta.unit }})</span></label>

      <select v-if="meta.type === 'select'" :value="meta.default"
        @change="$emit('update', { key, value: $event.target.value })">
        <option v-for="opt in meta.options" :key="opt" :value="opt">{{ opt }}</option>
      </select>

      <input v-else-if="meta.type === 'float'" type="number" step="0.1"
        :min="meta.min" :max="meta.max" :value="meta.default"
        @input="$emit('update', { key, value: parseFloat($event.target.value) })" />

      <input v-else type="text" :value="meta.default"
        @input="$emit('update', { key, value: $event.target.value })" />
    </div>

    <div class="topic-info" v-if="node.topic">
      <span>📡 {{ node.topic }}</span>
      <span>{{ node.messageType }}</span>
    </div>
  </div>
</template>

<style scoped>
.form-panel {
  width: 220px;
  background: #1a1a2e;
  border-left: 1px solid #333;
  padding: 16px;
  flex-shrink: 0;
}
.form-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #42b883;
  font-weight: bold;
  font-size: 13px;
  margin-bottom: 16px;
}
.form-header button {
  background: none;
  border: none;
  color: #888;
  cursor: pointer;
  font-size: 16px;
}
.field { margin-bottom: 12px; }
label { display: block; color: #aaa; font-size: 11px; margin-bottom: 4px; }
input, select {
  width: 100%;
  background: #16213e;
  border: 1px solid #444;
  border-radius: 4px;
  color: #fff;
  padding: 6px 8px;
  font-size: 13px;
  box-sizing: border-box;
}
.no-fields { color: #888; font-size: 12px; }
.topic-info {
  margin-top: 16px;
  padding-top: 12px;
  border-top: 1px solid #333;
  font-size: 11px;
  color: #666;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
</style>