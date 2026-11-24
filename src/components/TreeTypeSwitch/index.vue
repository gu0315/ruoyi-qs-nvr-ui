<template>
  <div class="tree-type-switch" role="tablist">
    <span
        v-for="opt in options"
        :key="String(opt.value)"
        class="tts-item"
        :class="{ active: modelValue === opt.value }"
        role="tab"
        :aria-selected="modelValue === opt.value"
        @click="select(opt.value)"
    >
      <el-icon><component :is="opt.icon" /></el-icon>
      {{ opt.label }}
    </span>
  </div>
</template>

<script setup lang="ts" name="TreeTypeSwitch">
/**
 * 设备树类型切换：业务分组 / 行政区划
 * v-model 为 true 表示行政区划，false 表示业务分组（与原 el-switch 的 showRegion 保持一致）
 *
 * 可通过 CSS 变量适配深色背景：
 *   --tts-bg / --tts-color / --tts-active-bg / --tts-active-color
 */
import { FolderOpened, Location } from '@element-plus/icons-vue'

const props = defineProps<{ modelValue: boolean }>()
const emit = defineEmits<{
  (e: 'update:modelValue', value: boolean): void
  (e: 'change', value: boolean): void
}>()

const options = [
  { label: '业务分组', value: false, icon: FolderOpened },
  { label: '行政区划', value: true, icon: Location }
]

function select(value: boolean) {
  if (value === props.modelValue) return
  emit('update:modelValue', value)
  emit('change', value)
}
</script>

<style scoped lang="scss">
.tree-type-switch {
  display: flex;
  width: 100%;
  padding: 3px;
  border-radius: 8px;
  background: var(--tts-bg, var(--el-fill-color-light));
  box-sizing: border-box;
  container-type: inline-size;
}

.tts-item {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 28px;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 400;
  white-space: nowrap;
  color: var(--tts-color, var(--el-text-color-regular));
  cursor: pointer;
  user-select: none;
  transition: all 0.2s;

  .el-icon {
    font-size: 14px;
  }

  &:hover:not(.active) {
    color: var(--el-color-primary);
  }

  &.active {
    background: var(--tts-active-bg, var(--el-bg-color));
    color: var(--tts-active-color, var(--el-color-primary));
    font-weight: 500;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
  }
}

/* 侧栏较窄时：先隐藏图标，再缩小字号 */
@container (max-width: 200px) {
  .tts-item .el-icon {
    display: none;
  }
}

@container (max-width: 130px) {
  .tts-item {
    font-size: 12px;
    gap: 0;
  }
}

html.dark .tree-type-switch {
  background: var(--tts-bg, rgba(255, 255, 255, 0.06));
}

html.dark .tts-item.active {
  background: var(--tts-active-bg, rgba(255, 255, 255, 0.12));
}
</style>
