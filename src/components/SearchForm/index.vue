<!--
  列表页筛选区，按《B端后台通用规范》§4.4 / §4.9（docs/design-spec/B端后台通用规范.md，补能平台 2.0）
  移植自 home-pile-manage/src/components/SearchForm，补充了 plain 模式（弹窗内使用宿主容器，不画卡片）。

  - 按容器宽度自适应 1～4 列；标签右对齐、按比例分宽、不带冒号、不加粗；控件占满所在列
  - 默认收起：最多显示两行，筛选项多于两行时出现「更多筛选」；收起只隐藏、不清空已填的值
  - 按钮组（更多筛选/收起、重置、搜索）跟在最后一个显示的筛选项后面：
    这一行还有空列就放在这一行最右边，排满了就另起一行最右边
  - 筛选卡：白底、圆角 12、内边距 24，与下方内容间距 16；可在「布局设置 → 搜索区卡片」开启，默认关闭

  用法：
    <search-form ref="queryRef" :model="queryParams" v-show="showSearch" @search="handleQuery" @reset="resetQuery">
      <el-form-item label="设备名称" prop="deviceName"><el-input v-model="queryParams.deviceName" placeholder="请输入设备名称" clearable /></el-form-item>
    </search-form>
  点「重置」时组件先调用 resetFields()，再触发 reset 事件；页面原来调用 proxy.resetForm('queryRef') 也照常可用。
-->
<template>
  <div ref="rootRef" class="search-form" :class="{ 'search-form--plain': !isCard }" :style="rootStyle">
    <el-form ref="formRef" :model="model" label-position="right" v-bind="$attrs" @submit.prevent @keyup.enter="handleSearch">
      <div class="search-form__grid">
        <component
          :is="node"
          v-for="(node, index) in getItems()"
          v-show="expanded || index < visibleCount()"
          :key="node.key ?? index"
        />
        <div class="search-form__actions">
          <el-button v-if="hasMore()" link type="primary" class="search-form__more" @click="expanded = !expanded">
            {{ expanded ? '收起' : '更多筛选' }}
            <el-icon class="search-form__arrow"><ArrowUp v-if="expanded" /><ArrowDown v-else /></el-icon>
          </el-button>
          <slot name="actions">
            <el-button class="search-form__reset" @click="handleReset">重置</el-button>
            <el-button type="primary" :loading="loading" @click="handleSearch">搜索</el-button>
          </slot>
        </div>
      </div>
    </el-form>
  </div>
</template>

<script setup>
import { computed, ref, useSlots, Fragment, Comment, Text } from 'vue'
import { useElementSize } from '@vueuse/core'
import { ArrowDown, ArrowUp } from '@element-plus/icons-vue'
import useSettingsStore from '@/store/modules/settings'

defineOptions({ name: 'SearchForm', inheritAttrs: false })

const props = defineProps({
  model: { type: Object, required: true },
  loading: { type: Boolean, default: false },
  defaultExpanded: { type: Boolean, default: false },
  /** 弹窗、侧栏等宿主容器内使用：不画筛选卡（无背景、无内边距） */
  plain: { type: Boolean, default: false }
})
const emit = defineEmits(['search', 'reset'])
const settingsStore = useSettingsStore()
// 卡片样式：布局设置「搜索区卡片」开关控制（默认关闭）；弹窗内的 plain 模式始终不画卡片
const isCard = computed(() => !props.plain && settingsStore.searchCard !== false)
const slots = useSlots()

const rootRef = ref()
const formRef = ref()
const expanded = ref(props.defaultExpanded)
const { width } = useElementSize(rootRef)

// §4.4.2 断点：按筛选区自身内容宽度（卡片模式扣除左右内边距 24×2）
const layout = computed(() => {
  const w = (width.value || 1200) - (isCard.value ? 48 : 0)
  if (w >= 1600) return { cols: 4, label: 8 }
  if (w > 992) return { cols: 3, label: 6 }
  if (w > 768) return { cols: 3, label: 8 }
  if (w > 576) return { cols: 2, label: 7 }
  return { cols: 1, label: 4 }
})
// 标签宽度占一列的比例（24 份制）
const rootStyle = computed(() => ({
  '--search-cols': layout.value.cols,
  '--search-label': `${(layout.value.label / 24) * 100}%`
}))
// 每个筛选项占一列，默认显示两行
function visibleCount() {
  return layout.value.cols * 2
}

function flatten(nodes) {
  const result = []
  for (const node of nodes || []) {
    if (node.type === Fragment) result.push(...flatten(node.children))
    else if (node.type === Comment) continue
    else if (node.type === Text && !String(node.children).trim()) continue
    else result.push(node)
  }
  return result
}
function getItems() {
  return flatten(slots.default?.())
}
function hasMore() {
  return getItems().length > visibleCount()
}

function handleSearch() {
  emit('search')
}
function handleReset() {
  formRef.value?.resetFields()
  emit('reset')
}

defineExpose({
  formRef,
  resetFields: (...args) => formRef.value?.resetFields(...args),
  clearValidate: (...args) => formRef.value?.clearValidate(...args),
  validate: (...args) => formRef.value?.validate(...args),
  validateField: (...args) => formRef.value?.validateField(...args)
})
</script>

<style lang="scss" scoped>
// 筛选卡：白底、圆角 12、内边距 24，与下方内容间距 16（§4.9.2 / §4.9.4）
.search-form {
  // 控件和按钮统一高度（规范为 32，本项目按需求加高到 36）
  --search-control-h: 36px;
  box-sizing: border-box;
  margin-bottom: 16px;
  padding: 24px;
  border: 1px solid var(--el-border-color-lighter);
  border-radius: 12px;
  background: var(--el-bg-color);
  // 项目里不少页面背景是白色，只用浅边框区分卡片，不加阴影
  container-type: inline-size;
}

.search-form--plain {
  padding: 0;
  border: none;
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.search-form :deep(> .el-form) {
  margin: 0 !important;
  padding: 0;
}

// 行间距 16、列间距为容器宽度的 2%（16～40）
.search-form__grid {
  // Element Plus 输入框、下拉框、日期框的高度都读这个变量
  --el-component-size: var(--search-control-h);
  display: grid;
  grid-template-columns: repeat(var(--search-cols), minmax(0, 1fr));
  row-gap: 16px;
  column-gap: clamp(16px, 2cqi, 40px);
}

.search-form__grid :deep(> .el-form-item) {
  display: flex;
  min-width: 0;
  margin: 0;
}

// 标签：14/22、400、右对齐、不带冒号、按比例分宽，放不下显示省略号
.search-form__grid :deep(> .el-form-item > .el-form-item__label),
.search-form__grid :deep(> .el-form-item > .el-form-item__label-wrap > .el-form-item__label) {
  display: block;
  flex: 0 0 auto;
  box-sizing: border-box;
  width: var(--search-label) !important;
  min-width: 0;
  height: var(--search-control-h);
  padding: 0 8px 0 0;
  overflow: hidden;
  color: var(--el-text-color-primary);
  font-size: 14px;
  font-weight: 400;
  line-height: var(--search-control-h);
  text-align: right;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.search-form__grid :deep(> .el-form-item > .el-form-item__content) {
  flex: 1;
  min-width: 0;
  margin-left: 0 !important;
  line-height: var(--search-control-h);
}

// 没有标签的项（如单个复选框）和同列控件左对齐
.search-form__grid :deep(> .el-form-item:not(:has(> .el-form-item__label)) > .el-form-item__content) {
  padding-left: var(--search-label);
}

// 控件占满所在列，去掉页面里写死的宽度
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-input),
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-select),
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-select-v2),
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-cascader),
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-tree-select),
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-input-number),
.search-form__grid :deep(> .el-form-item > .el-form-item__content > .el-date-editor) {
  width: 100% !important;
}

// 控件：高 36、圆角 6
.search-form__grid :deep(.el-input__wrapper),
.search-form__grid :deep(.el-select__wrapper),
.search-form__grid :deep(.el-date-editor.el-input__wrapper) {
  min-height: var(--search-control-h);
  border-radius: 6px;
}

// 单选组：标签到第一个选项 10，选项之间 10（§4.4.1）
.search-form__grid :deep(> .el-form-item .el-radio-group) {
  gap: 10px;
  padding-left: 2px;
}
.search-form__grid :deep(> .el-form-item .el-radio) {
  margin-right: 0;
}

// 按钮组：放在最后一列，当前行放不下时自动到下一行；右边缘和最右一列控件右边框对齐
.search-form__actions {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  grid-column: -2 / -1;
  gap: 12px;
  min-width: 0;
  height: var(--search-control-h);
  white-space: nowrap;

  // 按钮：高 36、圆角 6、左右 15、14/22、400，不带图标（§4.1，高度按本项目调整）
  // 项目全局 element-ui.scss 给按钮加了渐变、阴影和悬停位移，这里按规范统一为扁平样式
  :deep(.el-button) {
    height: var(--search-control-h);
    margin-left: 0;
    padding: 0 15px;
    border-radius: 6px;
    box-shadow: none;
    font-size: 14px;
    font-weight: 400;
    transform: none;
  }

  // 「搜索」为主按钮：主题色底、白字
  :deep(.el-button--primary:not(.is-link)) {
    border-color: var(--el-color-primary);
    background: var(--el-color-primary);
    color: #fff;

    &:hover,
    &:focus-visible {
      border-color: var(--el-color-primary-light-3);
      background: var(--el-color-primary-light-3);
      color: #fff;
    }

    &:active {
      border-color: var(--el-color-primary-dark-2);
      background: var(--el-color-primary-dark-2);
    }
  }

  // 「重置」为辅按钮：白底、1px 主题色边框、主题色文字
  :deep(.search-form__reset) {
    border-color: var(--el-color-primary);
    background: var(--el-bg-color);
    color: var(--el-color-primary);

    &:hover,
    &:focus-visible {
      border-color: var(--el-color-primary);
      background: var(--el-color-primary-light-9);
      color: var(--el-color-primary);
    }

    &:active {
      border-color: var(--el-color-primary-dark-2);
      color: var(--el-color-primary-dark-2);
    }
  }
}

// 「更多筛选」：14px 主题色 + 间距 2 + 16px 箭头
.search-form__actions :deep(.search-form__more) {
  padding: 0;
  border: none;

  .search-form__arrow {
    margin-left: 2px;
    font-size: 16px;
  }
}
</style>
