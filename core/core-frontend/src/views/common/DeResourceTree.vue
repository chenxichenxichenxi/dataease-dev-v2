<script setup lang="ts">
import dvDashboardSpineMobile from '@/assets/svg/dv-dashboard-spine-mobile.svg'
import dvDashboardSpineMobileDisabled from '@/assets/svg/dv-dashboard-spine-mobile-disabled.svg'
import icon_add_outlined from '@/assets/svg/icon_add_outlined.svg'
import dvCopyDark from '@/assets/svg/dv-copy-dark.svg'
import dvDelete from '@/assets/svg/dv-delete.svg'
import dvMove from '@/assets/svg/dv-move.svg'
import dvCancelPublish from '@/assets/svg/icon_undo_outlined.svg'
import { treeDraggbleChart } from '@/utils/treeDraggbleChart'
import { debounce } from 'lodash-es'
import dvRename from '@/assets/svg/dv-rename.svg'
import dvDashboardSpine from '@/assets/svg/dv-dashboard-spine.svg'
import dvDashboardSpineDisabled from '@/assets/svg/dv-dashboard-spine-disabled.svg'
import dvScreenSpine from '@/assets/svg/dv-screen-spine.svg'
import dvNewFolder from '@/assets/svg/dv-new-folder.svg'
import icon_fileAdd_outlined from '@/assets/svg/icon_file-add_outlined.svg'
import dvUseTemplate from '@/assets/svg/dv-use-template.svg'
import icon_searchOutline_outlined from '@/assets/svg/icon_search-outline_outlined.svg'
import dvSortAsc from '@/assets/svg/dv-sort-asc.svg'
import dvSortDesc from '@/assets/svg/dv-sort-desc.svg'
import dvFolder from '@/assets/svg/dv-folder.svg'
import icon_operationAnalysis_outlined from '@/assets/svg/icon_operation-analysis_outlined.svg'
import icon_edit_outlined from '@/assets/svg/icon_edit_outlined.svg'
import { onMounted, reactive, ref, toRefs, watch, nextTick, computed } from 'vue'
import {
  copyResource,
  deleteLogic,
  ResourceOrFolder,
  queryShareBaseApi,
  updateBase
} from '@/api/visualization/dataVisualization'
import { ElIcon, ElMessage, ElMessageBox, ElScrollbar } from 'element-plus-secondary'
import { Icon } from '@/components/icon-custom'
import { useEmitt } from '@/hooks/web/useEmitt'
import { HandleMore } from '@/components/handle-more'
import DeResourceGroupOpt from '@/views/common/DeResourceGroupOpt.vue'
import { useEmbedded } from '@/store/modules/embedded'
import { BusiTreeNode, BusiTreeRequest } from '@/models/tree/TreeNode'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import { useAppStoreWithOut } from '@/store/modules/app'
import { storeToRefs } from 'pinia'
import DvHandleMore from '@/components/handle-more/src/DvHandleMore.vue'
import { interactiveStoreWithOut } from '@/store/modules/interactive'
import { useShareStoreWithOut } from '@/store/modules/share'
const shareStore = useShareStoreWithOut()
const interactiveStore = interactiveStoreWithOut()
import { useI18n } from '@/hooks/web/useI18n'
import _ from 'lodash'
import DeResourceCreateOptV2 from '@/views/common/DeResourceCreateOptV2.vue'
import { useCache } from '@/hooks/web/useCache'
import { findParentIdByChildIdRecursive, onInitReady } from '@/utils/canvasUtils'
import { XpackComponent } from '@/components/plugin'
import treeSort, { treeParentWeight } from '@/utils/treeSortUtils'
import router from '@/router'
import { cancelRequestBatch } from '@/config/axios/service'
import { isFreeFolder } from '@/utils/utils'
const { wsCache } = useCache()

const dvMainStore = dvMainStoreWithOut()
const appStore = useAppStoreWithOut()
const embeddedStore = useEmbedded()
const { dvInfo } = storeToRefs(dvMainStore)
const { t } = useI18n()

const props = defineProps({
  curCanvasType: {
    type: String,
    required: true
  },
  showPosition: {
    required: false,
    type: String,
    default: 'preview'
  },
  resourceTable: {
    required: false,
    type: String,
    default: 'core'
  }
})
const defaultProps = {
  children: 'children',
  label: 'name',
  disabled: (data: any) => data.extraFlag1 === 0
}
const mounted = ref(false)
const rootManage = ref(false)
const anyManage = ref(false)
const { curCanvasType, showPosition } = toRefs(props)
const resourceLabel =
  curCanvasType.value === 'dataV' ? t('work_branch.big_data_screen') : t('work_branch.dashboard')
const newResourceLabel =
  curCanvasType.value === 'dataV' ? t('visualization.new_screen') : t('visualization.new_dashboard')
const selectedNodeKey = ref(null)
const filterText = ref(null)
const expandedArray = ref([])
const resourceListTree = ref()
const resourceGroupOpt = ref()
const resourceCreateOpt = ref()
const returnMounted = ref(false)
const state = reactive({
  pWeightMap: {},
  curSortType: 'time_desc',
  resourceTree: [] as BusiTreeNode[],
  originResourceTree: [] as BusiTreeNode[],
  folderMenuList: [
    {
      label: t('visualization.move_to'), //'移动到'
      command: 'move',
      svgName: dvMove
    },
    {
      label: t('visualization.rename'), //'重命名'
      command: 'rename',
      svgName: dvRename
    },
    {
      label: t('visualization.delete'), // 删除
      command: 'delete',
      svgName: dvDelete,
      divided: true
    }
  ],
  sortType: [
    {
      label: t('visualization.time_asc'), //'按时间升序'
      value: 'time_asc'
    },
    {
      label: t('visualization.time_desc'), //'按时间降序'
      value: 'time_desc'
    },
    {
      label: t('visualization.name_asc'), //'按名称升序'
      value: 'name_asc'
    },
    {
      label: t('visualization.name_desc'), //'按名称降序'
      value: 'name_desc'
    }
  ],
  templateCreatePid: 0
})

const dvSvgType = computed(() =>
  curCanvasType.value === 'dashboard' ? dvDashboardSpine : dvScreenSpine
)

const isEmbedded = computed(() => appStore.getIsDataEaseBi || appStore.getIsIframe)

const resourceTypeList = computed(() => {
  const list = [
    {
      label: t('work_branch.new_empty'), //'空白新建',
      svgName: dvSvgType.value,
      command: 'newLeaf'
    },
    {
      label: t('work_branch.new_using_template'),
      svgName: dvUseTemplate,
      command: 'newFromTemplate'
    },
    {
      label: t('work_branch.new_folder'), //'新建文件夹'
      divided: true,
      svgName: dvFolder,
      command: 'newFolder'
    }
  ]
  return list
})
const { handleDrop, allowDrop, handleDragStart } = treeDraggbleChart(
  state,
  'resourceTree',
  curCanvasType.value
)

const menuListWeight = id => {
  const pWeight = state.pWeightMap[id]
  return pWeight < 7 ? menuList : menuListWithCopy
}
const menuListWithCopy = [
  {
    label: t('visualization.cancel_publish'), //取消发布
    command: 'cancelPublish',
    svgName: dvCancelPublish
  },
  {
    label: t('visualization.copy'), //'复制',
    command: 'copy',
    svgName: dvCopyDark,
    divided: true
  },
  {
    label: t('visualization.move_to'), //'移动到',
    command: 'move',
    svgName: dvMove
  },
  {
    label: t('visualization.rename'), //'重命名',
    command: 'rename',
    svgName: dvRename
  },
  {
    label: t('visualization.delete'), //'删除',
    command: 'delete',
    svgName: dvDelete,
    divided: true
  }
]
const menuList = [
  {
    label: t('visualization.cancel_publish'), //取消发布
    command: 'cancelPublish',
    svgName: dvCancelPublish
  },
  {
    label: t('visualization.move_to'), //'移动到',
    command: 'move',
    svgName: dvMove,
    divided: true
  },
  {
    label: t('visualization.rename'), //'重命名',
    command: 'rename',
    svgName: dvRename
  },
  {
    label: t('visualization.delete'), //'删除',
    command: 'delete',
    svgName: dvDelete,
    divided: true
  }
]

const infoId = wsCache.get(curCanvasType.value === 'dashboard' ? 'db-info-id' : 'dv-info-id')
const routerDvId = router.currentRoute.value.query.dvId
const dvId = embeddedStore.dvId || infoId || routerDvId
wsCache.delete(curCanvasType.value === 'dashboard' ? 'db-info-id' : 'dv-info-id')
if (dvId && showPosition.value === 'preview') {
  selectedNodeKey.value = dvId
  returnMounted.value = true
}
const nodeExpand = data => {
  if (data.id) {
    expandedArray.value.push(data.id)
  }
}

const nodeCollapse = data => {
  if (data.id) {
    expandedArray.value.splice(expandedArray.value.indexOf(data.id), 1)
  }
}

const filterNode = (value: string, data: BusiTreeNode) => {
  if (showPosition.value === 'multiplexing' && data.id === dvInfo.value?.id) {
    return false
  }
  if (!value) return true
  return data.name?.toLocaleLowerCase().includes(value.toLocaleLowerCase())
}
//取消之前请求
const cancelPreRequest = () => {
  cancelRequestBatch('/dataVisualization/findById')
  cancelRequestBatch('/chartData/getData')
  cancelRequestBatch('/linkage/getVisualizationAllLinkageInfo/**')
  cancelRequestBatch('/linkJump/queryVisualizationJumpInfo/**')
}

const nodeClick = (data: BusiTreeNode, node) => {
  dvMainStore.setCurComponent({ component: null, index: null })
  if (showPosition.value !== 'multiplexing') {
    dvMainStore.setEditMode('preview')
  }
  if (node.disabled) {
    nextTick(() => {
      // 找到当前高亮的节点，移除高亮样式
      const currentNode = resourceListTree.value.$el.querySelector('.is-current')
      if (currentNode) {
        currentNode.classList.remove('is-current')
      }
      return // 阻止后续逻辑
    })
  } else {
    cancelPreRequest()
    selectedNodeKey.value = data.id
    if (data.leaf) {
      if (!embeddedStore.baseUrl) {
        let url = window.location.href
        const paramName = 'dvId'
        const paramValue = data.id
        // 检查是否已经有查询参数（在哈希部分）
        if (url.includes('?')) {
          const regex = new RegExp(`([?&])${paramName}=[^&]*`)
          if (regex.test(url)) {
            url = url.replace(regex, `$1${paramName}=${paramValue}`)
          } else {
            url += `&${paramName}=${paramValue}`
          }
        } else {
          url += `?${paramName}=${paramValue}`
        }
        window.history.replaceState(
          {
            path: url
          },
          '',
          url
        )
      }
      emit('nodeClick', data)
    } else {
      resourceListTree.value.setCurrentKey(null)
    }
  }
}

const getTree = async () => {
  const request = {
    busiFlag: curCanvasType.value,
    resourceTable: props.resourceTable
  } as BusiTreeRequest
  const isDashboard = curCanvasType.value == 'dashboard'
  await interactiveStore.setInteractive(request)
  const interactiveData = isDashboard ? interactiveStore.getPanel : interactiveStore.getScreen
  const nodeData = interactiveData.treeNodes
  rootManage.value = interactiveData.rootManage
  anyManage.value = interactiveData.anyManage
  if (
    dvInfo.value &&
    dvInfo.value.id &&
    !JSON.stringify(nodeData).includes(dvInfo.value.id) &&
    showPosition.value !== 'multiplexing'
  ) {
    dvMainStore.resetDvInfo()
  }
  let curSortType = sortList[Number(wsCache.get('TreeSort-backend')) ?? 1].value
  curSortType = wsCache.get(`TreeSort-${curCanvasType.value}`) ?? curSortType
  if (nodeData.length && nodeData[0]['id'] === '0' && nodeData[0]['name'] === 'root') {
    state.originResourceTree = nodeData[0]['children'] || []
    sortTypeChange(curSortType)
    afterTreeInit()
    return
  }
  state.originResourceTree = nodeData
  sortTypeChange(curSortType)
  afterTreeInit()
}

const flattedTree = computed<BusiTreeNode[]>(() => {
  return _.filter(flatTree(state.resourceTree), node => node.leaf)
})

const hasData = computed<boolean>(() => flattedTree.value.length > 0)

function flatTree(tree: BusiTreeNode[]) {
  let result = _.cloneDeep(tree)
  _.forEach(tree, node => {
    if (node.children && node.children.length > 0) {
      result = _.union(result, flatTree(node.children))
    }
  })
  return result
}

const afterTreeInit = () => {
  state.pWeightMap = treeParentWeight(state.originResourceTree, rootManage.value ? 9 : 0)
  mounted.value = true
  if (selectedNodeKey.value && returnMounted.value) {
    expandedArray.value = getDefaultExpandedKeys()
    returnMounted.value = false
  }
  onInitReady({ type: curCanvasType.value }, 'resource_tree_init_ready')
  nextTick(() => {
    resourceListTree.value.setCurrentKey(selectedNodeKey.value)
    resourceListTree.value.filter(filterText.value)
    nextTick(() => {
      document.querySelector('.is-current')?.firstChild?.click()
    })
  })
}

const copyLoading = ref(false)
const openType = wsCache.get('open-backend') === '1' ? '_self' : '_blank'
const emit = defineEmits(['nodeClick'])

const operation = (cmd: string, data: BusiTreeNode, nodeType: string) => {
  if (cmd === 'delete') {
    const msg = data.leaf ? '' : t('visualization.delete_tips')
    const tips_label = data.leaf ? resourceLabel : t('visualization.folder')
    ElMessageBox.confirm(t('visualization.delete_warn', [tips_label]), {
      confirmButtonType: 'danger',
      type: 'warning',
      tip: msg,
      autofocus: false,
      showClose: false
    }).then(() => {
      deleteLogic(data.id, curCanvasType.value).then(() => {
        ElMessage.success(t('visualization.delete_success'))
        getTree()
      })
    })
  } else if (cmd === 'cancelPublish') {
    const params = {
      id: data.id,
      nodeType: 'leaf',
      name: data.name,
      type: curCanvasType.value,
      mobileLayout: data?.extraFlag,
      status: 0
    }
    updateBase(params).then(() => {
      data['extraFlag1'] = 0
      if (dvInfo.value.id === data.id) {
        dvMainStore.updateDvInfoCall(0)
      }
      ElMessage.warning(t('visualization.cancel_publish_tips'))
    })
  } else if (cmd === 'edit') {
    resourceEdit(data.id)
  } else if (cmd === 'copy') {
    const targetPid = findParentIdByChildIdRecursive(state.resourceTree, data.id)
    const params: ResourceOrFolder = {
      nodeType: nodeType as 'folder' | 'leaf',
      name: data.name + '-copy',
      type: curCanvasType.value,
      id: data.id,
      pid: targetPid || '0'
    }

    copyLoading.value = true

    copyResource(params)
      .then(data => {
        const baseUrl =
          curCanvasType.value === 'dataV'
            ? `#/dvCanvas?opt=copy&pid=${params.pid}&dvId=${data.data}`
            : `#/dashboard?opt=copy&pid=${params.pid}&resourceId=${data.data}`
        if (isEmbedded.value) {
          embeddedStore.clearState()
          embeddedStore.setPid(params.pid as string)
          embeddedStore.setOpt('copy')
          if (curCanvasType.value === 'dataV') {
            embeddedStore.setDvId(data.data)
          } else {
            embeddedStore.setResourceId(data.data)
          }
          useEmitt().emitter.emit(
            'changeCurrentComponent',
            curCanvasType.value === 'dataV' ? 'VisualizationEditor' : 'DashboardEditor'
          )
          return
        }
        const newWindow = window.open(baseUrl, openType)
        initOpenHandler(newWindow)
      })
      .finally(() => {
        copyLoading.value = false
      })
  } else {
    resourceGroupOpt.value.optInit(nodeType, data, cmd, ['copy'].includes(cmd))
  }
}

const addOperation = (
  cmd: string,
  data?: BusiTreeNode,
  nodeType?: string,
  parentSelect?: boolean
) => {
  // 新建子节点的操作流程为先进行创建 后面选择所在目录
  if (cmd === 'newLeaf') {
    const baseUrl =
      curCanvasType.value === 'dataV' ? '#/dvCanvas?opt=create' : '#/dashboard?opt=create'
    let newWindow = null
    if (isEmbedded.value) {
      embeddedStore.clearState()
      embeddedStore.setOpt('create')
      if (data?.id) {
        embeddedStore.setPid(data?.id as string)
      }
      useEmitt().emitter.emit(
        'changeCurrentComponent',
        curCanvasType.value === 'dataV' ? 'VisualizationEditor' : 'DashboardEditor'
      )
      return
    }
    if (data?.id) {
      newWindow = window.open(baseUrl + `&pid=${data.id}`, openType)
    } else {
      newWindow = window.open(baseUrl, openType)
    }
    initOpenHandler(newWindow)
  } else if (cmd === 'newFromTemplate') {
    const params = {
      curPosition: 'create',
      pid: data?.id,
      templateType: curCanvasType.value === 'dataV' ? 'SCREEN' : 'PANEL'
    }
    resourceCreateOpt.value.optInit(params)
  } else {
    resourceGroupOpt.value.optInit(nodeType, data || {}, cmd, parentSelect)
  }
}

function createNewObject() {
  return addOperation('newLeaf', null, 'leaf', true)
}

const resourceEdit = resourceId => {
  const baseUrl = curCanvasType.value === 'dataV' ? '#/dvCanvas?dvId=' : '#/dashboard?resourceId='
  if (isEmbedded.value) {
    embeddedStore.clearState()
    if (curCanvasType.value === 'dataV') {
      embeddedStore.setDvId(resourceId)
    } else {
      embeddedStore.setResourceId(resourceId)
    }
    useEmitt().emitter.emit(
      'changeCurrentComponent',
      curCanvasType.value === 'dataV' ? 'VisualizationEditor' : 'DashboardEditor'
    )
    return
  }

  const newWindow = window.open(baseUrl + resourceId, openType)
  initOpenHandler(newWindow)
}

const resourceOptFinish = () => {
  getTree()
}

const resourceCreateFinish = templateData => {
  // do create
  wsCache.set(`de-template-data`, JSON.stringify(templateData))
  const baseUrl =
    curCanvasType.value === 'dataV'
      ? '#/dvCanvas?opt=create&createType=template'
      : '#/dashboard?opt=create&createType=template'
  let newWindow = null
  if (isEmbedded.value) {
    embeddedStore.clearState()
    embeddedStore.setOpt('create')
    embeddedStore.setCreateType('template')
    if (state.templateCreatePid) {
      embeddedStore.setPid(state.templateCreatePid as unknown as string)
    }
    useEmitt().emitter.emit(
      'changeCurrentComponent',
      curCanvasType.value === 'dataV' ? 'VisualizationEditor' : 'DashboardEditor'
    )
    return
  }

  if (state.templateCreatePid) {
    newWindow = window.open(baseUrl + `&pid=${state.templateCreatePid}`, openType)
  } else {
    newWindow = window.open(baseUrl, openType)
  }
  initOpenHandler(newWindow)
}

const getParentKeys = (tree, targetKey, parentKeys = []) => {
  for (const node of tree) {
    if (node.id === targetKey) {
      return parentKeys
    }
    if (node.children) {
      const newParentKeys = [...parentKeys, node.id]
      const result = getParentKeys(node.children, targetKey, newParentKeys)
      if (result) {
        return result
      }
    }
  }
  return null
}

const getDefaultExpandedKeys = () => {
  const parentKeys = getParentKeys(state.resourceTree, selectedNodeKey.value)
  if (parentKeys) {
    return [selectedNodeKey.value, ...parentKeys]
  } else {
    return []
  }
}

const sortList = [
  {
    name: t('visualization.time_asc'),
    value: 'time_asc'
  },
  {
    name: t('visualization.time_desc'),
    value: 'time_desc',
    divided: true
  },
  {
    name: t('visualization.name_asc'),
    value: 'name_asc'
  },
  {
    name: t('visualization.name_desc'),
    value: 'name_desc'
  }
]

const sortTypeTip = computed(() => {
  return sortList.find(ele => ele.value === state.curSortType).name
})

const handleSortTypeChange = sortType => {
  state.resourceTree = treeSort(state.originResourceTree, sortType)
  state.curSortType = sortType
  wsCache.set('TreeSort-' + curCanvasType.value, state.curSortType)
}

const sortTypeChange = sortType => {
  state.resourceTree = treeSort(state.originResourceTree, sortType)
  state.curSortType = sortType
}

const proxyAllowDrop = debounce((arg1, arg2) => {
  const flagArray = ['dashboard', 'dataV', 'dataset', 'datasource']
  const flag = flagArray.findIndex(item => item === curCanvasType.value)
  if (flag < 0 || !isFreeFolder(arg2, flag + 1)) {
    return allowDrop(arg1, arg2)
  }
  ElMessage.warning(t('free.save_error'))
  return false
}, 300)

watch(filterText, val => {
  resourceListTree.value.filter(val)
})

const openHandler = ref(null)
const initOpenHandler = newWindow => {
  if (openHandler?.value) {
    const pm = {
      methodName: 'initOpenHandler',
      args: newWindow
    }
    openHandler.value.invokeMethod(pm)
  }
}

const loadInit = () => {
  const historyTreeSort = wsCache.get('TreeSort-' + curCanvasType.value)
  if (historyTreeSort) {
    state.curSortType = historyTreeSort
  }
}

const loadShareBase = () => {
  queryShareBaseApi().then(res => {
    const param = {
      shareDisable: res.data?.disable,
      sharePeRequire: res.data?.peRequire
    }
    shareStore.setData(param)
  })
}
onMounted(() => {
  loadInit()
  getTree()
  loadShareBase()
})

defineExpose({
  rootManage,
  hasData,
  createNewObject,
  mounted
})
</script>

<template>
  <div class="resource-tree">
    <div class="tree-header">
      <div class="icon-methods" v-show="showPosition === 'preview'">
        <span class="title"> {{ resourceLabel }} </span>
        <div v-if="rootManage" class="flex-align-center">
          <el-tooltip :content="t('work_branch.new_folder')" placement="top" effect="dark">
            <el-icon
              class="custom-icon btn"
              style="margin-right: 20px"
              @click="addOperation('newFolder', null, 'folder')"
            >
              <Icon name="dv-new-folder"><dvNewFolder class="svg-icon" /></Icon>
            </el-icon>
          </el-tooltip>

          <el-tooltip :content="newResourceLabel" placement="top" effect="dark">
            <el-dropdown popper-class="menu-outer-dv_popper" trigger="hover">
              <el-icon class="custom-icon btn" @click="addOperation('newLeaf', null, 'leaf', true)">
                <Icon name="icon_file-add_outlined"
                  ><icon_fileAdd_outlined class="svg-icon"
                /></Icon>
              </el-icon>
              <template #dropdown>
                <el-dropdown-menu>
                  <el-dropdown-item @click="addOperation('newLeaf', null, 'leaf', true)">
                    <el-icon :class="`handle-icon color-${curCanvasType}`">
                      <Icon><component class="svg-icon" :is="dvSvgType"></component></Icon>
                    </el-icon>
                    {{ t('work_branch.new_empty') }}
                  </el-dropdown-item>
                  <el-dropdown-item @click="addOperation('newFromTemplate', null, 'leaf', true)">
                    <el-icon class="handle-icon">
                      <Icon name="dv-use-template"><dvUseTemplate class="svg-icon" /></Icon>
                    </el-icon>
                    {{ t('work_branch.new_using_template') }}
                  </el-dropdown-item>
                </el-dropdown-menu>
              </template>
            </el-dropdown>
          </el-tooltip>
        </div>
      </div>
      <el-input
        :placeholder="t('commons.search')"
        v-model="filterText"
        clearable
        class="search-bar"
      >
        <template #prefix>
          <el-icon>
            <Icon name="icon_search-outline_outlined"
              ><icon_searchOutline_outlined class="svg-icon"
            /></Icon>
          </el-icon>
        </template>
      </el-input>
      <el-dropdown @command="handleSortTypeChange" trigger="click">
        <el-icon class="filter-icon-span">
          <el-tooltip :offset="16" effect="dark" :content="sortTypeTip" placement="top">
            <Icon v-if="state.curSortType.includes('asc')" name="dv-sort-asc" class="opt-icon"
              ><dvSortAsc class="svg-icon opt-icon"
            /></Icon>
          </el-tooltip>
          <el-tooltip :offset="16" effect="dark" :content="sortTypeTip" placement="top">
            <Icon v-if="state.curSortType.includes('desc')" name="dv-sort-desc" class="opt-icon"
              ><dvSortDesc class="svg-icon opt-icon"
            /></Icon>
          </el-tooltip>
        </el-icon>
        <template #dropdown>
          <el-dropdown-menu style="width: 246px">
            <template :key="ele.value" v-for="ele in sortList">
              <el-dropdown-item
                class="ed-select-dropdown__item"
                :class="ele.value === state.curSortType && 'selected'"
                :command="ele.value"
              >
                {{ ele.name }}
              </el-dropdown-item>
              <li v-if="ele.divided" class="ed-dropdown-menu__item--divided"></li>
            </template>
          </el-dropdown-menu>
        </template>
      </el-dropdown>
    </div>
    <el-scrollbar class="custom-tree" v-loading="copyLoading">
      <el-tree
        menu
        ref="resourceListTree"
        :default-expanded-keys="expandedArray"
        :data="state.resourceTree"
        :props="defaultProps"
        node-key="id"
        highlight-current
        :expand-on-click-node="true"
        :filter-node-method="filterNode"
        @node-expand="nodeExpand"
        @node-collapse="nodeCollapse"
        @node-click="nodeClick"
        @node-drag-start="handleDragStart"
        :allow-drop="proxyAllowDrop"
        @node-drop="handleDrop"
        draggable
      >
        <template #default="{ node, data }">
          <span class="custom-tree-node" :class="{ 'node-disabled-custom': data.extraFlag1 === 0 }">
            <el-icon style="font-size: 18px" v-if="!data.leaf">
              <Icon name="dv-folder"><dvFolder class="svg-icon" /></Icon>
            </el-icon>
            <el-icon style="font-size: 18px" v-else-if="curCanvasType === 'dashboard'">
              <Icon v-if="data.extraFlag1"
                ><component
                  :is="data.extraFlag ? dvDashboardSpineMobile : dvDashboardSpine"
                ></component
              ></Icon>
              <Icon v-if="!data.extraFlag1"
                ><component
                  :is="data.extraFlag ? dvDashboardSpineMobileDisabled : dvDashboardSpineDisabled"
                ></component
              ></Icon>
            </el-icon>
            <el-icon
              class="icon-screen-new color-dataV"
              :class="{ 'color-dataV': data.extraFlag1, 'color-dataV-disabled': !data.extraFlag1 }"
              style="font-size: 18px"
              v-else
            >
              <Icon name="icon_operation-analysis_outlined"
                ><icon_operationAnalysis_outlined class="svg-icon"
              /></Icon>
            </el-icon>
            <span :title="node.label" class="label-tooltip">
              <el-tooltip
                class="box-item"
                effect="dark"
                :content="t('visualization.publish_tips1')"
                :disabled="data.extraFlag1"
                placement="top-start"
              >
                {{ node.label }}
              </el-tooltip>
            </span>
            <div class="icon-more" v-if="data.weight >= 7 && showPosition === 'preview'">
              <el-icon
                v-on:click.stop
                v-if="data.leaf"
                class="hover-icon"
                @click="resourceEdit(data.id)"
              >
                <Icon><icon_edit_outlined class="svg-icon" /></Icon>
              </el-icon>
              <handle-more
                @handle-command="
                  cmd => addOperation(cmd, data, cmd === 'newFolder' ? 'folder' : 'leaf')
                "
                :menu-list="resourceTypeList"
                :icon-name="icon_add_outlined"
                placement="bottom-start"
                v-if="!data.leaf"
              ></handle-more>
              <dv-handle-more
                @handle-command="cmd => operation(cmd, data, data.leaf ? 'leaf' : 'folder')"
                :node="data"
                :any-manage="anyManage"
                :resource-type="curCanvasType"
                :menu-list="data.leaf ? menuListWeight(data.id) : state.folderMenuList"
              ></dv-handle-more>
            </div>
          </span>
        </template>
      </el-tree>
      <de-resource-group-opt
        :cur-canvas-type="curCanvasType"
        @finish="resourceOptFinish"
        ref="resourceGroupOpt"
      />
      <de-resource-create-opt-v2
        :cur-canvas-type="curCanvasType"
        ref="resourceCreateOpt"
        @finish="resourceCreateFinish"
      ></de-resource-create-opt-v2>
    </el-scrollbar>
  </div>
  <XpackComponent ref="openHandler" jsname="L2NvbXBvbmVudC9lbWJlZGRlZC1pZnJhbWUvT3BlbkhhbmRsZXI=" />
</template>
<style lang="less" scoped>
.filter-icon-span {
  border: 1px solid #bbbfc4;
  width: 32px;
  height: 32px;
  border-radius: 4px;
  color: #1f2329;
  padding: 8px;
  margin-left: 8px;
  font-size: 16px;
  cursor: pointer;

  .opt-icon:focus {
    outline: none !important;
  }
  &:hover {
    background: #f5f6f7;
  }

  &:active {
    background: #eff0f1;
  }
}
.resource-tree {
  padding: 16px 0 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;

  .tree-header {
    padding: 0 16px;
  }

  .icon-methods {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    font-size: 20px;
    font-weight: 500;
    color: var(--TextPrimary, #1f2329);
    padding-bottom: 16px;
    .title {
      margin-right: auto;
      font-size: 16px;
      font-style: normal;
      font-weight: 500;
      line-height: 24px;
    }
    .custom-icon {
      font-size: 20px;
      &.btn {
        color: var(--ed-color-primary);
      }
      &:hover {
        cursor: pointer;
      }
    }
  }
  .search-bar {
    padding-bottom: 10px;
    width: calc(100% - 40px);
  }
}
.title-area {
  margin-left: 6px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.title-area-outer {
  display: flex;
  flex: 1 1 0%;
  width: 0px;
}
.custom-tree-node-list {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 14px;
  padding: 0 8px;
}
.father .child {
  visibility: hidden;
}

.father:hover .child {
  visibility: visible;
}

:deep(.ed-input__wrapper) {
  width: 80px;
}

.custom-tree {
  height: calc(100vh - 148px);
  padding: 0 8px;
}

.custom-tree-node {
  width: calc(100% - 30px);
  display: flex;
  align-items: center;
  box-sizing: content-box;
  padding-right: 4px;

  .label-tooltip {
    width: 100%;
    margin-left: 8.75px;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
  }
  .icon-more {
    margin-left: auto;
    display: none;
  }

  &:hover {
    .label-tooltip {
      width: calc(100% - 78px);
    }

    .icon-more {
      display: inline-flex;
    }
  }

  .icon-screen-new {
    border-radius: 4px;
    color: #fff;
    padding: 3px;
  }
}
</style>

<style lang="less">
.menu-outer-dv_popper {
  min-width: 140px;
  margin-top: -2px !important;

  .ed-icon {
    border-radius: 4px;
  }
}

.sort-type-normal {
  i {
    display: none;
  }
}

.sort-type-checked {
  color: var(--ed-color-primary);
  i {
    display: block;
  }
}

.node-disabled-custom {
  color: rgba(187, 191, 196, 1);
  cursor: not-allowed;
}

.color-dataV-disabled {
  background: #bbbfc4 !important;
}
</style>
