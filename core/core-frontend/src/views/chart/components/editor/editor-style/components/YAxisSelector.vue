<script lang="tsx" setup>
import icon_info_outlined from '@/assets/svg/icon_info_outlined.svg'
import { computed, onMounted, PropType, reactive, watch } from 'vue'
import { useI18n } from '@/hooks/web/useI18n'
import { COLOR_PANEL, DEFAULT_YAXIS_STYLE } from '@/views/chart/components/editor/util/chart'
import {
  isEnLocal,
  formatterType,
  getUnitTypeList,
  initFormatCfgUnit,
  onChangeFormatCfgUnitLanguage
} from '@/views/chart/components/js/formatter'
import { ElFormItem, ElMessage } from 'element-plus-secondary'

const { t } = useI18n()

const props = defineProps({
  themes: {
    type: String as PropType<EditorTheme>,
    default: 'dark'
  },
  chart: {
    type: Object,
    required: true
  },
  propertyInner: {
    type: Array<string>
  }
})

const predefineColors = COLOR_PANEL
const typeList = formatterType
const toolTip = computed(() => {
  return props.themes === 'dark' ? 'light' : 'dark'
})
const state = reactive({
  axisForm: JSON.parse(JSON.stringify(DEFAULT_YAXIS_STYLE))
})

const emit = defineEmits(['onChangeYAxisForm'])

watch(
  () => props.chart.customStyle.yAxis,
  () => {
    init()
  },
  { deep: true }
)

const fontSizeList = computed(() => {
  const arr = []
  for (let i = 10; i <= 40; i = i + 2) {
    arr.push({
      name: i + '',
      value: i
    })
  }
  for (let i = 50; i <= 200; i = i + 10) {
    arr.push({
      name: i + '',
      value: i
    })
  }
  return arr
})

const splitLineStyle = [
  { label: t('chart.line_type_solid'), value: 'solid' },
  { label: t('chart.line_type_dashed'), value: 'dashed' },
  { label: t('chart.line_type_dotted'), value: 'dotted' }
]

const changeAxisStyle = prop => {
  if (
    state.axisForm.axisValue.splitCount &&
    (state.axisForm.axisValue.splitCount > 100 || state.axisForm.axisValue.splitCount < 0)
  ) {
    ElMessage.error(t('chart.splitCount_less_100'))
    return
  }
  emit('onChangeYAxisForm', state.axisForm, prop)
}

function changeUnitLanguage(cfg: BaseFormatter, lang, prop: string) {
  onChangeFormatCfgUnitLanguage(cfg, lang)
  changeAxisStyle(prop)
}

const init = () => {
  const chart = JSON.parse(JSON.stringify(props.chart))
  if (chart.customStyle) {
    let customStyle = null
    if (Object.prototype.toString.call(chart.customStyle) === '[object Object]') {
      customStyle = JSON.parse(JSON.stringify(chart.customStyle))
    } else {
      customStyle = JSON.parse(chart.customStyle)
    }
    if (customStyle.yAxis) {
      state.axisForm = customStyle.yAxis
      initFormatCfgUnit(state.axisForm.axisLabelFormatter)
    }
  }
}

const showProperty = prop => props.propertyInner?.includes(prop)
const isBulletGraph = computed(() => {
  return ['bullet-graph'].includes(props.chart.type)
})

const isHorizontalLayout = computed(() => {
  return props.chart.customAttr.basicStyle.layout === 'horizontal'
})

onMounted(() => {
  init()
})
</script>

<template>
  <el-form
    ref="axisForm"
    :disabled="!state.axisForm.show"
    :model="state.axisForm"
    size="small"
    label-position="top"
  >
    <el-form-item
      class="form-item"
      :class="'form-item-' + themes"
      :label="t('chart.position')"
      v-if="showProperty('position')"
    >
      <el-radio-group
        v-model="state.axisForm.position"
        size="small"
        @change="changeAxisStyle('position')"
      >
        <div v-if="isBulletGraph">
          <div v-if="isHorizontalLayout">
            <el-radio :effect="props.themes" label="right">{{ t('chart.text_pos_top') }}</el-radio>
            <el-radio :effect="props.themes" label="left">{{
              t('chart.text_pos_bottom')
            }}</el-radio>
          </div>
          <div v-else>
            <el-radio :effect="props.themes" label="left">{{ t('chart.text_pos_left') }}</el-radio>
            <el-radio :effect="props.themes" label="right">{{
              t('chart.text_pos_right')
            }}</el-radio>
          </div>
        </div>
        <div v-else>
          <el-radio :effect="props.themes" label="left">{{ t('chart.text_pos_left') }}</el-radio>
          <el-radio :effect="props.themes" label="right">{{ t('chart.text_pos_right') }}</el-radio>
        </div>
      </el-radio-group>
    </el-form-item>

    <el-form-item class="form-item" :class="'form-item-' + themes">
      <el-checkbox
        size="small"
        :effect="props.themes"
        v-model="state.axisForm.nameShow"
        @change="changeAxisStyle('nameShow')"
      >
        {{ t('chart.axis_nameShow') }}
      </el-checkbox>
    </el-form-item>
    <div style="margin-left: 22px">
      <el-form-item
        class="form-item"
        :class="'form-item-' + themes"
        :label="t('chart.name')"
        v-if="showProperty('name')"
      >
        <el-input
          :effect="props.themes"
          v-model="state.axisForm.name"
          :disabled="!state.axisForm.nameShow"
          size="small"
          maxlength="50"
          @blur="changeAxisStyle('name')"
        />
      </el-form-item>

      <div style="display: flex">
        <el-form-item
          class="form-item"
          :class="'form-item-' + themes"
          v-if="showProperty('color')"
          style="padding-right: 4px"
          :label="t('chart.chart_style')"
        >
          <el-color-picker
            v-model="state.axisForm.color"
            :disabled="!state.axisForm.nameShow"
            class="color-picker-style"
            :predefine="predefineColors"
            @change="changeAxisStyle('color')"
            :effect="themes"
            is-custom
          />
        </el-form-item>
        <el-form-item
          class="form-item"
          :class="'form-item-' + themes"
          v-if="showProperty('fontSize')"
          style="padding-left: 4px"
        >
          <template #label>&nbsp;</template>
          <el-tooltip :content="t('chart.font_size')" :effect="toolTip" placement="top">
            <el-select
              :disabled="!state.axisForm.nameShow"
              style="width: 108px"
              :effect="props.themes"
              v-model="state.axisForm.fontSize"
              :placeholder="t('chart.axis_name_fontsize')"
              @change="changeAxisStyle('fontSize')"
            >
              <el-option
                v-for="option in fontSizeList"
                :key="option.value"
                :label="option.name"
                :value="option.value"
              />
            </el-select>
          </el-tooltip>
        </el-form-item>
      </div>
    </div>

    <template v-if="showProperty('axisValue')">
      <el-divider class="m-divider" :class="'m-divider--' + themes" />

      <div style="display: flex; flex-direction: row; justify-content: space-between">
        <label class="custom-form-item-label" :class="'custom-form-item-label--' + themes">
          {{ t('chart.axis_value') }}
          <el-tooltip class="item" :effect="toolTip" placement="top">
            <template #content><span v-html="t('chart.axis_tip')"></span></template>
            <span style="vertical-align: middle">
              <el-icon style="cursor: pointer">
                <Icon name="icon_info_outlined"><icon_info_outlined class="svg-icon" /></Icon>
              </el-icon>
            </span>
          </el-tooltip>
        </label>

        <el-form-item class="form-item" :class="'form-item-' + themes">
          <el-checkbox
            size="small"
            :effect="props.themes"
            v-model="state.axisForm.axisValue.auto"
            @change="changeAxisStyle('axisValue.auto')"
          >
            {{ t('chart.axis_auto') }}
          </el-checkbox>
        </el-form-item>
      </div>

      <template v-if="showProperty('axisValue') && !state.axisForm.axisValue.auto">
        <el-row :gutter="8">
          <el-col :span="12">
            <el-form-item
              class="form-item"
              :class="'form-item-' + themes"
              :label="t('chart.axis_value_max')"
            >
              <el-input-number
                controls-position="right"
                :effect="props.themes"
                v-model.number="state.axisForm.axisValue.max"
                @change="changeAxisStyle('axisValue.max')"
              />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item
              class="form-item"
              :class="'form-item-' + themes"
              :label="t('chart.axis_value_min')"
            >
              <el-input-number
                :effect="props.themes"
                controls-position="right"
                v-model.number="state.axisForm.axisValue.min"
                @change="changeAxisStyle('axisValue.min')"
              />
            </el-form-item>
          </el-col>
        </el-row>

        <label class="custom-form-item-label" :class="'custom-form-item-label--' + themes">
          {{ t('chart.axis_value_split_count') }}
          <el-tooltip class="item" :effect="toolTip" placement="top">
            <template #content>{{ t('chart.number_of_scales_tip') }}</template>
            <span style="vertical-align: middle">
              <el-icon style="cursor: pointer">
                <Icon name="icon_info_outlined"><icon_info_outlined class="svg-icon" /></Icon>
              </el-icon>
            </span>
          </el-tooltip>
        </label>

        <el-form-item class="form-item" :class="'form-item-' + themes">
          <el-input-number
            style="width: 100%"
            :effect="props.themes"
            controls-position="right"
            v-model.number="state.axisForm.axisValue.splitCount"
            @change="changeAxisStyle('axisValue.splitCount')"
          />
        </el-form-item>
      </template>
    </template>
    <el-divider
      v-if="showProperty('splitLine') || showProperty('axisLine')"
      class="m-divider"
      :class="'m-divider--' + themes"
    />
    <el-form-item class="form-item" :class="'form-item-' + themes" v-if="showProperty('axisLine')">
      <el-checkbox
        size="small"
        :effect="props.themes"
        v-model="state.axisForm.axisLine.show"
        @change="changeAxisStyle('axisLine.show')"
      >
        {{ t('chart.axis_show') }}
      </el-checkbox>
    </el-form-item>
    <div style="padding-left: 22px" v-if="showProperty('axisLine')">
      <div style="flex: 1; display: flex">
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-right: 4px">
          <el-color-picker
            :disabled="!state.axisForm.axisLine.show"
            v-model="state.axisForm.axisLine.lineStyle.color"
            :predefine="predefineColors"
            :effect="themes"
            @change="changeAxisStyle('axisLine.lineStyle.color')"
            is-custom
          />
        </el-form-item>
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding: 0 4px">
          <el-select
            :disabled="!state.axisForm.axisLine.show"
            style="width: 62px"
            :effect="props.themes"
            v-model="state.axisForm.axisLine.lineStyle.style"
            @change="changeAxisStyle('axisLine.lineStyle.style')"
          >
            <el-option
              v-for="option in splitLineStyle"
              :key="option.value"
              :value="option.value"
              :label="option.label"
            ></el-option>
          </el-select>
        </el-form-item>
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-left: 4px">
          <el-input-number
            :disabled="!state.axisForm.axisLine.show"
            style="width: 70px"
            :effect="props.themes"
            v-model="state.axisForm.axisLine.lineStyle.width"
            :min="1"
            :max="10"
            size="small"
            controls-position="right"
            @change="changeAxisStyle('axisLine.lineStyle.width')"
          />
        </el-form-item>
      </div>
    </div>
    <el-form-item
      class="form-item form-item-checkbox"
      :class="{
        'form-item-dark': themes === 'dark'
      }"
      v-if="showProperty('splitLine')"
    >
      <el-checkbox
        size="small"
        :effect="props.themes"
        v-model="state.axisForm.splitLine.show"
        @change="changeAxisStyle('splitLine.show')"
      >
        {{ t('chart.grid_show') }}
      </el-checkbox>
    </el-form-item>
    <div style="padding-left: 22px" v-if="showProperty('splitLine')">
      <div style="flex: 1; display: flex">
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-right: 4px">
          <el-color-picker
            :disabled="!state.axisForm.splitLine.show"
            v-model="state.axisForm.splitLine.lineStyle.color"
            :predefine="predefineColors"
            @change="changeAxisStyle('splitLine.lineStyle.color')"
            :effect="themes"
            is-custom
          />
        </el-form-item>
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding: 0 4px">
          <el-select
            :disabled="!state.axisForm.splitLine.show"
            style="width: 62px"
            :effect="props.themes"
            v-model="state.axisForm.splitLine.lineStyle.style"
            @change="changeAxisStyle('splitLine.lineStyle.style')"
          >
            <el-option
              v-for="option in splitLineStyle"
              :key="option.value"
              :value="option.value"
              :label="option.label"
            ></el-option>
          </el-select>
        </el-form-item>
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-left: 4px">
          <el-input-number
            :disabled="!state.axisForm.splitLine.show"
            style="width: 70px"
            :effect="props.themes"
            v-model="state.axisForm.splitLine.lineStyle.width"
            :min="1"
            :max="10"
            size="small"
            controls-position="right"
            @change="changeAxisStyle('splitLine.lineStyle.width')"
          />
        </el-form-item>
      </div>
    </div>
    <el-divider
      v-if="showProperty('axisLabel')"
      class="m-divider"
      :class="'m-divider--' + themes"
    />
    <el-form-item
      class="form-item form-item-checkbox"
      :class="{
        'form-item-dark': themes === 'dark'
      }"
      v-if="showProperty('axisLabel')"
    >
      <el-checkbox
        size="small"
        :effect="props.themes"
        v-model="state.axisForm.axisLabel.show"
        @change="changeAxisStyle('axisLabel.show')"
      >
        {{ t('chart.axis_label_show') }}
      </el-checkbox>
    </el-form-item>
    <div style="padding-left: 22px" v-if="showProperty('axisLabel')">
      <div style="flex: 1">
        <div style="display: flex">
          <el-form-item
            class="form-item"
            :class="'form-item-' + themes"
            style="padding-right: 4px"
            :label="t('chart.text')"
          >
            <el-color-picker
              :disabled="!state.axisForm.axisLabel.show"
              v-model="state.axisForm.axisLabel.color"
              :predefine="predefineColors"
              @change="changeAxisStyle('axisLabel.color')"
              :effect="themes"
              is-custom
            />
          </el-form-item>
          <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-left: 4px">
            <template #label>&nbsp;</template>
            <el-tooltip :content="t('chart.font_size')" :effect="toolTip" placement="top">
              <el-select
                :disabled="!state.axisForm.axisLabel.show"
                style="width: 108px"
                :effect="props.themes"
                v-model="state.axisForm.axisLabel.fontSize"
                :placeholder="t('chart.axis_label_fontsize')"
                @change="changeAxisStyle('axisLabel.fontSize')"
              >
                <el-option
                  v-for="option in fontSizeList"
                  :key="option.value"
                  :label="option.name"
                  :value="option.value"
                />
              </el-select>
            </el-tooltip>
          </el-form-item>
        </div>

        <el-form-item class="form-item" :class="'form-item-' + themes" :label="t('chart.rotate')">
          <el-input-number
            :disabled="!state.axisForm.axisLabel.show"
            style="width: 100%"
            :effect="props.themes"
            v-model="state.axisForm.axisLabel.rotate"
            :min="-90"
            :max="90"
            size="small"
            controls-position="right"
            @change="changeAxisStyle('axisLabel.rotate')"
          />
        </el-form-item>
        <el-form-item
          class="form-item"
          :class="'form-item-' + themes"
          :label="t('chart.length_limit')"
          v-if="showProperty('showLengthLimit')"
        >
          <el-input-number
            :disabled="!state.axisForm.axisLabel.show"
            style="width: 100%"
            :effect="props.themes"
            v-model="state.axisForm.axisLabel.lengthLimit"
            :min="1"
            size="small"
            controls-position="right"
            @change="changeAxisStyle('axisLabel.lengthLimit')"
          />
        </el-form-item>

        <template v-if="showProperty('axisLabelFormatter')">
          <el-form-item
            class="form-item"
            :class="'form-item-' + themes"
            :label="t('chart.value_formatter_type')"
          >
            <el-select
              :disabled="!state.axisForm.axisLabel.show"
              style="width: 100%"
              :effect="props.themes"
              v-model="state.axisForm.axisLabelFormatter.type"
              @change="changeAxisStyle('axisLabelFormatter.type')"
            >
              <el-option
                v-for="type in typeList"
                :key="type.value"
                :label="t('chart.' + type.name)"
                :value="type.value"
              />
            </el-select>
          </el-form-item>
          <el-form-item
            v-if="state.axisForm.axisLabelFormatter.type !== 'auto'"
            :label="t('chart.value_formatter_decimal_count')"
            class="form-item"
            :class="'form-item-' + themes"
          >
            <el-input-number
              :disabled="!state.axisForm.axisLabel.show"
              style="width: 100%"
              :effect="props.themes"
              v-model="state.axisForm.axisLabelFormatter.decimalCount"
              :precision="0"
              :min="0"
              :max="10"
              size="small"
              controls-position="right"
              @change="changeAxisStyle('axisLabelFormatter.decimalCount')"
            />
          </el-form-item>

          <template
            v-if="
              state.axisForm.axisLabel.show && state.axisForm.axisLabelFormatter.type !== 'percent'
            "
          >
            <el-row :gutter="8">
              <el-col :span="12" v-if="!isEnLocal">
                <el-form-item
                  :label="t('chart.value_formatter_unit_language')"
                  class="form-item"
                  :class="'form-item-' + themes"
                >
                  <el-select
                    size="small"
                    :effect="themes"
                    v-model="state.axisForm.axisLabelFormatter.unitLanguage"
                    :placeholder="t('chart.pls_select_field')"
                    @change="
                      v =>
                        changeUnitLanguage(
                          state.axisForm.axisLabelFormatter,
                          v,
                          'axisLabelFormatter'
                        )
                    "
                  >
                    <el-option :label="t('chart.value_formatter_unit_language_ch')" value="ch" />
                    <el-option :label="t('chart.value_formatter_unit_language_en')" value="en" />
                  </el-select>
                </el-form-item>
              </el-col>
              <el-col :span="isEnLocal ? 24 : 12">
                <el-form-item
                  class="form-item"
                  :class="'form-item-' + themes"
                  :label="t('chart.value_formatter_unit')"
                >
                  <el-select
                    :effect="props.themes"
                    v-model="state.axisForm.axisLabelFormatter.unit"
                    :placeholder="t('chart.pls_select_field')"
                    size="small"
                    @change="changeAxisStyle('axisLabelFormatter')"
                  >
                    <el-option
                      v-for="item in getUnitTypeList(
                        state.axisForm.axisLabelFormatter.unitLanguage
                      )"
                      :key="item.value"
                      :label="item.name"
                      :value="item.value"
                    />
                  </el-select>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="8">
              <el-col :span="24">
                <el-form-item
                  class="form-item"
                  :class="'form-item-' + themes"
                  :label="t('chart.value_formatter_suffix')"
                >
                  <el-input
                    :disabled="!state.axisForm.axisLabel.show"
                    :effect="props.themes"
                    v-model="state.axisForm.axisLabelFormatter.suffix"
                    size="small"
                    clearable
                    :placeholder="t('commons.input_content')"
                    @change="changeAxisStyle('axisLabelFormatter.suffix')"
                  />
                </el-form-item>
              </el-col>
            </el-row>
          </template>

          <el-form-item class="form-item" :class="'form-item-' + themes">
            <el-checkbox
              :disabled="!state.axisForm.axisLabel.show"
              size="small"
              :effect="props.themes"
              v-model="state.axisForm.axisLabelFormatter.thousandSeparator"
              @change="changeAxisStyle('axisLabelFormatter.thousandSeparator')"
              :label="t('chart.value_formatter_thousand_separator')"
            />
          </el-form-item>
        </template>
      </div>
    </div>
  </el-form>
</template>

<style lang="less" scoped>
.custom-form-item-label {
  margin-bottom: 4px;
  line-height: 20px;
  color: #646a73;
  font-size: 12px;
  font-style: normal;
  font-weight: 400;
  padding: 2px 12px 0 0;

  &.custom-form-item-label--dark {
    color: #a6a6a6;
  }
}
.form-item-checkbox {
  margin-bottom: 10px !important;
}
.m-divider {
  border-color: rgba(31, 35, 41, 0.15);
  margin: 0 0 16px;

  &.m-divider--dark {
    border-color: rgba(235, 235, 235, 0.15);
  }
}
</style>
