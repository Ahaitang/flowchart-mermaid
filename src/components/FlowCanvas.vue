<template>
  <div
    class="canvas-wrapper"
    ref="wrapperRef"
    @dragover.prevent
    @drop="onDrop"
    @wheel="onWheel"
    @mousedown="onCanvasMouseDown"
    @mousemove="onCanvasMouseMove"
    @mouseup="onCanvasMouseUp"
    @mouseleave="onCanvasMouseUp"
  >
    <svg ref="svgRef" id="canvas">
      <!-- 主内容组，应用缩放和平移 -->
      <g :transform="`translate(${pan.x}, ${pan.y}) scale(${scale})`">
        <!-- 子图边界框 -->
        <g
          v-for="sg in subgraphBoxes"
          :key="sg.id"
          class="subgraph-group"
          @mousedown.stop="onSubgraphMouseDown($event, sg)"
          @click.stop="selectSubgraph(sg)"
        >
          <rect
            :x="sg.x"
            :y="sg.y"
            :width="sg.width"
            :height="sg.height"
            rx="8"
            :fill="sg.fill"
            :fill-opacity="0.3"
            :stroke="sg.stroke"
            :stroke-width="props.selectedSubgraph?.id === sg.id ? 3 : 2"
            :stroke-dasharray="props.selectedSubgraph?.id === sg.id ? 'none' : '8,4'"
          />
          <rect
            :x="sg.x + 10"
            :y="sg.y"
            :width="sg.labelWidth"
            height="20"
            :fill="sg.fill"
          />
          <text
            :x="sg.x + 18"
            :y="sg.y + 14"
            font-size="12"
            font-weight="500"
            :fill="sg.stroke"
          >{{ sg.label }}</text>
        </g>

        <!-- 连线 -->
        <g
          v-for="edge in edges"
          :key="edge.id"
          :id="'edge-' + edge.id"
          class="edge-group"
          @click.stop="selectEdge(edge)"
        >
          <line
            :x1="edgePositions[edge.id]?.x1 || 0"
            :y1="edgePositions[edge.id]?.y1 || 0"
            :x2="edgePositions[edge.id]?.x2 || 0"
            :y2="edgePositions[edge.id]?.y2 || 0"
            :stroke="props.selectedEdge?.id === edge.id ? '#1976d2' : '#666'"
            :stroke-width="edge.type === 'thick' ? 3 : 2"
            :stroke-dasharray="edge.type === 'dashed' ? '5,5' : 'none'"
          />
          <polygon
            :points="edgePositions[edge.id]?.arrow || ''"
            fill="#666"
          />
          <rect
            v-if="edge.label"
            :x="(edgePositions[edge.id]?.x1 + edgePositions[edge.id]?.x2) / 2 - edge.label.length * 4"
            :y="(edgePositions[edge.id]?.y1 + edgePositions[edge.id]?.y2) / 2 - 8"
            :width="edge.label.length * 8 + 8"
            height="16"
            fill="white"
          />
          <text
            v-if="edge.label"
            :x="(edgePositions[edge.id]?.x1 + edgePositions[edge.id]?.x2) / 2"
            :y="(edgePositions[edge.id]?.y1 + edgePositions[edge.id]?.y2) / 2 + 4"
            font-size="11"
            text-anchor="middle"
            fill="#333"
          >{{ edge.label }}</text>
        </g>

        <!-- 节点 -->
        <g
          v-for="node in nodes"
          :key="node.id"
          :id="'node-' + node.id"
          class="node-group"
          :transform="`translate(${node.x}, ${node.y})`"
          :style="{ cursor: mode === 'select' ? 'move' : 'pointer' }"
        >
          <path
            :d="getNodePath(node.shape)"
            :fill="node.fill"
            :stroke="node.stroke"
            stroke-width="2"
            class="node-shape"
            @mousedown.stop="onNodeMouseDown($event, node)"
            @dblclick.stop="onNodeDoubleClick(node)"
          />
          <text
            :x="nodeSize.width / 2"
            :y="nodeSize.height / 2 + 5"
            text-anchor="middle"
            font-size="12"
            fill="#333"
            style="pointer-events: none"
          >{{ node.label }}</text>
          <rect
            x="-5"
            y="-5"
            :width="nodeSize.width + 10"
            :height="nodeSize.height + 10"
            fill="none"
            stroke="#1976d2"
            stroke-width="2"
            stroke-dasharray="5,5"
            :display="selectedNode?.id === node.id ? 'block' : 'none'"
            class="select-box"
            style="pointer-events: none"
          />
        </g>
      </g>
    </svg>

    <!-- 缩放控件 -->
    <div class="zoom-controls">
      <button @click="zoomIn" title="放大">+</button>
      <span class="zoom-level">{{ Math.round(scale * 100) }}%</span>
      <button @click="zoomOut" title="缩小">−</button>
      <button @click="resetZoom" title="重置">⟲</button>
    </div>

    <div class="hint">{{ hint }}</div>
  </div>
</template>

<script setup>
import { ref, computed, reactive, onMounted, onUnmounted, watch } from 'vue'

const props = defineProps({
  nodes: Array,
  edges: Array,
  subgraphs: Array,
  mode: String,
  nodeSize: Object,
  selectedEdge: Object,
  selectedSubgraph: Object
})

const emit = defineEmits(['add-node', 'select-node', 'select-edge', 'update-node', 'add-edge', 'edit-node', 'select-subgraph', 'move-subgraph'])

const svgRef = ref(null)
const wrapperRef = ref(null)
const selectedNode = ref(null)
const connectingFrom = ref(null)
const hint = ref('拖拽形状到画布 | 滚轮缩放 | 空白处拖拽画布')

const dragNodeId = ref(null)
const dragOffset = ref({ x: 0, y: 0 })

// 缩放和平移状态
const scale = ref(1)
const pan = reactive({ x: 0, y: 0 })
const isPanning = ref(false)
const panStart = reactive({ x: 0, y: 0 })
const spacePressed = ref(false)

const MIN_SCALE = 0.25
const MAX_SCALE = 3
const ZOOM_STEP = 0.1

// 子图边界框计算
const subgraphBoxes = computed(() => {
  return props.subgraphs.map(sg => {
    const sgNodes = props.nodes.filter(n => n.subgraph === sg.id)
    if (sgNodes.length === 0) return null

    let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
    sgNodes.forEach(n => {
      minX = Math.min(minX, n.x)
      minY = Math.min(minY, n.y)
      maxX = Math.max(maxX, n.x + props.nodeSize.width)
      maxY = Math.max(maxY, n.y + props.nodeSize.height)
    })

    const padding = 20
    return {
      id: sg.id,
      label: sg.label,
      fill: sg.fill || '#fff3e0',
      stroke: sg.stroke || '#ff9800',
      x: minX - padding,
      y: minY - padding - 20,
      width: maxX - minX + padding * 2,
      height: maxY - minY + padding * 2 + 20,
      labelWidth: sg.label.length * 12 + 16
    }
  }).filter(Boolean)
})

// 连线位置计算
const edgePositions = computed(() => {
  const positions = {}
  props.edges.forEach(edge => {
    const fromNode = props.nodes.find(n => n.id === edge.from)
    const toNode = props.nodes.find(n => n.id === edge.to)
    if (!fromNode || !toNode) return

    const fromCenter = {
      x: fromNode.x + props.nodeSize.width / 2,
      y: fromNode.y + props.nodeSize.height / 2
    }
    const toCenter = {
      x: toNode.x + props.nodeSize.width / 2,
      y: toNode.y + props.nodeSize.height / 2
    }

    const fromPoint = getEdgePoint(fromNode, toCenter)
    const toPoint = getEdgePoint(toNode, fromCenter)

    const angle = Math.atan2(toPoint.y - fromPoint.y, toPoint.x - fromPoint.x)
    const arrowSize = 8
    const arrow = [
      [toPoint.x, toPoint.y],
      [toPoint.x - arrowSize * Math.cos(angle - Math.PI / 6), toPoint.y - arrowSize * Math.sin(angle - Math.PI / 6)],
      [toPoint.x - arrowSize * Math.cos(angle + Math.PI / 6), toPoint.y - arrowSize * Math.sin(angle + Math.PI / 6)]
    ].join(' ')

    positions[edge.id] = {
      x1: fromPoint.x,
      y1: fromPoint.y,
      x2: toPoint.x,
      y2: toPoint.y,
      arrow
    }
  })
  return positions
})

function getNodePath(shape) {
  const w = props.nodeSize.width
  const h = props.nodeSize.height
  const paths = {
    rect: `M0,0 h${w} v${h} h${-w} z`,
    rounded: `M10,0 h${w-20} a10,10 0 0 1 10,10 v${h-20} a10,10 0 0 1 -10,10 h${-(w-20)} a10,10 0 0 1 -10,-10 v${-(h-20)} a10,10 0 0 1 10,-10`,
    stadium: `M${h/2},0 h${w-h} a${h/2},${h/2} 0 0 1 0,${h} h${-(w-h)} a${h/2},${h/2} 0 0 1 0,${-h}`,
    diamond: `M${w/2},0 l${w/2},${h/2} l${-w/2},${h/2} l${-w/2},${-h/2} z`,
    circle: {
      r: Math.min(w, h) / 2,
      cx: w / 2,
      cy: h / 2
    },
    hexagon: `M20,0 h${w-40} l20,${h/2} l${-20},${h/2} h${-(w-40)} l-20,${-h/2} z`,
    parallelogram: `M20,0 h${w-20} l${-20},${h} h${-(w-20)} z`,
    trapezoid: `M20,0 h${w-40} l20,${h} h${-w} z`
  }
  return paths[shape] || paths.rect
}

function getEdgePoint(node, target) {
  const cx = node.x + props.nodeSize.width / 2
  const cy = node.y + props.nodeSize.height / 2
  const hw = props.nodeSize.width / 2
  const hh = props.nodeSize.height / 2

  const dx = target.x - cx
  const dy = target.y - cy

  if (node.shape === 'circle') {
    const r = Math.min(hw, hh)
    const angle = Math.atan2(dy, dx)
    return { x: cx + r * Math.cos(angle), y: cy + r * Math.sin(angle) }
  }

  if (node.shape === 'diamond') {
    const angle = Math.atan2(dy, dx)
    const tan = hh / hw
    let px, py
    if (Math.abs(Math.tan(angle)) <= tan) {
      px = dx > 0 ? hw : -hw
      py = px * Math.tan(angle)
    } else {
      py = dy > 0 ? hh : -hh
      px = py / Math.tan(angle)
    }
    return { x: cx + px, y: cy + py }
  }

  const scaleFactor = Math.min(hw / Math.abs(dx || 1), hh / Math.abs(dy || 1))
  return { x: cx + dx * scaleFactor, y: cy + dy * scaleFactor }
}

function onDrop(e) {
  const shape = e.dataTransfer.getData('shape')
  if (shape) {
    const rect = wrapperRef.value.getBoundingClientRect()
    // 考虑缩放和平移
    const x = (e.clientX - rect.left - pan.x) / scale.value - props.nodeSize.width / 2
    const y = (e.clientY - rect.top - pan.y) / scale.value - props.nodeSize.height / 2
    emit('add-node', { shape, x: Math.max(10, x), y: Math.max(10, y) })
  }
}

function onNodeMouseDown(e, node) {
  if (props.mode === 'connect') {
    if (connectingFrom.value && connectingFrom.value !== node) {
      emit('add-edge', { from: connectingFrom.value.id, to: node.id })
      connectingFrom.value = null
      hint.value = '连线模式：点击起始节点，再点击目标节点'
    } else {
      connectingFrom.value = node
      hint.value = `已选择 ${node.id}，点击目标节点完成连线`
    }
  } else {
    selectNode(node)
    startDrag(e, node)
  }
}

function onNodeDoubleClick(node) {
  emit('edit-node', node)
}

function selectNode(node) {
  selectedNode.value = node
  emit('select-node', node)
}

function selectEdge(edge) {
  selectedNode.value = null
  emit('select-edge', edge)
}

function selectSubgraph(sg) {
  selectedNode.value = null
  emit('select-subgraph', props.subgraphs.find(s => s.id === sg.id))
}

// 子图拖拽
const dragSubgraphId = ref(null)
const dragSubgraphStartMouse = ref({ x: 0, y: 0 })
const dragSubgraphStartPositions = ref({})

function onSubgraphMouseDown(e, sg) {
  if (props.mode !== 'select') return
  e.preventDefault()

  const rect = wrapperRef.value.getBoundingClientRect()
  dragSubgraphId.value = sg.id
  dragSubgraphStartMouse.value = {
    x: (e.clientX - rect.left - pan.x) / scale.value,
    y: (e.clientY - rect.top - pan.y) / scale.value
  }

  // 记录子图内所有节点的初始位置
  const positions = {}
  props.nodes.forEach(n => {
    if (n.subgraph === sg.id) {
      positions[n.id] = { x: n.x, y: n.y }
    }
  })
  dragSubgraphStartPositions.value = positions

  document.addEventListener('mousemove', onSubgraphDrag)
  document.addEventListener('mouseup', endSubgraphDrag)
}

function onSubgraphDrag(e) {
  if (!dragSubgraphId.value) return

  const rect = wrapperRef.value.getBoundingClientRect()
  const mouseX = (e.clientX - rect.left - pan.x) / scale.value
  const mouseY = (e.clientY - rect.top - pan.y) / scale.value

  const dx = mouseX - dragSubgraphStartMouse.value.x
  const dy = mouseY - dragSubgraphStartMouse.value.y

  emit('move-subgraph', {
    id: dragSubgraphId.value,
    dx: Math.round(dx),
    dy: Math.round(dy),
    startPositions: dragSubgraphStartPositions.value
  })
}

function endSubgraphDrag() {
  dragSubgraphId.value = null
  dragSubgraphStartPositions.value = {}
  document.removeEventListener('mousemove', onSubgraphDrag)
  document.removeEventListener('mouseup', endSubgraphDrag)
}

function startDrag(e, node) {
  e.preventDefault()
  const rect = wrapperRef.value.getBoundingClientRect()
  dragNodeId.value = node.id
  dragOffset.value = {
    x: (e.clientX - rect.left - pan.x) / scale.value - node.x,
    y: (e.clientY - rect.top - pan.y) / scale.value - node.y
  }
  document.addEventListener('mousemove', onDrag)
  document.addEventListener('mouseup', endDrag)
}

function onDrag(e) {
  if (!dragNodeId.value) return
  const currentNode = props.nodes.find(n => n.id === dragNodeId.value)
  if (!currentNode) return

  const rect = wrapperRef.value.getBoundingClientRect()
  const newX = Math.max(0, (e.clientX - rect.left - pan.x) / scale.value - dragOffset.value.x)
  const newY = Math.max(0, (e.clientY - rect.top - pan.y) / scale.value - dragOffset.value.y)
  emit('update-node', { ...currentNode, x: Math.round(newX), y: Math.round(newY) })
}

function endDrag() {
  dragNodeId.value = null
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', endDrag)
}

// 缩放函数
function onWheel(e) {
  e.preventDefault()
  const delta = e.deltaY > 0 ? -ZOOM_STEP : ZOOM_STEP
  const newScale = Math.max(MIN_SCALE, Math.min(MAX_SCALE, scale.value + delta))

  // 以鼠标位置为中心缩放
  const rect = wrapperRef.value.getBoundingClientRect()
  const mouseX = e.clientX - rect.left
  const mouseY = e.clientY - rect.top

  const scaleRatio = newScale / scale.value
  pan.x = mouseX - (mouseX - pan.x) * scaleRatio
  pan.y = mouseY - (mouseY - pan.y) * scaleRatio

  scale.value = newScale
}

function zoomIn() {
  const newScale = Math.min(MAX_SCALE, scale.value + ZOOM_STEP)
  const rect = wrapperRef.value.getBoundingClientRect()
  const centerX = rect.width / 2
  const centerY = rect.height / 2

  const scaleRatio = newScale / scale.value
  pan.x = centerX - (centerX - pan.x) * scaleRatio
  pan.y = centerY - (centerY - pan.y) * scaleRatio

  scale.value = newScale
}

function zoomOut() {
  const newScale = Math.max(MIN_SCALE, scale.value - ZOOM_STEP)
  const rect = wrapperRef.value.getBoundingClientRect()
  const centerX = rect.width / 2
  const centerY = rect.height / 2

  const scaleRatio = newScale / scale.value
  pan.x = centerX - (centerX - pan.x) * scaleRatio
  pan.y = centerY - (centerY - pan.y) * scaleRatio

  scale.value = newScale
}

function resetZoom() {
  scale.value = 1
  pan.x = 0
  pan.y = 0
}

// 画布平移
let mouseDownTime = 0
let mouseDownPos = { x: 0, y: 0 }

function onCanvasMouseDown(e) {
  // 中键(1) 或 空格+左键(0) 或 左键在空白区域
  if (e.button === 1 || (e.button === 0 && spacePressed.value)) {
    isPanning.value = true
    panStart.x = e.clientX - pan.x
    panStart.y = e.clientY - pan.y
    e.preventDefault()
  } else if (e.button === 0 && e.target === svgRef.value) {
    // 左键在空白区域按下，记录时间和位置用于判断是点击还是拖拽
    mouseDownTime = Date.now()
    mouseDownPos = { x: e.clientX, y: e.clientY }
    isPanning.value = true
    panStart.x = e.clientX - pan.x
    panStart.y = e.clientY - pan.y
  }
}

function onCanvasMouseMove(e) {
  if (isPanning.value) {
    pan.x = e.clientX - panStart.x
    pan.y = e.clientY - panStart.y
  }
}

function onCanvasMouseUp(e) {
  if (e.button === 1 || (e.button === 0 && isPanning.value)) {
    // 如果是左键在空白区域，判断是点击还是拖拽
    if (e.button === 0 && e.target === svgRef.value) {
      const moved = Math.abs(e.clientX - mouseDownPos.x) > 5 || Math.abs(e.clientY - mouseDownPos.y) > 5
      const isClick = Date.now() - mouseDownTime < 200 && !moved

      if (isClick) {
        // 短时间没移动 = 点击，取消选择
        selectedNode.value = null
        emit('select-node', null)
      }
    }
    isPanning.value = false
  }
}

// 键盘事件处理
function onKeyDown(e) {
  if (e.code === 'Space' && !spacePressed.value) {
    spacePressed.value = true
    document.body.style.cursor = 'grab'
  }
}

function onKeyUp(e) {
  if (e.code === 'Space') {
    spacePressed.value = false
    document.body.style.cursor = ''
  }
}

// 点击空白取消选择
onMounted(() => {
  svgRef.value.addEventListener('click', (e) => {
    if (e.target === svgRef.value) {
      selectedNode.value = null
      emit('select-node', null)
    }
  })
  document.addEventListener('keydown', onKeyDown)
  document.addEventListener('keyup', onKeyUp)
})

onUnmounted(() => {
  document.removeEventListener('keydown', onKeyDown)
  document.removeEventListener('keyup', onKeyUp)
})

// 监听模式变化
watch(() => props.mode, (newMode) => {
  connectingFrom.value = null
  hint.value = newMode === 'select'
    ? '拖拽形状到画布 | 滚轮缩放 | 空白处拖拽画布'
    : '连线模式：点击起始节点，再点击目标节点'
})

defineExpose({ selectedNode })
</script>

<style scoped>
.canvas-wrapper {
  flex: 1;
  position: relative;
  overflow: hidden;
  background: #fafafa;
  background-image:
    linear-gradient(#e0e0e0 1px, transparent 1px),
    linear-gradient(90deg, #e0e0e0 1px, transparent 1px);
  background-size: 20px 20px;
  cursor: grab;
}
.canvas-wrapper:active {
  cursor: grabbing;
}
#canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
.zoom-controls {
  position: absolute;
  bottom: 50px;
  right: 10px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  background: white;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  padding: 6px;
}
.zoom-controls button {
  width: 32px;
  height: 28px;
  border: 1px solid #ddd;
  background: white;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  color: #333;
  display: flex;
  align-items: center;
  justify-content: center;
}
.zoom-controls button:hover {
  background: #f0f0f0;
}
.zoom-level {
  font-size: 11px;
  color: #666;
  text-align: center;
  padding: 2px 0;
  border-top: 1px solid #eee;
  border-bottom: 1px solid #eee;
}
.hint {
  position: absolute;
  bottom: 10px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0,0,0,0.7);
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  font-size: 12px;
  pointer-events: none;
}
.node-group {
  user-select: none;
  pointer-events: all;
}
.node-shape {
  pointer-events: all;
}
.edge-group {
  cursor: pointer;
}
.subgraph-group {
  cursor: move;
  pointer-events: all;
}
</style>