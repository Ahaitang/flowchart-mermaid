<template>
  <div class="app-container">
    <!-- 左侧形状工具栏 -->
    <ShapePalette
      @generate="generateMermaid"
      @clear="clearCanvas"
      @load-example="loadExample"
    />

    <!-- 中间画布区域 -->
    <div class="canvas-area">
      <div class="canvas-toolbar">
        <button
          :class="{ active: mode === 'select' }"
          @click="mode = 'select'"
        >选择</button>
        <button
          :class="{ active: mode === 'connect' }"
          @click="mode = 'connect'"
        >连线</button>
        <div class="divider"></div>
        <button @click="deleteSelected">删除选中</button>
        <div class="divider"></div>
        <label>方向: </label>
        <select v-model="flowDirection">
          <option value="TD">从上到下</option>
          <option value="LR">从左到右</option>
          <option value="TB">从上到下</option>
          <option value="RL">从右到左</option>
        </select>
      </div>
      <FlowCanvas
        ref="canvasRef"
        :nodes="nodes"
        :edges="edges"
        :subgraphs="subgraphs"
        :mode="mode"
        :selected-edge="selectedEdge"
        :selected-subgraph="selectedSubgraph"
        :node-size="nodeSize"
        @add-node="addNode"
        @select-node="selectNode"
        @select-edge="selectEdge"
        @update-node="updateNode"
        @add-edge="addEdge"
        @edit-node="node => showInput('编辑节点文本', node.label, node)"
        @select-subgraph="selectSubgraph"
        @move-subgraph="moveSubgraph"
      />
    </div>

    <!-- 右侧属性面板 -->
    <PropertiesPanel
      :subgraphs="subgraphs"
      :selected-node="selectedNode"
      :selected-edge="selectedEdge"
      :selected-subgraph="selectedSubgraph"
      :mermaid-code="mermaidCode"
      @add-subgraph="addSubgraph"
      @remove-subgraph="removeSubgraph"
      @update-node="updateNode"
      @update-edge="updateEdge"
      @update-subgraph="updateSubgraph"
      @select-subgraph="selectSubgraph"
      @update-options="updateOptions"
      @show-toast="showToast"
    />
  </div>

  <!-- 弹窗 -->
  <Modal
    v-model:visible="modalVisible"
    :type="modalType"
    :title="modalTitle"
    :message="modalMessage"
    :message-type="modalMessageType"
    :default-value="modalInputDefault"
    @confirm="onModalConfirm"
  />
</template>

<script setup>
import { ref, computed } from 'vue'
import ShapePalette from './components/ShapePalette.vue'
import FlowCanvas from './components/FlowCanvas.vue'
import PropertiesPanel from './components/PropertiesPanel.vue'
import Modal from './components/Modal.vue'

// 数据
const nodes = ref([])
const edges = ref([])
const subgraphs = ref([])
const nodeIdCounter = ref(1)
const selectedNode = ref(null)
const selectedEdge = ref(null)
const selectedSubgraph = ref(null)
const mode = ref('select')
const flowDirection = ref('TD')
const canvasRef = ref(null)
const includeStyles = ref(true)

// 弹窗状态
const modalVisible = ref(false)
const modalType = ref('message')
const modalTitle = ref('提示')
const modalMessage = ref('')
const modalMessageType = ref('info')
const modalInputDefault = ref('')
const editingNode = ref(null)

const nodeSize = { width: 120, height: 50 }

// 弹窗方法
function showToast(message, type = 'info') {
  modalType.value = 'message'
  modalTitle.value = '提示'
  modalMessage.value = message
  modalMessageType.value = type
  modalVisible.value = true
}

function showInput(title, defaultValue, node) {
  modalType.value = 'input'
  modalTitle.value = title
  modalInputDefault.value = defaultValue
  editingNode.value = node
  modalVisible.value = true
}

function onModalConfirm(value) {
  if (modalType.value === 'input' && editingNode.value) {
    updateNode({ ...editingNode.value, label: value || editingNode.value.id })
    editingNode.value = null
  }
}

// 形状默认颜色
const shapeColors = {
  rect: { fill: '#e3f2fd', stroke: '#1976d2' },
  rounded: { fill: '#e8f5e9', stroke: '#4caf50' },
  stadium: { fill: '#fff3e0', stroke: '#ff9800' },
  diamond: { fill: '#fce4ec', stroke: '#e91e63' },
  circle: { fill: '#f3e5f5', stroke: '#9c27b0' },
  hexagon: { fill: '#e0f7fa', stroke: '#00bcd4' },
  parallelogram: { fill: '#fff8e1', stroke: '#ffc107' },
  trapezoid: { fill: '#e8eaf6', stroke: '#3f51b5' }
}

// 节点操作
function addNode({ shape, x, y }) {
  const id = 'N' + nodeIdCounter.value++
  const colors = shapeColors[shape]
  nodes.value.push({
    id,
    shape,
    x,
    y,
    label: id,
    fill: colors.fill,
    stroke: colors.stroke,
    subgraph: ''
  })
  selectedNode.value = nodes.value[nodes.value.length - 1]
  selectedEdge.value = null
}

function updateNode(node) {
  const index = nodes.value.findIndex(n => n.id === node.id)
  if (index !== -1) {
    nodes.value.splice(index, 1, { ...node })
    // 同步更新 selectedNode
    if (selectedNode.value?.id === node.id) {
      selectedNode.value = nodes.value[index]
    }
  }
}

function selectNode(node) {
  selectedNode.value = node
  selectedEdge.value = null
  selectedSubgraph.value = null
}

// 连线操作
function addEdge({ from, to }) {
  if (edges.value.find(e => e.from === from && e.to === to)) return
  edges.value.push({
    id: 'E' + Date.now(),
    from,
    to,
    label: '',
    type: 'solid'
  })
}

function updateEdge(edge) {
  const index = edges.value.findIndex(e => e.id === edge.id)
  if (index !== -1) {
    edges.value.splice(index, 1, { ...edge })
    // 同步更新 selectedEdge
    if (selectedEdge.value?.id === edge.id) {
      selectedEdge.value = edges.value[index]
    }
  }
}

function selectEdge(edge) {
  selectedEdge.value = edge
  selectedNode.value = null
  selectedSubgraph.value = null
}

function selectSubgraph(sg) {
  selectedSubgraph.value = sg
  selectedNode.value = null
  selectedEdge.value = null
}

// 更新输出选项
function updateOptions(options) {
  includeStyles.value = options.includeStyles
}

// 子图操作
function addSubgraph({ id, label, fill, stroke }) {
  if (subgraphs.value.find(s => s.id === id)) {
    showToast('子图ID已存在', 'warning')
    return
  }
  subgraphs.value.push({ id, label, fill, stroke })
}

function updateSubgraph(sg) {
  const index = subgraphs.value.findIndex(s => s.id === sg.id)
  if (index !== -1) {
    subgraphs.value.splice(index, 1, { ...sg })
    if (selectedSubgraph.value?.id === sg.id) {
      selectedSubgraph.value = subgraphs.value[index]
    }
  }
}

function removeSubgraph(id) {
  subgraphs.value = subgraphs.value.filter(s => s.id !== id)
  nodes.value.forEach(n => {
    if (n.subgraph === id) n.subgraph = ''
  })
  if (selectedSubgraph.value?.id === id) {
    selectedSubgraph.value = null
  }
}

// 移动子图（移动所有内部节点）
function moveSubgraph({ id, dx, dy, startPositions }) {
  nodes.value = nodes.value.map(n => {
    if (n.subgraph === id && startPositions[n.id]) {
      return {
        ...n,
        x: startPositions[n.id].x + dx,
        y: startPositions[n.id].y + dy
      }
    }
    return n
  })
}

// 删除选中
function deleteSelected() {
  if (selectedNode.value) {
    edges.value = edges.value.filter(e =>
      e.from !== selectedNode.value.id && e.to !== selectedNode.value.id
    )
    nodes.value = nodes.value.filter(n => n.id !== selectedNode.value.id)
    selectedNode.value = null
  } else if (selectedEdge.value) {
    edges.value = edges.value.filter(e => e.id !== selectedEdge.value.id)
    selectedEdge.value = null
  }
}

// 生成 Mermaid 代码
const mermaidCode = computed(() => {
  let code = `flowchart ${flowDirection.value}\n`

  // 根级节点
  const rootNodes = nodes.value.filter(n => !n.subgraph)
  rootNodes.forEach(n => { code += `    ${getMermaidShape(n)}\n` })

  // 根级连接
  edges.value.filter(e => {
    const f = nodes.value.find(n => n.id === e.from)
    const t = nodes.value.find(n => n.id === e.to)
    return !f?.subgraph && !t?.subgraph
  }).forEach(e => { code += `    ${e.from} ${getMermaidArrow(e)} ${e.to}\n` })

  // 子图
  subgraphs.value.forEach(sg => {
    code += `\n    subgraph ${sg.id} ["${sg.label}"]\n`
    nodes.value.filter(n => n.subgraph === sg.id).forEach(n => {
      code += `        ${getMermaidShape(n)}\n`
    })
    edges.value.filter(e => {
      const f = nodes.value.find(n => n.id === e.from)
      const t = nodes.value.find(n => n.id === e.to)
      return f?.subgraph === sg.id && t?.subgraph === sg.id
    }).forEach(e => { code += `        ${e.from} ${getMermaidArrow(e)} ${e.to}\n` })
    code += '    end\n'
  })

  // 跨子图连接
  edges.value.filter(e => {
    const f = nodes.value.find(n => n.id === e.from)
    const t = nodes.value.find(n => n.id === e.to)
    return f?.subgraph !== t?.subgraph
  }).forEach(e => { code += `    ${e.from} ${getMermaidArrow(e)} ${e.to}\n` })

  // 样式（可选）
  if (includeStyles.value) {
    nodes.value.forEach(n => {
      code += `    style ${n.id} fill:${n.fill},stroke:${n.stroke},stroke-width:2px\n`
    })
  }

  return code
})

function getMermaidShape(node) {
  const l = node.label
  const shapes = {
    rect: `${node.id}[${l}]`,
    rounded: `${node.id}(${l})`,
    stadium: `${node.id}([${l}])`,
    diamond: `${node.id}{${l}}`,
    circle: `${node.id}((${l}))`,
    hexagon: `${node.id}{{${l}}}`,
    parallelogram: `${node.id}[/${l}/]`,
    trapezoid: `${node.id}[/${l}\\]`
  }
  return shapes[node.shape] || `${node.id}[${l}]`
}

function getMermaidArrow(edge) {
  const label = edge.label
  if (edge.type === 'dashed') return label ? `-. ${label} .->` : '-.->'
  if (edge.type === 'thick') return label ? `== ${label} ==>` : '==>'
  return label ? `-- ${label} -->` : '-->'
}

// 清空画布
function clearCanvas() {
  nodes.value = []
  edges.value = []
  subgraphs.value = []
  nodeIdCounter.value = 1
  selectedNode.value = null
  selectedEdge.value = null
  selectedSubgraph.value = null
}

// 加载示例
function loadExample() {
  clearCanvas()
  subgraphs.value = [
    { id: 'login', label: '登录流程', fill: '#e3f2fd', stroke: '#1976d2' },
    { id: 'process', label: 'SN冲突处理', fill: '#fce4ec', stroke: '#e91e63' }
  ]

  nodes.value = [
    { id: 'A', shape: 'stadium', x: 300, y: 30, label: '开始', fill: '#e8f5e9', stroke: '#4caf50', subgraph: '' },
    { id: 'B', shape: 'rect', x: 260, y: 120, label: '用户登录', fill: '#e3f2fd', stroke: '#1976d2', subgraph: 'login' },
    { id: 'C', shape: 'diamond', x: 260, y: 210, label: '验证?', fill: '#fce4ec', stroke: '#e91e63', subgraph: 'login' },
    { id: 'D', shape: 'rect', x: 260, y: 320, label: '检测SN', fill: '#e3f2fd', stroke: '#1976d2', subgraph: 'process' },
    { id: 'E', shape: 'diamond', x: 260, y: 410, label: '冲突?', fill: '#fce4ec', stroke: '#e91e63', subgraph: 'process' },
    { id: 'F', shape: 'rect', x: 100, y: 410, label: '解决冲突', fill: '#ffebee', stroke: '#f44336', subgraph: 'process' },
    { id: 'G', shape: 'rounded', x: 260, y: 500, label: '完成', fill: '#e8f5e9', stroke: '#4caf50', subgraph: '' },
    { id: 'H', shape: 'stadium', x: 300, y: 590, label: '结束', fill: '#f3e5f5', stroke: '#9c27b0', subgraph: '' }
  ]

  edges.value = [
    { id: 'E1', from: 'A', to: 'B', label: '', type: 'solid' },
    { id: 'E2', from: 'B', to: 'C', label: '', type: 'solid' },
    { id: 'E3', from: 'C', to: 'D', label: '成功', type: 'solid' },
    { id: 'E4', from: 'C', to: 'B', label: '失败', type: 'dashed' },
    { id: 'E5', from: 'D', to: 'E', label: '', type: 'solid' },
    { id: 'E6', from: 'E', to: 'F', label: '是', type: 'solid' },
    { id: 'E7', from: 'E', to: 'G', label: '否', type: 'solid' },
    { id: 'E8', from: 'F', to: 'G', label: '', type: 'solid' },
    { id: 'E9', from: 'G', to: 'H', label: '', type: 'solid' }
  ]

  nodeIdCounter.value = 9
}

function generateMermaid() {
  showToast('Mermaid 代码已生成！', 'success')
}
</script>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #f5f5f5;
  height: 100vh;
  overflow: hidden;
}
</style>

<style scoped>
.app-container {
  display: flex;
  height: 100vh;
}

.canvas-area {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.canvas-toolbar {
  padding: 8px 12px;
  background: #fff;
  border-bottom: 1px solid #ddd;
  display: flex;
  gap: 8px;
  align-items: center;
}

.canvas-toolbar button {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #fff;
  cursor: pointer;
  font-size: 13px;
}

.canvas-toolbar button:hover {
  background: #f5f5f5;
}

.canvas-toolbar button.active {
  background: #e3f2fd;
  border-color: #1976d2;
}

.canvas-toolbar .divider {
  width: 1px;
  height: 24px;
  background: #ddd;
  margin: 0 8px;
}

.canvas-toolbar select {
  padding: 6px;
  border: 1px solid #ddd;
  border-radius: 4px;
}
</style>