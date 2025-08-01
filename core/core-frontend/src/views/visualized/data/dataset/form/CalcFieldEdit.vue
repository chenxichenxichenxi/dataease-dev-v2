<script lang="ts" setup>
import icon_info_outlined from '@/assets/svg/icon_info_outlined.svg'
import icon_searchOutline_outlined from '@/assets/svg/icon_search-outline_outlined.svg'
import icon_adjustment_outlined from '@/assets/svg/icon_adjustment_outlined.svg'
import icon_edit_outlined from '@/assets/svg/icon_edit_outlined.svg'
import icon_deleteTrash_outlined from '@/assets/svg/icon_delete-trash_outlined.svg'
import { ref, reactive, onMounted, onBeforeUnmount, watch, unref, computed, nextTick } from 'vue'
import { useI18n } from '@/hooks/web/useI18n'
import CodeMirror from './CodeMirror.vue'
import { getFunction } from '@/api/dataset'
import { fieldType } from '@/utils/attr'
import { cloneDeep } from 'lodash-es'
import { guid } from './util'
import { iconFieldMap } from '@/components/icon-group/field-list'
export interface CalcFieldType {
  id?: string
  datasourceId?: string // 数据源id
  datasetTableId?: string // union node id
  datasetGroupId?: string // 有就传，没有null
  originName: string // 物理字段名
  name: string // 字段显示名
  dataeaseName?: string // 字段别名
  groupType: 'd' | 'q' // d=维度，q=指标
  type: string
  params?: Array<{ id: string; name: string; value: number }>
  checked: boolean
  deType: number // 字段类型
  deExtractType?: number // 字段原始类型
  extField?: number // 0=原始字段，2=复制或计算字段
  fieldShortName?: string // 字段别名
}
const { t } = useI18n()

const myCm = ref()
const searchField = ref('')
const searchFunction = ref('')

const mirror = ref()

const props = defineProps({
  crossDs: {
    type: Boolean,
    default: () => false
  }
})

const fields = [
  { label: t('dataset.text'), value: 0 },
  { label: t('dataset.time'), value: 1 },
  { label: t('dataset.value'), value: 2 },
  {
    label: t('dataset.value') + '(' + t('dataset.float') + ')',
    value: 3
  },
  { label: t('dataset.location'), value: 5 },
  { label: 'URL', value: 7 }
]

const defaultForm = {
  originName: '', // 物理字段名
  name: '', // 字段显示名
  groupType: 'd', // d=维度，q=指标
  type: 'VARCHAR',
  deType: 0, // 字段类型
  extField: 2,
  id: '',
  params: [],
  checked: true
}

const state = reactive({
  functionData: [],
  dimensionData: [],
  dimensionList: [],
  quotaData: []
})
const formQuotaRef = ref()
const formQuota = reactive({
  id: null,
  name: '',
  value: null
})
const dialogFormVisible = ref(false)
const formQuotaRules = {
  name: [
    { required: true, message: t('data_set.enter_parameter_name'), trigger: 'blur' },
    { min: 1, max: 50, message: t('data_set.enter_1_50_characters'), trigger: 'blur' }
  ],
  value: [{ required: true, message: t('data_set.parameter_default_value'), trigger: 'blur' }]
}

const formQuotaClose = () => {
  formQuotaRef.value.resetFields()
  dialogFormVisible.value = false
}

const formQuotaConfirm = () => {
  formQuotaRef.value.validate(val => {
    if (val) {
      if (!formQuota.id) {
        formQuota.id = `params_${guid()}`
      }
      const q = cloneDeep(unref(formQuota))
      fieldForm.params = [q]
      const i = state.quotaData.find(ele => ele.id === formQuota.id)
      if (i) {
        const str = mirror.value.state.doc.toString()
        const name2Auto = []
        fieldForm.originName = setNameIdTrans('name', 'id', str, name2Auto)
        Object.assign(i, cloneDeep(unref(formQuota)))

        nextTick(() => {
          mirror.value.dispatch({
            changes: {
              from: 0,
              to: mirror.value.viewState.state.doc.length,
              insert: setNameIdTrans('id', 'name', fieldForm.originName)
            }
          })
        })
      } else {
        state.quotaData.push(q)
      }
      formQuotaClose()
    }
  })
}

const fieldForm = reactive<CalcFieldType>({ ...(defaultForm as CalcFieldType) })

const setFieldForm = () => {
  const str = mirror.value.state.doc.toString()
  const name2Auto = []
  fieldForm.originName = setNameIdTrans('name', 'id', str, name2Auto)
}

const setNameIdTrans = (from, to, originName, name2Auto?: string[]) => {
  let name2Id = originName
  const nameIdMap = [...dimensionDataList, ...quotaDataList].reduce((pre, next) => {
    pre[next[from]] = next[to]
    return pre
  }, {})
  const on = originName.match(/\[(.+?)\]/g) || []
  if (on) {
    on.forEach(itm => {
      const ele = itm.slice(1, -1)
      if (name2Auto) {
        name2Auto.push(nameIdMap[ele])
      }
      name2Id = name2Id.replace(`[${ele}]`, `[${nameIdMap[ele]}]`)
    })
  }
  return name2Id
}

let quotaDataList = []
let dimensionDataList = []
const initEdit = (obj, dimensionData, quotaData) => {
  formQuota.id = null
  Object.assign(fieldForm, { ...defaultForm, ...obj })
  state.dimensionData = dimensionData
  state.quotaData = quotaData.concat(fieldForm.params || [])
  quotaDataList = cloneDeep(quotaData.concat(fieldForm.params || []))
  dimensionDataList = cloneDeep(dimensionData)
  setTimeout(() => {
    formField.value.clearValidate()
  }, 100)
  if (!obj.originName) {
    mirror.value.dispatch({
      changes: {
        from: 0,
        to: mirror.value.viewState.state.doc.length,
        insert: ''
      }
    })
    return
  }
  nextTick(() => {
    mirror.value.dispatch({
      changes: {
        from: 0,
        to: mirror.value.viewState.state.doc.length,
        insert: setNameIdTrans('id', 'name', obj.originName)
      }
    })
  })
}

const insertFieldToCodeMirror = (value: string) => {
  mirror.value.dispatch({
    changes: { from: mirror.value.viewState.state.selection.ranges[0].from, insert: value },
    selection: { anchor: mirror.value.viewState.state.selection.ranges[0].from }
  })
}

onMounted(() => {
  mirror.value = myCm.value.codeComInit()
})

onBeforeUnmount(() => {
  mirror.value.destroy?.()
})

const insertParamToCodeMirror = (value: string) => {
  mirror.value.dispatch({
    changes: { from: mirror.value.viewState.state.selection.ranges[0].from, insert: value },
    selection: { anchor: mirror.value.viewState.state.selection.ranges[0].from }
  })
}
let functions = []
const initFunction = () => {
  getFunction().then(res => {
    functions = cloneDeep(res)
    state.functionData = cloneDeep(res)
  })
}
watch(
  () => searchField.value,
  val => {
    if (val && val !== '') {
      state.dimensionData = JSON.parse(
        JSON.stringify(
          dimensionDataList.filter(
            ele =>
              ele.name.toLocaleLowerCase().includes(val.toLocaleLowerCase()) && ele.extField === 0
          )
        )
      )
      state.quotaData = JSON.parse(
        JSON.stringify(
          quotaDataList.filter(
            ele =>
              ele.name.toLocaleLowerCase().includes(val.toLocaleLowerCase()) && ele.extField === 0
          )
        )
      )
    } else {
      state.dimensionData = JSON.parse(JSON.stringify(dimensionDataList))
      state.quotaData = JSON.parse(JSON.stringify(quotaDataList))
    }
  }
)

watch(
  () => searchFunction.value,
  val => {
    if (val && val !== '') {
      state.functionData = JSON.parse(
        JSON.stringify(
          functions.filter(ele => {
            return ele.func.toLocaleLowerCase().includes(val.toLocaleLowerCase())
          })
        )
      )
    } else {
      state.functionData = cloneDeep(functions)
    }
  }
)

const formField = ref()

defineExpose({
  initEdit,
  setFieldForm,
  fieldForm,
  formField
})
const parmasTitle = ref('')
const addParmasToQuota = () => {
  if (disableCaParams.value) return
  parmasTitle.value = t('data_set.add_calculation_parameters')
  if (!fieldForm.params) {
    fieldForm.params = []
  }
  dialogFormVisible.value = true
}

const updateParmasToQuota = () => {
  const [o] = fieldForm.params
  parmasTitle.value = t('data_set.edit_calculation_parameters')
  Object.assign(formQuota, o || {})
  dialogFormVisible.value = true
}

const disableCaParams = computed(() => {
  return !!fieldForm.params?.length
})

const delParmasToQuota = () => {
  const [o] = fieldForm.params
  fieldForm.params = []
  const str = mirror.value.state.doc.toString()
  const name2Auto = []
  fieldForm.originName = setNameIdTrans('name', 'id', str, name2Auto).replaceAll(`[${o.id}]`, '')
  state.quotaData = state.quotaData.filter(ele => ele.id !== o.id)
  mirror.value.dispatch({
    changes: {
      from: 0,
      to: mirror.value.viewState.state.doc.length,
      insert: setNameIdTrans('id', 'name', fieldForm.originName)
    }
  })
}
initFunction()
</script>

<template>
  <div @keydown.stop @keyup.stop class="calcu-field">
    <div class="calcu-cont">
      <div style="flex: 1">
        <div style="max-width: 488px">
          <el-form
            require-asterisk-position="right"
            ref="formField"
            @keydown.stop.prevent.enter
            :model="fieldForm"
            label-position="top"
          >
            <el-form-item
              prop="name"
              :rules="[
                {
                  required: true,
                  message: t('dataset.input_edit_name')
                },
                {
                  max: 50,
                  message: t('commons.char_can_not_more_50')
                }
              ]"
              :label="t('dataset.field_edit_name')"
            >
              <el-input v-model="fieldForm.name" :placeholder="t('dataset.input_edit_name')" />
            </el-form-item>
          </el-form>
          <div>
            <el-form label-position="top" ref="form" inline :model="fieldForm">
              <el-form-item class="mr12" :label="t('dataset.data_type')">
                <div class="btn-select">
                  <el-button
                    @click="fieldForm.groupType = 'd'"
                    :class="[fieldForm.groupType === 'd' && 'is-active']"
                    text
                  >
                    {{ t('chart.dimension_abb') }}
                  </el-button>
                  <el-button
                    @click="fieldForm.groupType = 'q'"
                    :class="[fieldForm.groupType === 'q' && 'is-active']"
                    text
                  >
                    {{ t('chart.quota_abb') }}
                  </el-button>
                </div>
              </el-form-item>
              <el-form-item class="mr0" :label="t('dataset.field_type')">
                <el-select v-model="fieldForm.deType" style="width: 376px">
                  <template #prefix>
                    <el-icon>
                      <Icon
                        ><component
                          class="svg-icon"
                          :class="`field-icon-${fieldType[fieldForm.deType]}`"
                          :is="iconFieldMap[fieldType[fieldForm.deType]]"
                        ></component
                      ></Icon>
                    </el-icon>
                  </template>
                  <el-option
                    v-for="item in fields"
                    :key="item.value"
                    :label="item.label"
                    :value="item.value"
                  >
                    <span style="display: flex; align-items: center">
                      <el-icon>
                        <Icon
                          ><component
                            class="svg-icon"
                            :class="`field-icon-${fieldType[item.value]}`"
                            :is="iconFieldMap[fieldType[item.value]]"
                          ></component
                        ></Icon>
                      </el-icon>
                    </span>
                    <span style="margin-left: 5px; font-size: 12px; color: #8492a6">{{
                      item.label
                    }}</span>
                  </el-option>
                </el-select>
              </el-form-item>
            </el-form>
          </div>
          <div class="mb8 field-exp">
            <span>{{ t('dataset.field_exp') }}</span>
            <span>*</span>
            <el-tooltip class="item" effect="dark" placement="top">
              <template #content>
                <div v-if="props.crossDs">{{ t('dataset.calc_tips.tip1') }}</div>
                <div v-else>{{ t('dataset.calc_tips.tip1_1') }}</div>
                <div>{{ t('dataset.calc_tips.tip2') }}</div>
              </template>
              <el-icon size="16px">
                <Icon name="icon_info_outlined"><icon_info_outlined class="svg-icon" /></Icon>
              </el-icon>
            </el-tooltip>
          </div>
          <code-mirror
            :quotaMap="state.quotaData.map(ele => ele.name)"
            :dimensionMap="state.dimensionData.map(ele => ele.name)"
            ref="myCm"
            height="318px"
            dom-id="calcField"
          ></code-mirror>
        </div>
      </div>
      <div class="padding-lr">
        <span class="mb8">
          {{ t('dataset.click_ref_field') }}
          <el-tooltip class="item" effect="dark" placement="bottom">
            <template #content>
              {{ t('dataset.calc_tips.tip3') }}
              <br />
              {{ t('dataset.calc_tips.tip4') }}
              <br />
              {{ t('dataset.calc_tips.tip5') }}
            </template>
            <el-icon size="16px">
              <Icon name="icon_info_outlined"><icon_info_outlined class="svg-icon" /></Icon>
            </el-icon>
          </el-tooltip>
        </span>
        <div class="padding-lr-content">
          <el-input v-model="searchField" :placeholder="t('dataset.edit_search')" clearable>
            <template #prefix>
              <el-icon>
                <Icon name="icon_search-outline_outlined"
                  ><icon_searchOutline_outlined class="svg-icon"
                /></Icon>
              </el-icon>
            </template>
          </el-input>
          <div class="field-height">
            <el-scrollbar>
              <span>{{ t('chart.dimension') }}</span>
              <div v-if="state.dimensionData.length" class="field-list">
                <span
                  v-for="item in state.dimensionData"
                  :key="item.id"
                  class="item-dimension flex-align-center"
                  :title="item.name"
                  @click="insertFieldToCodeMirror('[' + item.name + ']')"
                >
                  <el-icon>
                    <Icon
                      ><component
                        class="svg-icon"
                        :class="`field-icon-${fieldType[item.deType]}`"
                        :is="iconFieldMap[fieldType[item.deType]]"
                      ></component
                    ></Icon>
                  </el-icon>
                  <span class="ellipsis" :title="item.name">{{ item.name }}</span>
                </span>
              </div>

              <div v-else class="class-na">{{ t('dataset.na') }}</div>
            </el-scrollbar>
          </div>
          <div class="quota-btn_de">
            <span>{{ t('chart.quota') }}</span>
            <el-tooltip
              effect="dark"
              :content="
                disableCaParams
                  ? t('data_set.parameter_is_supported')
                  : t('data_set.add_calculation_parameters')
              "
              placement="top"
            >
              <el-icon class="hover-icon_quota" @click="addParmasToQuota">
                <Icon
                  :class="[`field-icon-${fieldType[0]}`, disableCaParams && 'not-allow']"
                  style="color: #646a73"
                  name="icon_adjustment_outlined"
                  ><icon_adjustment_outlined class="svg-icon"
                /></Icon>
              </el-icon>
            </el-tooltip>
          </div>
          <div class="field-height">
            <el-scrollbar>
              <div v-if="state.quotaData.length" class="field-list">
                <span
                  v-for="item in state.quotaData"
                  :key="item.id"
                  class="item-quota flex-align-center"
                  @click="insertFieldToCodeMirror('[' + item.name + ']')"
                >
                  <el-icon v-if="!item.groupType">
                    <Icon name="icon_adjustment_outlined"
                      ><icon_adjustment_outlined class="svg-icon"
                    /></Icon>
                  </el-icon>
                  <el-icon v-else>
                    <Icon
                      ><component
                        class="svg-icon"
                        :class="`field-icon-${fieldType[item.deType]}`"
                        :is="iconFieldMap[fieldType[item.deType]]"
                      ></component
                    ></Icon>
                  </el-icon>
                  <span class="ellipsis" :title="item.name">{{ item.name }}</span>
                  <div v-if="!item.groupType" class="icon-right">
                    <el-icon @click.stop="updateParmasToQuota" class="hover-icon">
                      <Icon name="icon_edit_outlined"><icon_edit_outlined class="svg-icon" /></Icon>
                    </el-icon>
                    <el-icon @click.stop="delParmasToQuota" class="hover-icon">
                      <Icon name="icon_delete-trash_outlined"
                        ><icon_deleteTrash_outlined class="svg-icon"
                      /></Icon>
                    </el-icon>
                  </div>
                </span>
              </div>
              <div v-else class="class-na">{{ t('dataset.na') }}</div>
            </el-scrollbar>
          </div>
        </div>
      </div>
      <div class="padding-lr">
        <span class="mb8">
          {{ t('dataset.click_ref_function') }}
          <el-tooltip class="item" effect="dark" placement="bottom">
            <template #content>
              <div v-if="props.crossDs">
                {{ t('dataset.calc_tips.tip6') }}
                <br />
                {{ t('dataset.calc_tips.tip8') }}
                <br />
                https://calcite.apache.org/docs/reference.html
              </div>
              <div v-else>{{ t('dataset.calc_tips.tip7') }}</div>
            </template>
            <el-icon size="16px">
              <Icon name="icon_info_outlined"><icon_info_outlined class="svg-icon" /></Icon>
            </el-icon>
          </el-tooltip>
        </span>
        <div class="padding-lr-content">
          <el-input
            v-model="searchFunction"
            style="margin-bottom: 8px"
            :placeholder="t('dataset.edit_search')"
            clearable
          >
            <template #prefix>
              <el-icon>
                <Icon name="icon_search-outline_outlined"
                  ><icon_searchOutline_outlined class="svg-icon"
                /></Icon>
              </el-icon>
            </template>
          </el-input>
          <el-row class="function-height">
            <div v-if="state.functionData.length" style="width: 100%">
              <el-popover
                v-for="(item, index) in state.functionData"
                :key="index"
                class="function-pop"
                placement="right"
                width="200"
                trigger="hover"
                :open-delay="500"
              >
                <template #reference>
                  <span
                    class="function-style flex-align-center"
                    @click="insertParamToCodeMirror(item.func)"
                    >{{ item.func }}</span
                  >
                </template>
                <p class="pop-title">{{ item.name }}</p>
                <p class="pop-info">{{ item.func }}</p>
                <p class="pop-info">{{ item.desc }}</p>
              </el-popover>
            </div>
            <div v-else class="class-na">{{ t('chart.no_function') }}</div>
          </el-row>
        </div>
      </div>
    </div>
    <el-dialog
      :before-close="formQuotaClose"
      v-model="dialogFormVisible"
      append-to-body
      class="create-dialog"
      :title="t('data_set.add_calculation_parameters')"
      width="500"
    >
      <el-form
        @keydown.stop.prevent.enter
        label-position="top"
        ref="formQuotaRef"
        :model="formQuota"
        :rules="formQuotaRules"
      >
        <el-form-item :label="t('data_set.parameter_name')" prop="name">
          <el-input
            style="width: 100%"
            v-model="formQuota.name"
            :placeholder="t('data_set.enter_1_50_characters')"
          />
        </el-form-item>
        <el-form-item :label="t('data_set.parameter_default_value_de')" prop="value">
          <el-input-number
            style="width: 100%"
            v-model="formQuota.value"
            :placeholder="t('data_set.enter_a_number')"
            controls-position="right"
          />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="formQuotaClose">{{ t('chart.cancel') }}</el-button>
          <el-button type="primary" @click="formQuotaConfirm"> {{ t('chart.confirm') }} </el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<style lang="less" scoped>
.calcu-field {
  .calcu-cont {
    color: #606266;
    font-size: 14px;
    box-sizing: border-box;
    display: flex;
    justify-content: space-between;
  }

  .mr12 {
    margin-right: 12px;
  }

  .mr0 {
    margin-right: 0;

    :deep(.ed-select__prefix::after) {
      display: none;
    }
  }

  .btn-select {
    width: 100px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #ffffff;
    border: 1px solid #bbbfc4;
    border-radius: 4px;

    .is-active {
      background: var(--ed-color-primary-1a, rgba(51, 112, 255, 0.1));
    }

    .ed-button:not(.is-active) {
      color: #1f2329;
    }
    .ed-button.is-text {
      height: 24px;
      width: 44px;
      line-height: 24px;
    }
    .ed-button + .ed-button {
      margin-left: 4px;
    }
  }

  .mb8 {
    margin-bottom: 8px;
    display: inline-flex;
    align-items: center;
    color: #1f2329;

    &.field-exp {
      & > :nth-child(2) {
        margin: 0 -0.67px 0 2px;
        color: #f54a45;
        font-family: var(--de-custom_font, 'PingFang');
        font-size: 14px;
        font-style: normal;
        font-weight: 400;
        line-height: 22px;
      }
    }

    .ed-icon {
      color: #646a73;
      margin-left: 4.67px;
    }
  }
}
.padding-lr {
  margin-left: 12px;
  width: 214px;
  overflow-y: hidden;
  .padding-lr-content {
    padding: 12px;
    border: 1px solid var(--deCardStrokeColor, #dee0e3);
    box-sizing: border-box;
    height: 500px;
    border-radius: 4px;
  }
}
.hover-icon_quota {
  cursor: pointer;
  border-radius: 4px;
  font-size: 16px;
  position: relative;

  &[aria-expanded='true'] {
    &::after {
      content: '';
      position: absolute;
      width: 24px;
      height: 24px;
      background: rgba(31, 35, 41, 0.1);
      border-radius: 4px;
      transform: translate(-50%, -50%);
      top: 50%;
      left: 50%;
    }
  }

  &:hover {
    &::after {
      content: '';
      position: absolute;
      width: 24px;
      height: 24px;
      background: rgba(31, 35, 41, 0.1);
      border-radius: 4px;
      transform: translate(-50%, -50%);
      top: 50%;
      left: 50%;
    }
  }

  &:active {
    &::after {
      content: '';
      position: absolute;
      width: 24px;
      height: 24px;
      background: rgba(31, 35, 41, 0.2);
      border-radius: 4px;
      transform: translate(-50%, -50%);
      top: 50%;
      left: 50%;
    }
  }
}

.quota-btn_de {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: -12px;
  color: #1f2329;
}
.field-height {
  height: calc(50% - 41px);
  margin-top: 12px;
  overflow-y: auto;
  & > :nth-child(1) {
    color: #1f2329;
  }

  .not-allow {
    cursor: not-allowed;
    color: #bbbfc4 !important;
  }
}
.item-dimension,
.item-quota {
  padding: 1px 8px;
  border: solid 1px #dee0e3;
  background-color: white;
  color: #1f2329;

  .ed-icon {
    font-size: 16px;
    margin-right: 4px;
  }
  height: 28px;
  margin-top: 4px;
  word-break: break-all;
  border-radius: 4px;

  .icon-right {
    display: none;
    margin-left: auto;
    align-items: center;
    .ed-icon {
      margin: 0 0 0 6px;
    }
  }
}

.item-dimension:hover {
  border-color: var(--ed-color-primary, #3370ff);
  background: var(--ed-color-primary-1a, rgba(51, 112, 255, 0.1));
  cursor: pointer;
}

.item-quota {
  .ed-icon {
    color: #04b49c;
  }
}

.item-quota:hover {
  background: rgba(4, 180, 156, 0.1);
  border-color: #04b49c;
  cursor: pointer;
  .icon-right {
    display: flex;
  }
}

.function-style {
  min-height: 28px;
  padding: 0px 8px;
  margin-bottom: 4px;
  border-radius: 4px;
  color: #1f2329;
  &:hover {
    background: rgba(31, 35, 41, 0.1);
  }
}

.function-style:hover {
  border-color: var(--ed-color-primary, #3370ff);
  cursor: pointer;
}
.function-height {
  height: calc(100% - 29px);
  overflow: auto;
  width: calc(100% + 16px);
  margin-left: -8px;
}
.function-pop :deep(.ed-popover) {
  padding: 6px !important;
}
.pop-title {
  margin: 6px 0 0 0;
  font-size: 14px;
  font-weight: 500;
}
.pop-info {
  margin: 6px 0 0 0;
  font-size: 12px;
}

.class-na {
  margin-top: 8px;
  text-align: center;
  font-size: 14px;
  color: var(--deTextDisable);
}
</style>

<style lang="less">
.calcu-field {
  .cm-scroller {
    height: 320px;
    border: 1px solid #bbbfc4;
    border-radius: 4px;
    overflow-y: auto;
    background: #fff;
  }

  .cm-focused {
    outline: none;
  }
}
</style>
