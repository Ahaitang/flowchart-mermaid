<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="visible" class="modal-overlay" @click.self="cancel">
        <div class="modal-container">
          <div class="modal-header">
            <span class="modal-title">{{ title }}</span>
            <button class="modal-close" @click="cancel">×</button>
          </div>
          <div class="modal-body">
            <div v-if="type === 'message'" class="modal-message">
              <span class="modal-icon" :class="iconType">{{ icon }}</span>
              <span>{{ message }}</span>
            </div>
            <div v-else-if="type === 'input'" class="modal-input-group">
              <label v-if="label">{{ label }}</label>
              <input
                ref="inputRef"
                v-model="inputValue"
                type="text"
                class="modal-input"
                :placeholder="placeholder"
                @keyup.enter="confirm"
              >
            </div>
          </div>
          <div class="modal-footer">
            <button v-if="type === 'input'" class="modal-btn secondary" @click="cancel">取消</button>
            <button class="modal-btn primary" @click="confirm">{{ type === 'input' ? '确定' : '好的' }}</button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, watch, nextTick, computed } from 'vue'

const props = defineProps({
  visible: Boolean,
  type: { type: String, default: 'message' }, // 'message' | 'input'
  title: { type: String, default: '提示' },
  message: { type: String, default: '' },
  label: { type: String, default: '' },
  placeholder: { type: String, default: '' },
  defaultValue: { type: String, default: '' },
  messageType: { type: String, default: 'info' } // 'info' | 'success' | 'warning' | 'error'
})

const emit = defineEmits(['update:visible', 'confirm', 'cancel'])

const inputValue = ref('')
const inputRef = ref(null)

const iconType = computed(() => props.messageType)
const icon = computed(() => {
  const icons = {
    info: 'ℹ️',
    success: '✓',
    warning: '⚠',
    error: '✕'
  }
  return icons[props.messageType] || icons.info
})

watch(() => props.visible, (val) => {
  if (val) {
    inputValue.value = props.defaultValue
    if (props.type === 'input') {
      nextTick(() => {
        inputRef.value?.focus()
        inputRef.value?.select()
      })
    }
  }
})

function confirm() {
  if (props.type === 'input') {
    emit('confirm', inputValue.value)
  } else {
    emit('confirm')
  }
  emit('update:visible', false)
}

function cancel() {
  emit('cancel')
  emit('update:visible', false)
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.modal-container {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  min-width: 320px;
  max-width: 480px;
  overflow: hidden;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.modal-title {
  font-size: 16px;
  font-weight: 600;
}

.modal-close {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  font-size: 18px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.modal-close:hover {
  background: rgba(255, 255, 255, 0.3);
}

.modal-body {
  padding: 24px 20px;
}

.modal-message {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 14px;
  color: #333;
}

.modal-icon {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  flex-shrink: 0;
}

.modal-icon.info {
  background: #e3f2fd;
  color: #1976d2;
}

.modal-icon.success {
  background: #e8f5e9;
  color: #4caf50;
}

.modal-icon.warning {
  background: #fff3e0;
  color: #ff9800;
}

.modal-icon.error {
  background: #ffebee;
  color: #f44336;
}

.modal-input-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.modal-input-group label {
  font-size: 13px;
  color: #666;
}

.modal-input {
  width: 100%;
  padding: 12px 14px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 14px;
  transition: border-color 0.2s;
  outline: none;
}

.modal-input:focus {
  border-color: #667eea;
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 16px 20px;
  background: #f9f9f9;
  border-top: 1px solid #eee;
}

.modal-btn {
  padding: 10px 24px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.modal-btn.primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.modal-btn.primary:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.modal-btn.secondary {
  background: #fff;
  color: #666;
  border: 1px solid #ddd;
}

.modal-btn.secondary:hover {
  background: #f5f5f5;
}

/* Transition */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.9);
}
</style>