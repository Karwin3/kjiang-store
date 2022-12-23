<template>
  <el-dialog
    v-model="formVisible"
    :title="t('dealerDevice.batchDialog_title')"
    width="768px"
    @close="handleClose(formRef)"
  >
    <el-alert
      title="Tips"
      type="warning"
      style="margin-bottom: 20px"
      :closable="false"
      show-icon
    >
      <template v-slot>
        <p>单个文件不超过2M</p>
        <div>
          必须为Excel文件(只能上传后缀是.xlsx,xls,cvs的文件)
          <el-link type="primary" href="/dealerDeviceTemplate.xlsx"
            >下载模板</el-link
          >
        </div>
      </template>
    </el-alert>

    <el-form ref="formRef" :model="form" label-width="100px" :rules="rules">
      <el-form-item :label="t('dealerDevice.operator')" prop="operator">
        <el-input v-model="form.operator" maxlength="30"></el-input>
      </el-form-item>
      <!-- 文件上传 -->
      <el-form-item :label="t('dealerDevice.batchDialog_file')" required>
        <el-upload
          class="upload-demo"
          ref="uploadFile"
          action="#"
          drag
          multiple
          :limit="1"
          :on-exceed="exceedFile"
          :before-remove="beforeRemove"
          :on-remove="handleRemove"
          :file-list="fileList"
          :http-request="handleUploadFile"
        >
          <el-icon class="el-icon--upload"><upload-filled /></el-icon>
          <div class="el-upload__text">
            Drop file here or
            <em :loading="dataMap.loading">click to upload</em>
          </div>
        </el-upload>
      </el-form-item>
    </el-form>
    <el-alert
      v-show="rowsErr.length ? true : false"
      title="错误提示"
      type="warning"
      :closable="false"
    >
      <p>上传失败，您提交的内容可能存在如下错误：</p>
      <ul>
        <li>sn: 只允许输入字母和数字,最少16位,最多30位</li>
      </ul>
      <el-row :gutter="20">
        <el-col :span="7">错误行号：</el-col>
        <el-col
          :span="4"
          v-for="item in rowsErr"
          :key="item"
          style="maring-left: 20px"
          >{{ item }}
        </el-col>
      </el-row>
    </el-alert>

    <template #footer>
      <span class="dialog-footer">
        <el-button
          :loading="btnLoading"
          type="primary"
          @click="handleDia(formRef)"
          >{{ t('user.confirm') }}</el-button
        >
      </span>
    </template>
  </el-dialog>
  <!-- 重复设备提示 -->
  <el-dialog
    v-model="resultTableVisible"
    title="以下设备已存在"
    @close="snResultClear"
  >
    <el-row :gutter="20">
      <el-col :span="6" v-for="item in resultSN.sn" key="item"
        >{{ item }}
      </el-col>
    </el-row>
  </el-dialog>
</template>
<script setup lang="ts">
import { ref, reactive } from 'vue'
import { addDealerDevice } from '@/api/dealerDevice'
import { FormInstance, FormRules } from 'element-plus'
import { ElMessage, ElMessageBox } from 'element-plus'
import type { UploadProps, UploadUserFile } from 'element-plus'
import { UploadFilled } from '@element-plus/icons-vue'
import { useI18n } from 'vue-i18n'
import * as XLSX from 'xlsx'

export interface batchForm {
  formVisible: boolean
}

const { t } = useI18n()
const emits = defineEmits(['fetchData'])
// 表单规则
const rules = reactive<FormRules>({
  operator: [
    {
      required: true,
      message: t('dealerDevice.please_input_operator'),
      trigger: 'blur'
    }
  ]
})
// 弹窗表单内容
let form = reactive({
  sn: [] as string[],
  operator: ''
})
let formRef = ref<FormInstance>()
let formVisible = ref(false)
let btnLoading = ref(false)
let resultTableVisible = ref(false)
let resultSN = reactive({
  sn: [] as string[]
})

// 文件上传部分
type uploadDataType = {
  sn: string
}
const dataMap = reactive({
  loading: false,
  excelData: {
    header: ref<Array<String>>([]),
    results: ref<Array<uploadDataType>>([])
  }
})
// 错误行号
const rowsErr = ref<Array<number>>([])
// 文档内容格式
const table_key = reactive({
  sn: /^[a-zA-Z0-9]{16,30}$/
})
let fileList = ref<Array<any>>([])

// 重复设备列表清空
function snResultClear() {
  resultSN.sn = []
}
// 文件上传 start
// 调接口上传文件数据
function handleUploadFile(param: any) {
  const file = param.file
  if (!fileValidation(param)) return param.onError()
  readerData(file)
  return param.onSuccess()
}
// 文档格式校验
function fileValidation(param: any) {
  const file = param.file
  const extension = file.name.substring(file.name.lastIndexOf('.') + 1)
  const size = file.size / 1024 / 1024
  const fileType = ['xlsx', 'xls', 'csv'].some(
    (fileType) => fileType === extension
  )
  // 类型
  if (!fileType) {
    ElMessage.warning(t('common.fileFormat'))
    fileList.value = []
    param.onError()
    return false
  }
  // 大小
  if (size > 2) {
    ElMessage.warning(t('dealerDevice.fileSize'))
    fileList.value = []
    param.onError()
    return false
  }
  return true
}
// 获取文件表头
function getHeaderRow(sheet: { [key: string]: any }) {
  const headers: string[] = []
  const range = XLSX.utils.decode_range(sheet['!ref'])
  const R = range.s.r
  // start in the first row
  for (let C = range.s.c; C <= range.e.c; ++C) {
    // walk every column in the range
    const cell = sheet[XLSX.utils.encode_cell({ c: C, r: R })]
    // find the cell in the first row
    let hdr = ''
    if (cell && cell.t) hdr = XLSX.utils.format_cell(cell)
    if (hdr === '') {
      hdr = 'UNKNOWN ' + C // replace with your desired default
    }
    headers.push(hdr.trim())
  }
  return headers
}
// 判断文档表单内容是否含有必填项
function getInclude(arr1: any, arr2: any) {
  let temp = []
  for (const item of arr2) {
    arr1.find((i: any) => i === item) ? temp.push(item) : ''
  }
  return temp.length ? true : false
}
// 存储excel文件数据
function generateData(header: any, results: any) {
  // 文档表头校验
  let tHeader = ['sn']
  if (!getInclude(header, tHeader)) {
    ElMessage.error(t('dealerDevice.fileError'))
    fileList.value = []
    return false
  }
  // 文档内容格式校验
  for (let [index, item] of results.entries()) {
    for (let key in table_key) {
      const reg = table_key[key]
      const value = item[key]
      if (!reg.test(value)) {
        rowsErr.value.push(index + 1)
      }
    }
  }
  if (rowsErr.value.length) {
    return false
  }
  dataMap.excelData.header = header
  dataMap.excelData.results = results
}
// 处理excel文件数据
function readerData(rawFile: File) {
  dataMap.loading = true
  const reader = new FileReader()
  reader.onload = (e) => {
    const data = (e.target as FileReader).result
    const workbook = XLSX.read(data, { type: 'array' })
    const firstSheetName = workbook.SheetNames[0]
    const worksheet = workbook.Sheets[firstSheetName]
    const header = getHeaderRow(worksheet)
    const results: any = XLSX.utils.sheet_to_json(worksheet)
    // 去除空格
    const result_new = []
    for (let item of results) {
      const obj = {}
      for (let key in item) {
        const key_new = key.trim()
        const item_new = item[key].trim()
        obj[key_new] = item_new
      }
      result_new.push(obj)
    }
    generateData(header, result_new)
    dataMap.loading = false
  }
  reader.readAsArrayBuffer(rawFile)
}
// 文件超出个数限制时的钩子
function exceedFile() {
  ElMessage.warning(t('common.only_file'))
}

// 移除文件之前的钩子
const beforeRemove: UploadProps['beforeRemove'] = (uploadFile, uploadFiles) => {
  return ElMessageBox.confirm(t('common.remove_file'), 'Warning', {
    confirmButtonText: t('common.confirm'),
    cancelButtonText: t('common.cancel'),
    type: 'warning'
  }).then(
    () => {
      fileList.value = []
      rowsErr.value = []
      ElMessage({
        type: 'success',
        message: t('dealerDevice.tipMessageCom')
      })
      return true
    },
    () => false
  )
}

// 移除文件
function handleRemove() {
  fileList.value = []
  rowsErr.value = []
}
// 文件上传 end

// 弹窗的确认和取消处理
async function handleDia(formEl: FormInstance | undefined) {
  if (!formEl) return
  await formEl.validate((valid) => {
    if (valid) {
      if (!fileList.value.length) {
        ElMessage.error(t('dealerDevice.select_file'))
        return false
      }
      if (!dataMap.excelData.results.length) {
        ElMessage.error(t('common.fileDataFormat'))
        return false
      }
      btnLoading.value = true
      for (let item of dataMap.excelData.results) {
        form.sn.push(item.sn)
      }
      addDealerDevice(form)
        .then((res) => {
          setTimeout(() => {
            if (res.result) {
              ElMessage({
                type: 'success',
                message: '上传成功，但重复设备上传失败'
              })
              resultTableVisible.value = true
              resultSN.sn = res.result
            } else {
              ElMessage({
                type: 'success',
                message: t('dealerDevice.AddSuccess')
              })
            }
            form.sn = []
            form.operator = ''
            emits('fetchData')
            formVisible.value = false
            btnLoading.value = false
          }, 500)
        })
        .catch(() => {
          setTimeout(() => {
            btnLoading.value = false
          }, 500)
        })
    }
  })
}
// 关闭弹窗
function handleClose(formEl: FormInstance | undefined) {
  if (!formEl) return
  formEl.resetFields()
  handleRemove()
}

defineExpose({
  formVisible
})
</script>
