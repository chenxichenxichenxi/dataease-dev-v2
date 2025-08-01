<script lang="ts" setup>
import { keycodes } from '@/utils/DeShortcutKey.js'
import eventBus from '@/utils/eventBus'
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { toRefs } from 'vue'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import { storeToRefs } from 'pinia'

const canEdit = ref(false)
const ctrlKey = ref(17)
const isCtrlDown = ref(false)

const emit = defineEmits(['input'])
const text = ref(null)
const textOut = ref(null)
const scrollScale0 = ref('100%')
const scrollScale100 = ref('100%')
let timeId = null
const textOutClientWidth = ref(200)

const props = defineProps({
  propValue: {
    type: String,
    required: true,
    default: ''
  },
  element: {
    type: Object,
    default() {
      return {
        id: null,
        propValue: ''
      }
    }
  },
  showPosition: {
    required: false,
    type: String,
    default: 'preview'
  }
})

const { element, showPosition } = toRefs(props)
const dvMainStore = dvMainStoreWithOut()
const { editMode, curComponent } = storeToRefs(dvMainStore)

const onComponentClick = () => {
  if (curComponent.value.id !== element.value.id) {
    canEdit.value = false
  }
}

const handleInput = e => {
  emit('input', element.value, e.target.innerHTML)
}

const handleKeydown = e => {
  // 阻止冒泡，防止触发复制、粘贴组件操作
  canEdit.value && e.stopPropagation()
  if (e.keyCode == ctrlKey.value) {
    isCtrlDown.value = true
  } else if (isCtrlDown.value && canEdit.value && keycodes.includes(e.keyCode)) {
    e.stopPropagation()
  } else if (e.keyCode == 46) {
    // deleteKey
    e.stopPropagation()
  }
}

const handleKeyup = e => {
  // 阻止冒泡，防止触发复制、粘贴组件操作
  canEdit.value && e.stopPropagation()
  if (e.keyCode == ctrlKey.value) {
    isCtrlDown.value = false
  }
}

const handleMousedown = e => {
  if (canEdit.value) {
    e.stopPropagation()
  }
}

const clearStyle = e => {
  e.preventDefault()
  const clp = e.clipboardData
  const text = clp.getData('text/plain') || ''
  if (text !== '') {
    document.execCommand('insertText', false, text)
  }
}

const handleBlur = e => {
  element.value.propValue = e.target.innerHTML || ''
  const html = e.target.innerHTML
  if (html !== '') {
    element.value.propValue = e.target.innerHTML
  } else {
    element.value.propValue = ''
    nextTick(function () {
      element.value.propValue = ''
    })
  }
  canEdit.value = false
}

const marqueeTxt = computed(
  () => !canEdit.value && !element.value['resizing'] && !element.value['dragging']
)

const setEdit = () => {
  if (['canvas', 'canvasDataV', 'edit'].includes(showPosition.value) && !element.value['isLock']) {
    canEdit.value = true
    // 全选
    selectText(text.value)
  }
}
const selectText = element => {
  const selection = window.getSelection()
  const range = document.createRange()
  range.selectNodeContents(element)
  selection.removeAllRanges()
  selection.addRange(range)
}
onBeforeUnmount(() => {
  eventBus.off('componentClick', onComponentClick)
  if (timeId) {
    clearInterval(timeId)
    timeId = null
  }
})

// 滚动速度已px为单位 注意 这里的总宽度是还原到缩放前的 这样不同缩放比例下的跑马灯视觉上滚动速度（按照比例）一致
const varStyle = computed(() => [
  {
    '--scroll-speed': `${
      element.value.style.scrollSpeed === 0 || !textOut.value
        ? 0
        : (textOutClientWidth.value + text.value.clientWidth) / element.value.style.scrollSpeed
    }s`,
    '--scroll-scale0': `${scrollScale0.value}`,
    '--scroll-scale100': `${scrollScale100.value}`
  }
])

const init = () => {
  timeId = setInterval(() => {
    const outerId = ['canvas', 'canvasDataV', 'edit'].includes(showPosition.value)
      ? 'shape-id-' + element.value.id
      : 'wrapper-outer-id-' + element.value.id
    const componentOut = document.getElementById(outerId)
    if (componentOut && text.value) {
      const textValue = text.value.clientWidth
      textOutClientWidth.value = componentOut.clientWidth
      scrollScale0.value = (textOutClientWidth.value * 100) / textValue + '%'
      scrollScale100.value = '-100%'
    } else {
      scrollScale0.value = '100%'
      scrollScale100.value = '-100%'
    }
  }, 1000)
}
onMounted(() => {
  init()
})
</script>

<template>
  <div
    v-if="editMode == 'edit'"
    :style="varStyle"
    class="v-text"
    ref="textOut"
    @keydown="handleKeydown"
    @keyup="handleKeyup"
    @dblclick="setEdit"
  >
    <div
      ref="text"
      :contenteditable="canEdit"
      :class="{ 'can-edit': canEdit, 'marquee-txt': marqueeTxt }"
      tabindex="0"
      @paste="clearStyle"
      @mousedown="handleMousedown"
      @blur="handleBlur"
      @input="handleInput"
      v-html="element['propValue']"
    ></div>
  </div>
  <div v-else class="v-text preview" ref="textOut" :style="varStyle">
    <div class="marquee-txt" ref="text" v-html="element['propValue']"></div>
  </div>
</template>

<style lang="less" scoped>
.v-text {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  div {
    outline: none;
    word-break: break-all;
    padding: 4px;
    white-space: nowrap;
  }

  .can-edit {
    cursor: text;
    width: 100%;
    display: grid;
    align-items: center;
    height: 100%;
  }
}

.preview {
  user-select: none;
}

.marquee {
  margin-left: 100px;
  width: 300px;
  white-space: nowrap;
  overflow: hidden;
  border: 1px solid #4c7cee;
}
.marquee-txt {
  display: inline-block;
  animation: marqueeAnimation var(--scroll-speed) linear infinite;
}
@keyframes marqueeAnimation {
  0% {
    transform: translate(var(--scroll-scale0), 0);
  }
  100% {
    transform: translate(var(--scroll-scale100), 0);
  }
}
</style>
