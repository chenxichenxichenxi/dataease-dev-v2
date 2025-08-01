<script lang="ts" setup>
import icon_letterSpacing_outlined from '@/assets/svg/icon_letter-spacing_outlined.svg'
import icon_bold_outlined from '@/assets/svg/icon_bold_outlined.svg'
import icon_italic_outlined from '@/assets/svg/icon_italic_outlined.svg'
import { PropType, computed, onMounted, reactive, watch, nextTick } from 'vue'
import { useI18n } from '@/hooks/web/useI18n'
import {
  COLOR_PANEL,
  CHART_FONT_LETTER_SPACE,
  DEFAULT_INDICATOR_NAME_STYLE,
  DEFAULT_BASIC_STYLE,
  CHART_FONT_FAMILY_ORIGIN
} from '@/views/chart/components/editor/util/chart'
import { cloneDeep, defaultsDeep } from 'lodash-es'
import Icon from '@/components/icon-custom/src/Icon.vue'
import { useAppearanceStoreWithOut } from '@/store/modules/appearance'

const { t } = useI18n()
const appearanceStore = useAppearanceStoreWithOut()

const props = defineProps({
  chart: {
    type: Object,
    required: true
  },
  themes: {
    type: String as PropType<EditorTheme>,
    default: 'dark'
  },
  propertyInner: {
    type: Array<string>
  }
})

const emit = defineEmits(['onIndicatorNameChange'])
const toolTip = computed(() => {
  return props.themes === 'dark' ? 'light' : 'dark'
})
const predefineColors = COLOR_PANEL
const fontFamily = CHART_FONT_FAMILY_ORIGIN.concat(
  appearanceStore.fontList.map(ele => ({
    name: ele.name,
    value: ele.name
  }))
)
const fontLetterSpace = CHART_FONT_LETTER_SPACE

const state = reactive({
  indicatorNameForm: JSON.parse(JSON.stringify(DEFAULT_INDICATOR_NAME_STYLE)),
  basicStyleForm: {} as ChartBasicStyle
})

const fontSizeList = computed(() => {
  const arr = []
  for (let i = 10; i <= 60; i = i + 2) {
    arr.push({
      name: i + '',
      value: i
    })
  }
  for (let i = 70; i <= 210; i += 10) {
    arr.push({
      name: i + '',
      value: i
    })
  }
  return arr
})

const changeTitleStyle = prop => {
  emit('onIndicatorNameChange', state.indicatorNameForm, prop)
}

const init = () => {
  const TEMP_DEFAULT_BASIC_STYLE = cloneDeep(DEFAULT_BASIC_STYLE)
  delete TEMP_DEFAULT_BASIC_STYLE.alpha

  state.basicStyleForm = defaultsDeep(
    cloneDeep(props.chart?.customAttr?.basicStyle),
    cloneDeep(TEMP_DEFAULT_BASIC_STYLE)
  )

  const customText = defaultsDeep(
    cloneDeep(props.chart?.customAttr?.indicatorName),
    cloneDeep(DEFAULT_INDICATOR_NAME_STYLE)
  )

  state.indicatorNameForm = cloneDeep(customText)

  //第一次颜色可能赋值失败，单独赋值一次
  nextTick(() => {
    state.indicatorNameForm.color = customText.color
  })
}

onMounted(() => {
  init()
})

watch(
  () => props.chart?.customAttr?.indicatorName,
  () => {
    init()
  },
  { deep: true }
)

function getFormData() {
  return state.indicatorNameForm
}

defineExpose({ getFormData })
</script>

<template>
  <div>
    <el-form
      ref="indicatorNameForm"
      :disabled="!state.indicatorNameForm.show"
      :model="state.indicatorNameForm"
      label-position="top"
      size="small"
    >
      <el-form-item
        class="form-item"
        :class="'form-item-' + themes"
        :effect="themes"
        :label="t('chart.text')"
      >
        <el-select
          :effect="themes"
          v-model="state.indicatorNameForm.fontFamily"
          :placeholder="t('chart.font_family')"
          @change="changeTitleStyle('fontFamily')"
        >
          <el-option
            v-for="option in fontFamily"
            :key="option.value"
            :label="option.name"
            :value="option.value"
          />
        </el-select>
      </el-form-item>

      <div style="display: flex">
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-right: 4px">
          <el-color-picker
            :effect="themes"
            v-model="state.indicatorNameForm.color"
            class="color-picker-style"
            :predefine="predefineColors"
            @change="changeTitleStyle('color')"
            is-custom
            show-alpha
          />
        </el-form-item>
        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding: 0 4px">
          <el-tooltip :content="t('chart.font_size')" :effect="toolTip" placement="top">
            <el-select
              style="width: 56px"
              :effect="themes"
              v-model="state.indicatorNameForm.fontSize"
              :placeholder="t('chart.text_fontsize')"
              size="small"
              @change="changeTitleStyle('fontSize')"
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

        <el-form-item class="form-item" :class="'form-item-' + themes" style="padding-left: 4px">
          <el-select
            size="small"
            :effect="themes"
            v-model="state.indicatorNameForm.letterSpace"
            :placeholder="t('chart.quota_letter_space')"
            @change="changeTitleStyle('letterSpace')"
          >
            <template #prefix>
              <el-icon>
                <Icon name="icon_letter-spacing_outlined"
                  ><icon_letterSpacing_outlined class="svg-icon"
                /></Icon>
              </el-icon>
            </template>
            <el-option
              v-for="option in fontLetterSpace"
              :key="option.value"
              :label="option.name"
              :value="option.value"
            />
          </el-select>
        </el-form-item>
      </div>

      <el-space>
        <el-form-item class="form-item" :class="'form-item-' + themes">
          <el-checkbox
            :effect="themes"
            class="icon-checkbox"
            v-model="state.indicatorNameForm.isBolder"
            @change="changeTitleStyle('isBolder')"
          >
            <el-tooltip :effect="toolTip" placement="top">
              <template #content>
                {{ t('chart.bolder') }}
              </template>
              <div
                class="icon-btn"
                :class="{ dark: themes === 'dark', active: state.indicatorNameForm.isBolder }"
              >
                <el-icon>
                  <Icon name="icon_bold_outlined"><icon_bold_outlined class="svg-icon" /></Icon>
                </el-icon>
              </div>
            </el-tooltip>
          </el-checkbox>
        </el-form-item>

        <el-form-item class="form-item" :class="'form-item-' + themes">
          <el-checkbox
            :effect="themes"
            class="icon-checkbox"
            v-model="state.indicatorNameForm.isItalic"
            @change="changeTitleStyle('isItalic')"
          >
            <el-tooltip :effect="toolTip" placement="top">
              <template #content>
                {{ t('chart.italic') }}
              </template>
              <div
                class="icon-btn"
                :class="{ dark: themes === 'dark', active: state.indicatorNameForm.isItalic }"
              >
                <el-icon>
                  <Icon name="icon_italic_outlined"><icon_italic_outlined class="svg-icon" /></Icon>
                </el-icon>
              </div>
            </el-tooltip>
          </el-checkbox>
        </el-form-item>
      </el-space>

      <el-form-item class="form-item" :class="'form-item-' + themes">
        <el-checkbox
          size="small"
          :effect="themes"
          v-model="state.indicatorNameForm.fontShadow"
          @change="changeTitleStyle('fontShadow')"
        >
          {{ t('chart.font_shadow') }}
        </el-checkbox>
      </el-form-item>
      <el-form-item
        class="form-item name-value-spacing-input"
        :class="'form-item-' + themes"
        :label="t('chart.name_value_spacing')"
      >
        <el-input-number
          step-strictly
          v-model="state.indicatorNameForm.nameValueSpacing"
          size="small"
          :min="0"
          :max="100"
          :value-on-clear="0"
          :precision="0"
          :step="1"
          :effect="themes"
          @change="changeTitleStyle('nameValueSpacing')"
        />
      </el-form-item>
    </el-form>
  </div>
</template>

<style lang="less" scoped>
:deep(.ed-input .ed-select__prefix--light) {
  padding-right: 6px;
}
.icon-btn {
  font-size: 16px;
  line-height: 16px;
  width: 24px;
  height: 24px;
  text-align: center;
  border-radius: 4px;
  padding-top: 4px;

  color: #1f2329;

  cursor: pointer;

  &.dark {
    color: #a6a6a6;
    &.active {
      color: var(--ed-color-primary);
      background-color: var(--ed-color-primary-1a, rgba(51, 112, 255, 0.1));
    }
    &:hover {
      background-color: rgba(255, 255, 255, 0.1);
    }
  }

  &.active {
    color: var(--ed-color-primary);
    background-color: var(--ed-color-primary-1a, rgba(51, 112, 255, 0.1));
  }

  &:hover {
    background-color: rgba(31, 35, 41, 0.1);
  }
}

.is-disabled {
  .icon-btn {
    color: #8f959e;
    cursor: not-allowed;

    &:hover {
      background-color: inherit;
    }

    &.active {
      background-color: #f5f7fa;
      &:hover {
        background-color: #f5f7fa;
      }
    }
    &.dark {
      color: #5f5f5f;
      &.active {
        background-color: #373737;
        &:hover {
          background-color: #373737;
        }
      }
    }
  }
}

.icon-checkbox {
  :deep(.ed-checkbox__input) {
    display: none;
  }
  :deep(.ed-checkbox__label) {
    padding: 0;
  }
}

.icon-radio-group {
  :deep(.ed-radio) {
    margin-right: 8px;

    &:last-child {
      margin-right: 0;
    }
  }
  :deep(.ed-radio__input) {
    display: none;
  }
  :deep(.ed-radio__label) {
    padding: 0;
  }
}
.position-divider {
  width: 1px;
  height: 18px;
  margin-bottom: 16px;
  background: rgba(31, 35, 41, 0.15);

  &.position-divider--dark {
    background: rgba(235, 235, 235, 0.15);
  }
}
.remark-label {
  color: var(--N600, #646a73);
  font-family: var(--de-custom_font, 'PingFang');
  font-size: 12px;
  font-style: normal;
  font-weight: 400;
  line-height: 20px;

  &.remark-label--dark {
    color: var(--N600-Dark, #a6a6a6);
  }
}
.name-value-spacing-input {
  display: flex !important;
  :deep(label) {
    line-height: 28px !important;
    margin-bottom: 0 !important;
  }
  :deep(.ed-input__inner) {
    text-align: center !important;
  }
}
</style>
