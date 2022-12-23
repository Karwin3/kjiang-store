<template>
  <el-dialog
    v-model="formVisible"
    :title="t('dealerDevice.addDialog_title')"
    width="768px"
    @close="handleClose(formRef)"
  >
    <el-form ref="formRef" :model="form" :rules="rules" label-width="100px">
      <el-form-item :label="t('dealerDevice.sn')" prop="sn_first">
        <el-input v-model="form.sn_first" maxlength="30"></el-input>
      </el-form-item>
      <el-form-item :label="t('dealerDevice.operator')" prop="operator">
        <el-input v-model="form.operator" maxlength="30"></el-input>
      </el-form-item>
    </el-form>
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
</template>
<script setup lang="ts">
import { ref, reactive } from 'vue'
import { FormInstance, FormRules, ElMessage } from 'element-plus'
import { addDealerDevice } from '@/api/dealerDevice'
import { useI18n } from 'vue-i18n'
export interface addForm {
  formVisible: boolean
}
const { t } = useI18n()
const emits = defineEmits(['fetchData'])
// 表单规则
const rules = reactive<FormRules>({
  sn_first: [
    {
      required: true,
      message: t('dealerDevice.please_input_sn'),
      trigger: 'blur'
    },
    {
      min: 16,
      max: 30,
      message: t('dealerDevice.please_input_sn_lenght'),
      trigger: 'blur'
    }
  ],
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
  sn: ref<Array<string>>([]),
  sn_first: '',
  operator: ''
})

let formVisible = ref(false)
let formRef = ref<FormInstance>()
let btnLoading = ref(false)

// 关闭弹窗
function handleClose(formEl: FormInstance | undefined) {
  if (!formEl) return
  formEl.resetFields()
}
// 弹窗的确认和取消处理
function handleDia(formEl: FormInstance | undefined) {
  if (!formEl) return
  formEl.validate((valid) => {
    if (valid) {
      btnLoading.value = true
      form.sn.push(form.sn_first)
      addDealerDevice(form)
        .then(() => {
          ElMessage({
            type: 'success',
            message: t('dealerDevice.AddSuccess')
          })
          setTimeout(() => {
            emits('fetchData')
            btnLoading.value = false
            formVisible.value = false
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
defineExpose({
  formVisible
})
</script>
