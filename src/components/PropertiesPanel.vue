<template>
  <div class="properties">
    <div class="properties-header">属性 & 输出</div>
    <div class="properties-body">
      <!-- 子图管理 -->
      <div class="prop-group">
        <div class="prop-group-title">子图 (Subgraph)</div>
        <div class="prop-row">
          <input
            v-model="newSubgraph.id"
            type="text"
            placeholder="ID"
            style="width: 60px"
          >
          <input
            v-model="newSubgraph.label"
            type="text"
            placeholder="标题"
            style="flex: 1"
          >
          <button class="btn-primary" style="padding: 4px 8px" @click="addSubgraph">+</button>
        </div>
        <div class="subgraph-list">
          <div v-if="subgraphs.length === 0" class="empty-hint">暂无子图</div>
          <div
            v-for="sg in subgraphs"
            :key="sg.id"
            class="subgraph-item"
          >
            <div class="color-dot"></div>
            <span class="subgraph-name">{{ sg.label }}</span>
            <button @click="$emit('remove-subgraph', sg.id)">×</button>
          </div>
        </div>
      </div>

      <!-- 节点属性 -->
      <div v-if="selectedNode" class="prop-group">
        <div class="prop-group-title">节点属性</div>
        <div class="prop-row">
          <label>ID:</label>
          <input type="text" :value="selectedNode.id" readonly>
        </div>
        <div class="prop-row">
          <label>文本:</label>
          <input
            type="text"
            :value="selectedNode.label"
            @change="$emit('update-node', { ...selectedNode, label: $event.target.value })"
          >
        </div>
        <div class="prop-row">
          <label>子图:</label>
          <select
            :value="selectedNode.subgraph"
            @change="$emit('update-node', { ...selectedNode, subgraph: $event.target.value })"
          >
            <option value="">无</option>
            <option v-for="sg in subgraphs" :key="sg.id" :value="sg.id">{{ sg.label }}</option>
          </select>
        </div>
        <div class="prop-row">
          <label>填充:</label>
          <input
            type="color"
            :value="selectedNode.fill"
            @change="$emit('update-node', { ...selectedNode, fill: $event.target.value })"
          >
        </div>
        <div class="prop-row">
          <label>边框:</label>
          <input
            type="color"
            :value="selectedNode.stroke"
            @change="$emit('update-node', { ...selectedNode, stroke: $event.target.value })"
          >
        </div>
      </div>

      <!-- 连线属性 -->
      <div v-if="selectedEdge" class="prop-group">
        <div class="prop-group-title">连线属性</div>
        <div class="prop-row">
          <label>标签:</label>
          <input
            type="text"
            :value="selectedEdge.label"
            @change="$emit('update-edge', { ...selectedEdge, label: $event.target.value })"
          >
        </div>
        <div class="prop-row">
          <label>样式:</label>
          <select
            :value="selectedEdge.type"
            @change="$emit('update-edge', { ...selectedEdge, type: $event.target.value })"
          >
            <option value="solid">实线</option>
            <option value="dashed">虚线</option>
            <option value="thick">粗线</option>
          </select>
        </div>
      </div>
    </div>

    <!-- 输出区域 -->
    <div class="output-section">
      <div class="output-options">
        <label>
          <input type="checkbox" v-model="includeStyles" />
          包含颜色样式
        </label>
      </div>
      <textarea
        :value="mermaidCode"
        readonly
        placeholder="Mermaid 代码..."
      ></textarea>
      <div class="btn-row">
        <button class="btn-primary" @click="copyCode">复制代码</button>
        <button class="btn-secondary" @click="downloadCode">下载 .mmd</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, watch } from 'vue'

const props = defineProps({
  subgraphs: Array,
  selectedNode: Object,
  selectedEdge: Object,
  mermaidCode: String
})

const emit = defineEmits(['add-subgraph', 'remove-subgraph', 'update-node', 'update-edge', 'update-options', 'show-toast'])

const newSubgraph = reactive({ id: '', label: '' })
const includeStyles = ref(true)

// 选项变化时通知父组件
watch(includeStyles, (val) => {
  emit('update-options', { includeStyles: val })
})

function addSubgraph() {
  if (!newSubgraph.id) {
    emit('show-toast', '请输入子图ID', 'warning')
    return
  }
  emit('add-subgraph', { id: newSubgraph.id, label: newSubgraph.label || newSubgraph.id })
  newSubgraph.id = ''
  newSubgraph.label = ''
}

function copyCode() {
  if (!props.mermaidCode) {
    emit('show-toast', '没有内容', 'warning')
    return
  }
  navigator.clipboard.writeText(props.mermaidCode).then(() => {
    emit('show-toast', '已复制到剪贴板', 'success')
  })
}

function downloadCode() {
  if (!props.mermaidCode) {
    emit('show-toast', '请先生成代码', 'warning')
    return
  }
  const a = document.createElement('a')
  a.href = URL.createObjectURL(new Blob([props.mermaidCode], { type: 'text/plain' }))
  a.download = 'flowchart.mmd'
  a.click()
  emit('show-toast', '下载成功', 'success')
}
</script>

<style scoped>
.properties {
  width: 280px;
  background: #fff;
  border-left: 1px solid #ddd;
  display: flex;
  flex-direction: column;
}
.properties-header {
  padding: 12px;
  background: #1976d2;
  color: white;
  font-weight: 500;
}
.properties-body {
  flex: 1;
  padding: 12px;
  overflow-y: auto;
}
.prop-group {
  margin-bottom: 16px;
}
.prop-group-title {
  font-size: 12px;
  color: #666;
  margin-bottom: 8px;
  font-weight: 500;
}
.prop-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}
.prop-row label {
  width: 60px;
  font-size: 12px;
  color: #333;
}
.prop-row input[type="text"] {
  flex: 1;
  padding: 6px 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 12px;
}
.prop-row input[type="color"] {
  width: 40px;
  height: 28px;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}
.prop-row select {
  flex: 1;
  padding: 6px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 12px;
}
.subgraph-list {
  margin-top: 8px;
  max-height: 120px;
  overflow-y: auto;
}
.empty-hint {
  color: #999;
  font-size: 11px;
  padding: 8px;
}
.subgraph-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  background: #fff3e0;
  border: 1px dashed #ff9800;
  border-radius: 4px;
  margin-bottom: 4px;
  font-size: 12px;
}
.color-dot {
  width: 12px;
  height: 12px;
  background: #ff9800;
  border-radius: 2px;
}
.subgraph-name {
  flex: 1;
  font-weight: 500;
  color: #e65100;
}
.subgraph-item button {
  padding: 2px 6px;
  background: #ef5350;
  color: white;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  font-size: 11px;
}
.output-section {
  border-top: 1px solid #ddd;
  padding: 12px;
}
.output-options {
  display: flex;
  gap: 16px;
  margin-bottom: 8px;
  font-size: 12px;
}
.output-options label {
  display: flex;
  align-items: center;
  gap: 4px;
  cursor: pointer;
}
.output-section textarea {
  width: 100%;
  height: 150px;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 8px;
  font-family: 'Consolas', monospace;
  font-size: 11px;
  resize: none;
}
.btn-row {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}
.btn-row button {
  flex: 1;
  padding: 8px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
}
.btn-primary {
  background: #1976d2;
  color: white;
}
.btn-secondary {
  background: #757575;
  color: white;
}
</style>