<script setup lang="tsx">
import { reactive, ref } from 'vue'
import eventBus from '@/utils/eventBus'
import Icon from '@/components/icon-custom/src/Icon.vue'
import { CANVAS_MATERIAL } from '@/custom-component/common/ComponentConfig'
import { ElScrollbar } from 'element-plus-secondary'
import DeDecoration from '@/custom-component/de-decoration/Component.vue'

defineProps({
  propValue: {
    type: Array,
    default: () => []
  },
  element: {
    type: Object,
    default() {
      return {
        propValue: null
      }
    }
  }
})

const commonGroup = ref<InstanceType<typeof ElScrollbar>>()

const state = reactive({
  curCategory: 'CanvasBoard',
  groupList: CANVAS_MATERIAL
})

const scrollTo = offsetTop => {
  commonGroup?.value.setScrollTop(offsetTop)
}

const anchorPosition = anchor => {
  const element = document.querySelector(anchor)
  scrollTo(element.offsetTop)
}

const newComponent = ({ category, innerType }) => {
  eventBus.emit('handleNew', { componentName: category, innerType: innerType })
}

const handleDragStart = e => {
  e.dataTransfer.setData('id', e.target.dataset.id)
}

const groupActiveChange = category => {
  state.curCategory = category
  anchorPosition('#' + category)
}

const findUrl = name => {
  return new URL(`/src/assets/dynamic-background/${name}`, import.meta.url).href
}
</script>

<template>
  <el-row class="group" @dragstart="handleDragStart">
    <div class="group-left">
      <ul class="ul-custom">
        <li
          class="li-custom"
          :class="{ 'li-custom-active': state.curCategory === groupInfo.category }"
          v-for="groupInfo in state.groupList"
          :key="groupInfo.category"
          @click="groupActiveChange(groupInfo.category)"
        >
          {{ groupInfo.title }}
        </li>
      </ul>
    </div>
    <el-scrollbar ref="commonGroup" class="group-right" height="392px">
      <el-row
        style="padding: 1px"
        :id="groupInfo.category"
        v-for="groupInfo in state.groupList"
        :key="groupInfo.title"
      >
        <el-col
          v-show="state.curCategory === groupInfo.category"
          :class="'item' + groupInfo.span"
          :span="groupInfo.span"
          v-for="chartInfo in groupInfo.details"
          :key="chartInfo.title"
        >
          <div
            v-on:click="newComponent({ category: groupInfo.category, innerType: chartInfo.value })"
            class="item-top"
            draggable="true"
            :data-id="groupInfo.category + '&' + chartInfo.value"
            :title="chartInfo.title"
          >
            <Icon
              v-if="['outer_svg', 'graphical'].includes(chartInfo.type)"
              class-name="item-top-icon"
              ><component class="svg-icon item-top-icon" :is="chartInfo.icon"></component
            ></Icon>
            <DeDecoration
              :curStyle="{ width: 530, height: 373 }"
              :element="{ innerType: chartInfo.value }"
              :scale="0.15"
              v-else-if="['de_decoration'].includes(chartInfo.type)"
            ></DeDecoration>
            <component v-else style="color: #a6a6a6" :is="chartInfo.icon"></component>
          </div>
          <div v-if="chartInfo.title" class="item-bottom">
            <span>{{ chartInfo.title }}</span>
          </div>
        </el-col>
      </el-row>
    </el-scrollbar>
  </el-row>
</template>

<style lang="less" scoped>
.group {
  display: flex;
  max-height: 400px;
  height: 100%;
  margin-top: 4px;
  .group-left {
    width: 100px;
    height: 100%;
    .ul-custom {
      padding-inline-start: 0px;
      color: @canvas-main-font-color;
      .li-custom {
        margin-top: 1px;
        font-weight: 400;
        font-size: 14px;
        line-height: 32px;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        list-style-type: none;
        list-style-position: inside;
        border-radius: 4px;
        padding-left: 8px;
        &:hover {
          background: rgba(255, 255, 255, 0.1);
          cursor: pointer;
        }
      }

      .li-custom a:hover {
        background: none;
      }

      .li-a {
        color: #1f2329;
      }
    }
  }
  .group-right {
    border-left: 1px solid @side-outline-border-color;
    flex: 1;
    padding: 4px 0 4px 12px;
  }
}

.li-custom-active {
  background: var(--ed-color-primary-1a, rgba(51, 112, 255, 0.1));
  color: var(--ed-color-primary) !important;
  .li-a {
    color: var(--ed-color-primary) !important;
  }
}

.item8 {
  margin-bottom: 12px;
  .item-top {
    width: 88px;
    height: 64px;
    background: #1a1a1a;
    padding: 4px;
    border-radius: 4px;
    cursor: pointer;
    &:hover {
      outline: 1px solid var(--ed-color-primary);
    }
    .item-top-icon {
      width: 80px;
      height: 56px;
      color: @canvas-main-font-color;
    }
  }
  .item-bottom {
    height: 20px;
    line-height: 20px;
    color: #a6a6a6;
    font-size: 12px;
    text-align: center;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    width: 88px;
  }
}

.item4 {
  margin-bottom: 12px;
  .item-top {
    width: 28px;
    height: 28px;
    border-radius: 4px;
    padding: 4px;
    cursor: pointer;
    &:hover {
      outline: 1px solid var(--ed-color-primary);
    }
    .item-top-icon {
      width: 20px;
      height: 20px;
      color: @canvas-main-font-color;
    }
  }
  .item-bottom {
    height: 20px;
    line-height: 20px;
    color: #a6a6a6;
    font-size: 12px;
    text-align: center;
  }
}
</style>
