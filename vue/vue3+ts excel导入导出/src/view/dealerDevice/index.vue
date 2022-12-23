<template>
  <div class="app-container workOrder-container">
    <div class="filter-container">
      <el-input
        class="filter-item"
        v-model="listQuery.like"
        :placeholder="t('dealerDevice.searchField')"
        @change="resetFetchData"
      />
      <el-button type="primary" @click="resetFetchData">
        {{ t('common.search_btn') }}
      </el-button>
      <el-button
        type="primary"
        @click="handleAdd"
        v-if="checkPermission(all_btn_code.dealerDevice.add)"
      >
        {{ t('common.add_btn') }}
      </el-button>
      <el-button
        type="primary"
        @click="handleImportBatch"
        v-if="checkPermission(all_btn_code.dealerDevice.import)"
      >
        {{ t('common.fileImport_btn') }}
      </el-button>
      <el-button
        type="primary"
        @click="handleExportAll"
        v-if="checkPermission(all_btn_code.dealerDevice.exportAll)"
      >
        {{ t('common.fileExportAll_btn') }}
      </el-button>
    </div>
    <div class="table-content">
      <el-table
        ref="multipleTableNode"
        :data="dealerDeviceData"
        border
        stripe
        v-loading="listLoading"
        @selection-change="handleSelectionChange"
      >
        <el-table-column type="selection" width="40" />
        <el-table-column prop="id" :label="t('dealerDevice.id')" />
        <el-table-column prop="sn" :label="t('dealerDevice.sn')" />
        <el-table-column prop="operator" :label="t('dealerDevice.operator')" />
        <el-table-column
          prop="create_time"
          :label="t('dealerDevice.createTime')"
        >
          <template v-slot="scope">
            {{ parseTime(scope.row.create_time) }}
          </template>
        </el-table-column>
        <el-table-column
          prop="update_time"
          :label="t('dealerDevice.updateTime')"
        >
          <template v-slot="scope">
            {{ parseTime(scope.row.update_time) }}
          </template>
        </el-table-column>
        <el-table-column :label="t('dealerDevice.operation')">
          <template v-slot="{ row }">
            <el-button
              type="text"
              size="default"
              @click="handleDel(row)"
              v-if="checkPermission(all_btn_code.dealerDevice.delete)"
            >
              {{ t('common.delete_btn') }}
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <!-- 分页器 -->
    <Pagination
      :total="totalCount"
      v-model:page="listQuery.page"
      v-model:limit="listQuery.count"
      @pagination="fetchData"
    />
    <!-- 批量删除按钮 -->
    <el-button
      v-if="checkPermission(all_btn_code.dealerDevice.delete_batch)"
      :disabled="multipleSelection.length <= 0"
      :loading="removeLoading"
      type="danger"
      style="margin-top: 10px"
      @click="handleBatchDel"
    >
      {{ t('common.delete_batch_btn') }}</el-button
    >
    <!-- 批量导出按钮 -->
    <el-button
      v-if="checkPermission(all_btn_code.dealerDevice.exportBatch)"
      :disabled="multipleSelection.length <= 0"
      type="primary"
      style="margin-top: 10px"
      @click="handleExportBatch"
    >
      {{ t('common.export_batch_btn') }}
    </el-button>
    <!-- 新增 弹窗 -->
    <AddForm ref="operationAddForm" @fetchData="fetchData" />
    <!-- 文件导入 弹窗 -->
    <BatchForm ref="operationBatchForm" @fetchData="fetchData" />
  </div>
</template>
<script setup lang="ts">
import Pagination from '@/components/Pagination/index.vue'
import { reactive, onMounted, ref, watch, nextTick } from 'vue'
import { getDealerDevice, delDealerDevice } from '@/api/dealerDevice'
import { dealerDeviceRowType } from '~/api'
import { useI18n } from 'vue-i18n'
import { parseTime, formatJson } from '@/utils/index'
import { ElMessage, ElMessageBox } from 'element-plus'
import { exportJson2Excel } from '@/utils/excel'
import AddForm, { addForm } from './components/addForm.vue'
import BatchForm, { batchForm } from './components/batchForm.vue'
import { all_btn_code, checkPermission } from '@/utils/checkBtnPermission'

const { t } = useI18n()
const operationAddForm = ref<addForm | null>(null)
const operationBatchForm = ref<batchForm | null>(null)
let dealerDeviceData = ref<Array<dealerDeviceRowType>>([])
let listQuery = reactive({
  page: 1,
  count: 10,
  like: ''
})
const dataMap = reactive({
  list: dealerDeviceData,
  listLoading: true,
  downloadLoading: false,
  filename: '经销商出厂表',
  autoWidth: true,
  bookType: 'xlsx'
})
// table选中
let multipleSelection = ref<Array<dealerDeviceRowType>>([])
let totalCount = ref(0)
let listLoading = ref(false)

let removeLoading = ref(false)
// 弹窗表单内容
let form = reactive({
  sn: [],
  operator: ''
})

watch(totalCount, () => {
  if (
    totalCount.value === (listQuery.page - 1) * listQuery.count &&
    totalCount.value !== 0
  ) {
    listQuery.page -= 1
    fetchData()
  }
})

onMounted(() => {
  fetchData()
})
// 打开新增弹窗
function handleAdd() {
  nextTick(() => {
    if (operationAddForm.value !== null) {
      operationAddForm.value.formVisible = true
    }
  })
}

// 打开文件导入弹窗
function handleImportBatch() {
  nextTick(() => {
    if (operationBatchForm.value !== null) {
      operationBatchForm.value.formVisible = true
    }
  })
}

// 文件全部导出
function handleExportAll() {
  const tHeader = ['id', 'sn', 'operator', 'create_time', 'update_time']
  const filterVal = [...tHeader]
  const list = Object.assign({}, listQuery)
  list.count = totalCount.value
  list.page = 1
  let dateList = ref<Array<dealerDeviceRowType>>([])
  getDealerDevice(list).then((res) => {
    const { result } = res
    dateList.value = result.record ?? []
    const data = formatJson(filterVal, dateList.value)
    exportJson2Excel(
      tHeader,
      data,
      dataMap.filename !== '' ? dataMap.filename : undefined
    )
  })
}
// 批量导出
function handleExportBatch() {
  // const from = unref(multipleTableNode)
  if (multipleSelection.value.length) {
    const tHeader = ['id', 'sn', 'operator', 'create_time', 'update_time']
    const filterVal = [...tHeader]
    const list = multipleSelection.value
    console.log(list)
    const data = formatJson(filterVal, list)
    exportJson2Excel(
      tHeader,
      data,
      dataMap.filename !== '' ? dataMap.filename : undefined
    )
  } else {
    ElMessage.warning('Please select at least one item')
  }
}
// table选中处理
function handleSelectionChange(val: Array<dealerDeviceRowType>) {
  multipleSelection.value = val
}
// 批量删除
function handleBatchDel() {
  ElMessageBox.confirm(t('dealerDevice.tipSelected'), 'Tip', {
    confirmButtonText: t('dealerDevice.confirm_btn'),
    cancelButtonText: t('dealerDevice.cancel_btn'),
    type: 'warning'
  })
    .then(() => {
      let data: { sn: string[] } = {
        sn: []
      }
      multipleSelection.value.forEach((item) => {
        data.sn.push(item.sn)
      })
      delDealerDevice(data).then(() => {
        ElMessage({
          type: 'success',
          message: t('dealerDevice.tipMessageCom')
        })
        fetchData()
      })
    })
    .catch(() => {
      ElMessage({
        type: 'info',
        message: t('dealerDevice.tipMessageCan')
      })
    })
}

// 获取经销商设备列表
function fetchData() {
  listLoading.value = true
  getDealerDevice(listQuery)
    .then((res) => {
      const { result } = res
      dealerDeviceData.value = result.record ?? []
      console.log(dealerDeviceData.value)
      totalCount.value = result.total_count
    })
    .finally(() => {
      listLoading.value = false
    })
}

// 查询
function resetFetchData() {
  listQuery.page = 1
  fetchData()
}
// 删除
function handleDel(row: dealerDeviceRowType) {
  ElMessageBox.confirm(t('dealerDevice.tip') + `(${row.sn})?`, 'Tips', {
    confirmButtonText: t('dealerDevice.confirm_btn'),
    cancelButtonText: t('dealerDevice.cancel_btn'),
    type: 'warning'
  })
    .then(() => {
      let data = {
        sn: [row.sn]
      }
      delDealerDevice(data).then(() => {
        ElMessage({
          type: 'success',
          message: t('dealerDevice.tipMessageCom')
        })
        fetchData()
      })
    })
    .catch(() => {
      ElMessage({
        type: 'info',
        message: t('dealerDevice.tipMessageCan')
      })
    })
}
</script>
