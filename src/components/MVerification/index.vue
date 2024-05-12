<script setup lang="ts">
import {getCodeImg} from '@/api/login';
import {codeDataType} from "./index";

interface DialogType {
  modelValue: boolean,
  resultMethod: Function | undefined
}

const visibleDialog = ref<boolean>(false)
const props = withDefaults(defineProps<DialogType>(), {
  modelValue: false,
  resultMethod: new Function()
})
const emit = defineEmits(['update:modelValue'])
const modelValue = computed(() => {
  console.log(visibleDialog.value)
  visibleDialog.value = props.modelValue
});
const codeData = reactive<codeDataType>({
  count: 5,
  image: '',
  time: 0,
  timer: 0,
  text: '刷新',
  code: '',
  uuid: '',
  disabled: true
})

watch(
  () => props.modelValue,
  (newVal, oldVal) => {
    nextTick(() => {
      getCodeDefault()
      visibleDialog.value = newVal
    })
  },
  {
    deep: true,
    immediate: true,
  }
);

function getCodeHandle(event: any) {
  const {clientX, clientY} = event;
  /*使用脚本触发的登录*/
  if (!event.isTrusted || clientX === 0 || clientY === 0) {
    return
  }
  if (props.resultMethod) {
    props.resultMethod(codeData)
  }
}

function handleDialog(open: boolean) {
  emit('update:modelValue', open)
}

/*默认调用的验证码图片接口*/
function getCodeDefault() {
  let nowTime = Date.now()
  let count = codeData.count
  if (nowTime - codeData.time > codeData.count * 1000) {
    getCodeImg().then(res => {
      codeData.image = 'data:image/png;base64,' + res.data.img
      codeData.uuid = res.data.uuid || ''
    })
    codeData.timer = setInterval(() => {
      count--
      codeData.text = String(count)
      if (count <= 0) {
        clearInterval(codeData.timer)
        codeData.text = '刷新'
        codeData.disabled = false
      }
    }, 1000)
    codeData.time = nowTime
    codeData.disabled = true
  }
}
</script>

<template>
  <div>
    <el-dialog
      draggable
      append-to-body
      title="请输入验证码"
      width="400px"
      style="border-radius: 20px;"
      v-model="visibleDialog"
      @open="handleDialog(true)"
      @close="handleDialog(false)"
    >
      <div class="dialog-box">
        <div class="dialog-image-box">
          <div>
            <el-image :src="codeData.image" @click="getCodeDefault">
              <template #error> 加载失败</template>
            </el-image>
          </div>
          <el-input
            v-model="codeData.code"
            size="large"
            auto-complete="off"
            placeholder="按照图片提示，写出对应的验证码"
          >
            <template #prefix>
              <svg-icon icon-class="validCode"/>
            </template>
          </el-input>
        </div>
        <div class="dialog-button-box">
          <div>
            <el-button size="large" type="primary" @click="getCodeHandle">确定</el-button>
          </div>
          <div>
            <el-button
              size="large"
              type="success"
              @click="getCodeDefault"
              :disabled="codeData.disabled">
              {{ codeData.text }}
            </el-button>
          </div>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<style scoped lang="scss">
.dialog-box {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 20px 20px;

  .dialog-image-box {
    display: flex;
    flex-direction: column;
    gap: 20px;

    div:nth-child(1) {
      height: 100px;
      line-height: 100px;
      width: 100%;
      text-align: center;
      background-color: #cde0ff;
      border-radius: 5px;
    }
  }

  .dialog-button-box {
    display: flex;
    flex-wrap: nowrap;
    gap: 20px;

    div:nth-child(1) {
      width: 80%;
    }

    div:nth-child(2) {
      width: 20%;
      min-width: 70px;
    }

    button {
      width: 100%;
    }
  }
}
</style>
