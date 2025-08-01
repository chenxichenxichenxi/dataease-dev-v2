<script lang="tsx" setup>
import icon_sortAToZ_outlined from '@/assets/svg/icon_sort-a-to-z_outlined.svg'
import icon_sortZToA_outlined from '@/assets/svg/icon_sort-z-to-a_outlined.svg'
import icon_sort_outlined from '@/assets/svg/icon_sort_outlined.svg'
import icon_deleteTrash_outlined from '@/assets/svg/icon_delete-trash_outlined.svg'
import icon_down_outlined1 from '@/assets/svg/icon_down_outlined-1.svg'
import icon_right_outlined from '@/assets/svg/icon_right_outlined.svg'
import icon_done_outlined from '@/assets/svg/icon_done_outlined.svg'
import icon_edit_outlined from '@/assets/svg/icon_edit_outlined.svg'
import icon_visible_outlined from '@/assets/svg/icon_visible_outlined.svg'
import icon_invisible_outlined from '@/assets/svg/icon_invisible_outlined.svg'
import { useI18n } from '@/hooks/web/useI18n'
import { computed, onMounted, ref, toRefs, watch } from 'vue'
import { getItemType } from '@/views/chart/components/editor/drag-item/utils'
import { fieldType } from '@/utils/attr'
import { iconFieldMap } from '@/components/icon-group/field-list'

const { t } = useI18n()

const tagType = ref('success')
const showDateExt = ref(false)

const props = defineProps({
  param: {
    type: Object,
    required: false
  },
  item: {
    type: Object,
    required: true
  },
  index: {
    type: Number,
    required: true
  },
  chart: {
    type: Object,
    required: true
  },
  dimensionData: {
    type: Array,
    required: true
  },
  quotaData: {
    type: Array,
    required: true
  },
  themes: {
    type: String,
    default: 'dark'
  },
  type: {
    type: String,
    required: true
  }
})

const emit = defineEmits([
  'onDimensionItemRemove',
  'onCustomSort',
  'onDimensionItemChange',
  'onNameEdit',
  'valueFormatter',
  'onToggleHide',
  'editSortPriority'
])

const { item } = toRefs(props)
const toolTip = computed(() => {
  return props.themes || 'dark'
})
const showValueFormatter = computed<boolean>(() => {
  return (
    (props.chart.type === 'table-normal' || props.chart.type === 'table-info') &&
    (props.item.deType === 2 || props.item.deType === 3)
  )
})

watch(
  [() => props.dimensionData, () => props.item, () => props.chart.type],
  () => {
    getItemTagType()
  },
  { deep: true }
)

const clickItem = param => {
  if (!param) {
    return
  }
  switch (param.type) {
    case 'rename':
      showRename()
      break
    case 'remove':
      removeItem()
      break
    case 'filter':
      // editFilter()
      break
    case 'formatter':
      valueFormatter()
      break
    case 'toggleHide':
      toggleHide()
      break
    case 'sortPriority':
      emit('editSortPriority')
      break
    default:
      break
  }
}

const beforeClickItem = type => {
  return {
    type
  }
}

const sort = param => {
  if (param.type === 'custom_sort') {
    const item = {
      index: props.index,
      sort: param.type
    }
    emit('onCustomSort', item)
  } else {
    item.value.index = props.index
    item.value.sort = param.type
    item.value.customSort = []
    delete item.value.axisType
    emit('onDimensionItemChange', item.value)
  }
}

const beforeSort = type => {
  return {
    type: type
  }
}

const dateStyle = param => {
  item.value.dateStyle = param.type
  item.value.axisType = props.type
  emit('onDimensionItemChange', item.value)
}

const beforeDateStyle = type => {
  return {
    type: type
  }
}

const datePattern = param => {
  item.value.datePattern = param.type
  item.value.axisType = props.type
  emit('onDimensionItemChange', item.value)
}

const beforeDatePattern = type => {
  return {
    type: type
  }
}

const showRename = () => {
  item.value.index = props.index
  item.value.renameType = props.type
  // item.value.dsFieldName = getOriginFieldName(props.dimensionData, props.quotaData, item.value)
  emit('onNameEdit', item.value)
}

const removeItem = () => {
  item.value.index = props.index
  item.value.removeType = props.type
  emit('onDimensionItemRemove', item.value)
}

const getItemTagType = () => {
  tagType.value = getItemType(props.dimensionData, props.quotaData, props.item)
}

const valueFormatter = () => {
  item.value.index = props.index
  item.value.formatterType = props.type
  emit('valueFormatter', item.value)
}
const showCustomSort = item => {
  if (props.chart.type === 'symbolic-map' || props.chart.type === 'flow-map') {
    return false
  }
  return !item.chartId && (item.deType === 0 || item.deType === 5)
}

const NOT_SUPPORT_SORT = ['word-cloud', 'stock-line', 'treemap', 'circle-packing']
const showSort = computed(() => {
  const { type: chartType } = props.chart
  const { type: propType } = props
  const notShowSort = NOT_SUPPORT_SORT.includes(chartType)
  if (notShowSort || propType === 'extColor') {
    return false
  }
  const isChartMix = chartType.includes('chart-mix')
  const isDimensionType = ['dimension', 'dimensionStack', 'dimensionExt'].includes(propType)
  return !isChartMix || isDimensionType
})
const showSortPriority = computed(() => {
  if (props.chart.type.includes('chart-mix')) {
    return false
  }
  return showSort.value
})
const toggleHide = () => {
  item.value.index = props.index
  item.value.hide = !item.value.hide
  item.value.axisType = props.type
  emit('onToggleHide', item.value)
}
const showHideIcon = computed(() => {
  return ['table-info', 'table-normal'].includes(props.chart.type) && item.value.hide
})

onMounted(() => {
  getItemTagType()
})
</script>

<template>
  <span class="item-style">
    <el-dropdown :effect="themes" trigger="click" @command="clickItem">
      <el-tag
        class="item-axis father"
        :class="['editor-' + props.themes, `${themes}_icon-right`]"
        :style="{ backgroundColor: tagType + '0a', border: '1px solid ' + tagType }"
      >
        <span v-if="type !== 'extColor'" style="display: flex; color: #646a73">
          <el-icon v-if="'asc' === item.sort && showSort">
            <Icon name="icon_sort-a-to-z_outlined"
              ><icon_sortAToZ_outlined class="svg-icon"
            /></Icon>
          </el-icon>
          <el-icon v-if="'desc' === item.sort && showSort">
            <Icon name="icon_sort-z-to-a_outlined"
              ><icon_sortZToA_outlined class="svg-icon"
            /></Icon>
          </el-icon>
          <el-icon v-if="'custom_sort' === item.sort && showSort">
            <Icon name="icon_sort_outlined"><icon_sort_outlined class="svg-icon" /></Icon>
          </el-icon>
          <el-icon>
            <Icon :className="`field-icon-${fieldType[[2, 3].includes(item.deType) ? 2 : 0]}`"
              ><component
                class="svg-icon"
                :class="`field-icon-${fieldType[[2, 3].includes(item.deType) ? 2 : 0]}`"
                :is="iconFieldMap[fieldType[item.deType]]"
              ></component
            ></Icon>
          </el-icon>
        </span>
        <span v-if="type === 'extColor'" style="display: flex; color: #646a73">
          <el-icon>
            <Icon
              :className="`field-icon-${fieldType[[2, 3].includes(item.deType) ? 2 : 0]}`"
              :name="`field_${fieldType[item.deType]}`"
              ><component
                class="svg-icon"
                :class="`field-icon-${fieldType[[2, 3].includes(item.deType) ? 2 : 0]}`"
                :is="iconFieldMap[fieldType[item.deType]]"
              ></component
            ></Icon>
          </el-icon>
        </span>
        <el-tooltip :effect="toolTip" placement="top">
          <template #content>
            <table>
              <tbody>
                <tr>
                  <td>{{ t('dataset.field_origin_name') }}</td>
                  <td>:</td>
                  <td>{{ item.name }}</td>
                </tr>
                <tr>
                  <td>{{ t('chart.show_name') }}</td>
                  <td>:</td>
                  <td>{{ item.chartShowName ? item.chartShowName : item.name }}</td>
                </tr>
              </tbody>
            </table>
          </template>
          <span
            class="item-span-style"
            :class="{
              'hidden-status': showHideIcon,
              'sort-status': showSort && item.sort !== 'none'
            }"
          >
            <span class="item-name">{{ item.chartShowName ? item.chartShowName : item.name }}</span>
          </span>
        </el-tooltip>
        <el-icon v-if="showHideIcon" style="margin-left: 4px">
          <Icon><icon_invisible_outlined class="svg-icon inner-class" /></Icon>
        </el-icon>
        <el-tooltip :effect="toolTip" placement="top">
          <template #content>
            <span>{{ t('chart.delete') }}</span>
          </template>
          <el-icon class="child remove-icon">
            <Icon
              ><icon_deleteTrash_outlined @click="removeItem" class="svg-icon inner-class"
            /></Icon>
          </el-icon>
        </el-tooltip>
        <el-icon class="child" style="position: absolute; top: 7px; right: 10px; cursor: pointer">
          <Icon name="icon_down_outlined-1"><icon_down_outlined1 class="svg-icon" /></Icon>
        </el-icon>
      </el-tag>
      <template #dropdown>
        <el-dropdown-menu
          :effect="themes"
          class="drop-style"
          :class="themes === 'dark' ? 'dark-dimension-quota' : ''"
        >
          <el-dropdown-item @click.prevent v-if="showSort">
            <el-dropdown
              :effect="themes"
              popper-class="data-dropdown_popper_mr9"
              placement="right-start"
              style="width: 100%; height: 100%"
              @command="sort"
            >
              <span class="inner-dropdown-menu menu-item-padding">
                <span class="menu-item-content">
                  <el-icon>
                    <Icon name="icon_sort_outlined"><icon_sort_outlined class="svg-icon" /></Icon>
                  </el-icon>
                  <span>{{ t('chart.sort') }}</span>
                  <span class="summary-span-item">({{ t('chart.' + props.item.sort) }})</span>
                </span>
                <el-icon>
                  <Icon name="icon_right_outlined"><icon_right_outlined class="svg-icon" /></Icon>
                </el-icon>
              </span>
              <template #dropdown>
                <el-dropdown-menu
                  :effect="themes"
                  class="drop-style sub"
                  :class="themes === 'dark' ? 'dark-dimension-quota' : ''"
                >
                  <el-dropdown-item class="menu-item-padding" :command="beforeSort('none')">
                    <span
                      class="sub-menu-content"
                      :class="'none' === item.sort ? 'content-active' : ''"
                    >
                      {{ t('chart.none') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'none' === item.sort"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item class="menu-item-padding" :command="beforeSort('asc')">
                    <span
                      class="sub-menu-content"
                      :class="'asc' === item.sort ? 'content-active' : ''"
                    >
                      {{ t('chart.asc') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'asc' === item.sort"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item class="menu-item-padding" :command="beforeSort('desc')">
                    <span
                      class="sub-menu-content"
                      :class="'desc' === item.sort ? 'content-active' : ''"
                    >
                      {{ t('chart.desc') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'desc' === item.sort"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    v-if="showCustomSort(item)"
                    :command="beforeSort('custom_sort')"
                  >
                    <span
                      class="sub-menu-content"
                      :class="'custom_sort' === item.sort ? 'content-active' : ''"
                    >
                      {{ t('chart.custom_sort') }}{{ t('chart.sort') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'custom_sort' === item.sort"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                </el-dropdown-menu>
              </template>
            </el-dropdown>
          </el-dropdown-item>
          <el-dropdown-item
            v-if="showSortPriority"
            :command="beforeClickItem('sortPriority')"
            class="menu-item-padding"
          >
            <el-icon />
            <span>{{ t('chart.sort_priority') }}</span>
          </el-dropdown-item>
          <el-dropdown-item
            @click.prevent
            v-if="item.deType === 1"
            :divided="
              !chart.type.includes('chart-mix') ||
              (chart.type.includes('chart-mix') &&
                (type === 'dimension' || type === 'dimensionStack' || type === 'dimensionExt'))
            "
          >
            <el-dropdown
              :effect="themes"
              placement="right-start"
              popper-class="data-dropdown_popper_mr9"
              style="width: 100%; height: 100%"
              @command="dateStyle"
            >
              <span class="inner-dropdown-menu menu-item-padding">
                <span class="menu-item-content">
                  <el-icon> </el-icon>
                  <span>{{ t('chart.dateStyle') }}</span>
                  <span class="summary-span-item">({{ t('chart.' + item.dateStyle) }})</span>
                </span>
                <el-icon>
                  <Icon name="icon_right_outlined"><icon_right_outlined class="svg-icon" /></Icon>
                </el-icon>
              </span>
              <template #dropdown>
                <el-dropdown-menu
                  :effect="themes"
                  class="drop-style sub"
                  :class="themes === 'dark' ? 'dark-dimension-quota' : ''"
                >
                  <el-dropdown-item class="menu-item-padding" :command="beforeDateStyle('y')">
                    <span
                      class="sub-menu-content"
                      :class="'y' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    v-if="showDateExt"
                    :command="beforeDateStyle('y_Q')"
                  >
                    <span
                      class="sub-menu-content"
                      :class="'y_Q' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_Q') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_Q' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item class="menu-item-padding" :command="beforeDateStyle('y_M')">
                    <span
                      class="sub-menu-content"
                      :class="'y_M' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_M') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_M' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    v-if="showDateExt"
                    :command="beforeDateStyle('y_W')"
                  >
                    <span
                      class="sub-menu-content"
                      :class="'y_W' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_W') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_W' === item.dateStyle">
                          <icon_done_outlined class="svg-icon" />
                        </Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item class="menu-item-padding" :command="beforeDateStyle('y_M_d')">
                    <span
                      class="sub-menu-content"
                      :class="'y_M_d' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_M_d') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_M_d' === item.dateStyle">
                          <icon_done_outlined class="svg-icon" />
                        </Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    v-if="
                      !(chart.type.includes('bar-range') && ['quota', 'quotaExt'].includes(type))
                    "
                    :command="beforeDateStyle('H_m_s')"
                    divided
                  >
                    <span
                      class="sub-menu-content"
                      :class="'H_m_s' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.H_m_s') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'H_m_s' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    :command="beforeDateStyle('y_M_d_H')"
                    :divided="
                      chart.type.includes('bar-range') && ['quota', 'quotaExt'].includes(type)
                    "
                  >
                    <span
                      class="sub-menu-content"
                      :class="'y_M_d_H' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_M_d_H') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_M_d_H' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    :command="beforeDateStyle('y_M_d_H_m')"
                    :divided="
                      chart.type.includes('bar-range') && ['quota', 'quotaExt'].includes(type)
                    "
                  >
                    <span
                      class="sub-menu-content"
                      :class="'y_M_d_H_m' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_M_d_H_m') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_M_d_H_m' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    :command="beforeDateStyle('y_M_d_H_m_s')"
                  >
                    <span
                      class="sub-menu-content"
                      :class="'y_M_d_H_m_s' === item.dateStyle ? 'content-active' : ''"
                    >
                      {{ t('chart.y_M_d_H_m_s') }}
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'y_M_d_H_m_s' === item.dateStyle"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                </el-dropdown-menu>
              </template>
            </el-dropdown>
          </el-dropdown-item>
          <el-dropdown-item @click.prevent v-if="item.deType === 1">
            <el-dropdown
              :effect="themes"
              placement="right-start"
              popper-class="data-dropdown_popper_mr9"
              style="width: 100%; height: 100%"
              @command="datePattern"
            >
              <span class="inner-dropdown-menu menu-item-padding">
                <span class="menu-item-content">
                  <el-icon> </el-icon>
                  <span>{{ t('chart.datePattern') }}</span>
                  <span class="summary-span-item">({{ t('chart.' + item.datePattern) }})</span>
                </span>
                <el-icon>
                  <Icon name="icon_right_outlined"><icon_right_outlined class="svg-icon" /></Icon>
                </el-icon>
              </span>
              <template #dropdown>
                <el-dropdown-menu
                  :effect="themes"
                  class="drop-style sub"
                  :class="themes === 'dark' ? 'dark-dimension-quota' : ''"
                >
                  <el-dropdown-item
                    class="menu-item-padding"
                    :command="beforeDatePattern('date_sub')"
                  >
                    <span
                      class="sub-menu-content"
                      :class="'date_sub' === item.datePattern ? 'content-active' : ''"
                    >
                      {{ t('chart.date_sub') }}(1990-01-01)
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'date_sub' === item.datePattern"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                  <el-dropdown-item
                    class="menu-item-padding"
                    :command="beforeDatePattern('date_split')"
                  >
                    <span
                      class="sub-menu-content"
                      :class="'date_split' === item.datePattern ? 'content-active' : ''"
                    >
                      {{ t('chart.date_split') }}(1990/01/01)
                      <el-icon class="sub-menu-content--icon">
                        <Icon name="icon_done_outlined" v-if="'date_split' === item.datePattern"
                          ><icon_done_outlined class="svg-icon"
                        /></Icon>
                      </el-icon>
                    </span>
                  </el-dropdown-item>
                </el-dropdown-menu>
              </template>
            </el-dropdown>
          </el-dropdown-item>
          <el-dropdown-item
            class="menu-item-padding"
            :divided="
              !chart.type.includes('chart-mix') ||
              (chart.type.includes('chart-mix') &&
                (type === 'dimension' || type === 'dimensionStack' || type === 'dimensionExt'))
            "
            :command="beforeClickItem('rename')"
          >
            <el-icon>
              <icon name="icon_edit_outlined"><icon_edit_outlined class="svg-icon" /></icon>
            </el-icon>
            <span>{{ t('chart.show_name_set') }}</span>
          </el-dropdown-item>
          <el-dropdown-item
            class="menu-item-padding"
            v-if="['table-normal', 'table-info'].includes(chart.type)"
            :command="beforeClickItem('toggleHide')"
          >
            <el-icon>
              <icon
                ><icon_visible_outlined v-if="item.hide === true" class="svg-icon" />
                <icon_invisible_outlined v-else class="svg-icon"
              /></icon>
            </el-icon>
            <span>{{ item.hide === true ? t('chart.show') : t('chart.hide') }}</span>
          </el-dropdown-item>
          <el-dropdown-item
            class="menu-item-padding"
            v-if="showValueFormatter"
            :divided="!showValueFormatter"
            :command="beforeClickItem('formatter')"
          >
            <el-icon />
            <span>{{ t('chart.value_formatter') }}...</span>
          </el-dropdown-item>
          <el-dropdown-item class="menu-item-padding" divided :command="beforeClickItem('remove')">
            <el-icon>
              <icon name="icon_delete-trash_outlined"
                ><icon_deleteTrash_outlined class="svg-icon"
              /></icon>
            </el-icon>
            <span>{{ t('chart.delete') }}</span>
          </el-dropdown-item>
        </el-dropdown-menu>
      </template>
    </el-dropdown>
  </span>
</template>

<style lang="less" scoped>
:deep(.ed-dropdown-menu__item) {
  padding: 0;
}
:deep(.ed-dropdown-menu__item.menu-item-padding) {
  padding: 5px 16px;
}

.menu-item-padding {
  padding: 5px 16px;
}
.item-style {
  position: relative;
  width: 100%;
  display: block;
  overflow: hidden;
  .ed-dropdown {
    display: flex;
  }
  :deep(.ed-tag__content) {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
}

.item-axis {
  padding: 1px 8px;
  margin-bottom: 3px;
  height: 28px;
  line-height: 28px;
  display: flex;
  border-radius: 4px;
  box-sizing: border-box;
  white-space: nowrap;
  width: 100%;
  justify-content: space-between;
  align-items: center;
  background-color: #3370ff0a;
  border: 1px solid var(--ed-color-primary);
}

.item-axis:hover {
  cursor: pointer;
}

span {
  font-size: 12px;
}

.summary-span {
  margin-left: 4px;
  color: #878d9f;
  position: absolute;
  right: 25px;
}

.inner-dropdown-menu {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;

  .menu-item-content {
    display: flex;
    flex-direction: row;
    align-items: center;
  }
}

.sub-menu-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;

  &.content-active {
    color: var(--ed-color-primary);
  }

  .sub-menu-content--icon {
    margin-left: 8px;
  }
}

.item-span-drop {
  color: #a6a6a6;
  display: flex;
}

.item-span-style {
  display: flex;
  max-width: 170px;
  color: #1f2329;
  margin-left: 4px;

  &.hidden-status,
  &.sort-status {
    max-width: 150px;
  }
  &.hidden-status[class*='sort-status'] {
    max-width: 135px !important;
  }

  .item-name {
    flex: 1;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
  }
}

.editor-dark {
  .item-span-style {
    color: #ffffff !important;
  }
}

.summary-span-item {
  margin-left: 4px;
}

.drop-style {
  :deep(.ed-dropdown-menu__item) {
    height: 32px;
    min-width: 218px;
  }
  &.sub {
    :deep(.ed-dropdown-menu__item) {
      min-width: 118px;
    }
  }
  :deep(.ed-dropdown-menu__item:not(.is_disabled):focus) {
    color: inherit;
    background-color: rgba(31, 35, 41, 0.1);
  }
  &.dark-dimension-quota {
    background-color: #292929;
    border: 1px solid #434343;
    :deep(.ed-dropdown-menu__item--divided) {
      border-color: #ebebeb26;
    }
    .inner-dropdown-menu {
      color: rgba(235, 235, 235, 1);
    }
    :deep(.ed-dropdown-menu__item:not(.is-disabled):hover) {
      background-color: #ebebeb1a;
    }
    :deep(.ed-dropdown-menu__item) {
      color: rgba(235, 235, 235, 1);
    }
    :deep(.ed-dropdown-menu__item.is-disabled) {
      color: #a6a6a6;
    }
    :deep(.ed-dropdown-menu__item:not(.is_disabled):focus) {
      background-color: rgba(235, 235, 235, 0.1);
    }
  }
}
.remove-icon {
  position: absolute;
  top: 7px;
  right: 24px;
  cursor: pointer;
  .inner-class {
    font-size: 14px;
  }
}

.father {
  &.dark_icon-right {
    .child {
      color: #a6a6a6;
    }
  }

  &.light_icon-right {
    .child {
      color: #646a73;
    }
  }
  .child {
    font-size: 14px;
    visibility: hidden;
  }
}

.father:hover .child {
  visibility: visible;
}

.father:hover .item-span-style {
  max-width: 130px;
  &.hidden-status,
  &.sort-status {
    max-width: 120px;
  }
  &.hidden-status[class*='sort-status'] {
    max-width: 100px !important;
  }
}
</style>
<style lang="less">
.data-dropdown_popper_mr9 {
  margin-left: -9px !important;
}
.menu-item-padding {
  span {
    font-size: 14px;
    color: #1f2329;
  }
  .ed-icon {
    color: #646a73;
    font-size: 16px !important;
  }

  .sub-menu-content--icon {
    color: var(--ed-color-primary);
    margin-right: -7px;
  }
  :nth-child(1).ed-icon {
    margin-right: 8px;
  }
  .menu-item-content {
    :nth-child(1).ed-icon {
      margin-right: 8px;
    }
  }
}
.dark-dimension-quota {
  span {
    color: #ebebeb;
  }
  .ed-icon {
    color: #a6a6a6;
  }

  .sub-menu-content--icon {
    color: var(--ed-color-primary);
    margin-right: -7px !important;
  }
}
</style>
