<template>
  <div class="container">
    <!-- 左侧形状工具栏 -->
    <div class="sidebar">
      <div class="sidebar-header">形状工具</div>
      <div class="shape-list">
        <div
          v-for="shape in shapes"
          :key="shape.type"
          class="shape-item"
          draggable="true"
          @dragstart="onDragStart($event, shape.type)"
        >
          <svg width="40" height="28" v-html="shape.svg"></svg>
          <span class="shape-name">{{ shape.name }}</span>
        </div>
      </div>
      <div class="sidebar-actions">
        <button class="btn-success" @click="$emit('generate')">生成 Mermaid</button>
        <button class="btn-secondary" @click="$emit('clear')">清空画布</button>
        <button class="btn-secondary" @click="$emit('load-example')">加载示例</button>
      </div>
    </div>
  </div>
</template>

<script setup>
const shapes = [
  { type: 'rect', name: '矩形', svg: '<rect x="2" y="2" width="36" height="24" fill="#e3f2fd" stroke="#1976d2" stroke-width="2"/>' },
  { type: 'rounded', name: '圆角矩形', svg: '<rect x="2" y="2" width="36" height="24" rx="6" fill="#e8f5e9" stroke="#4caf50" stroke-width="2"/>' },
  { type: 'stadium', name: '体育场形', svg: '<rect x="2" y="2" width="36" height="24" rx="12" fill="#fff3e0" stroke="#ff9800" stroke-width="2"/>' },
  { type: 'diamond', name: '菱形', svg: '<polygon points="20,2 38,14 20,26 2,14" fill="#fce4ec" stroke="#e91e63" stroke-width="2"/>' },
  { type: 'circle', name: '圆形', svg: '<circle cx="20" cy="14" r="12" fill="#f3e5f5" stroke="#9c27b0" stroke-width="2"/>' },
  { type: 'hexagon', name: '六边形', svg: '<polygon points="10,2 30,2 38,14 30,26 10,26 2,14" fill="#e0f7fa" stroke="#00bcd4" stroke-width="2"/>' },
  { type: 'parallelogram', name: '平行四边形', svg: '<polygon points="8,2 38,2 32,26 2,26" fill="#fff8e1" stroke="#ffc107" stroke-width="2"/>' },
  { type: 'trapezoid', name: '梯形', svg: '<polygon points="10,2 30,2 38,26 2,26" fill="#e8eaf6" stroke="#3f51b5" stroke-width="2"/>' }
]

function onDragStart(e, shapeType) {
  e.dataTransfer.setData('shape', shapeType)
}
</script>

<style scoped>
.sidebar {
  width: 200px;
  background: #fff;
  border-right: 1px solid #ddd;
  display: flex;
  flex-direction: column;
}
.sidebar-header {
  padding: 12px;
  background: #1976d2;
  color: white;
  font-weight: 500;
}
.shape-list {
  flex: 1;
  padding: 10px;
  overflow-y: auto;
}
.shape-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  margin-bottom: 8px;
  cursor: grab;
  transition: all 0.2s;
  background: #fafafa;
}
.shape-item:hover {
  border-color: #1976d2;
  background: #e3f2fd;
}
.shape-item svg {
  flex-shrink: 0;
}
.shape-item .shape-name {
  font-size: 12px;
  color: #666;
}
.sidebar-actions {
  padding: 10px;
  border-top: 1px solid #ddd;
}
.sidebar-actions button {
  width: 100%;
  padding: 10px;
  margin-bottom: 8px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 13px;
}
.btn-primary {
  background: #1976d2;
  color: white;
}
.btn-secondary {
  background: #757575;
  color: white;
}
.btn-success {
  background: #4caf50;
  color: white;
}
</style>