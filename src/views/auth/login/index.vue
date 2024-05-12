<template>
  <div class="login-background-box" id="init-background"></div>
  <div class="login-box">
    <el-form ref="loginRef" :model="loginForm" :rules="loginRules" class="login-form-box">
      <div class="login-top-box">
        <div>{{ VITE_APP_TITLE }}</div>
        <div>
          <el-radio-group
            class="login-label-box"
            v-model="commonLoginData"
            size="small">
            <el-radio-button
              v-for="(item,index) in commonLoginDetails"
              :key="index"
              :label="item.value">
              {{ item.text }}
            </el-radio-button>
          </el-radio-group>
        </div>
      </div>
      <el-form-item prop="tenantId" v-if="tenantEnabled">
        <el-select
          class="login-label-box"
          v-model="loginForm.tenantId"
          filterable
          size="large"
          placeholder="请选择/输入公司名称"
        >
          <el-option
            v-for="item in tenantList"
            :key="item.tenantId"
            :label="item.companyName"
            :value="item.tenantId"
          ></el-option>
          <template #prefix>
            <svg-icon icon-class="company"/>
          </template>
        </el-select>
      </el-form-item>
      <el-form-item prop="username">
        <el-input
          class="login-form-input-box"
          v-model="loginForm.username"
          type="text"
          size="large"
          auto-complete="off"
          placeholder="账号"
        >
          <template #prefix>
            <svg-icon icon-class="user"/>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item prop="password">
        <el-input
          v-model="loginForm.password"
          type="password"
          size="large"
          auto-complete="off"
          placeholder="密码"
          @keyup.enter="handleLogin"
        >
          <template #prefix>
            <svg-icon icon-class="password"/>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item>
        <el-checkbox v-model="loginForm.rememberMe" size="large">
          <template #default>
            <div class="login-checkbox-box">
              <div>延长我的登录状态</div>
              <div>（请不要在陌生的设备上勾选）</div>
            </div>
          </template>
        </el-checkbox>
      </el-form-item>
      <el-form-item style="width:100%;">
        <el-button
          size="large"
          type="primary"
          style="width:100%;"
          @click.prevent="handleLogin"
        >
          <span>登 录</span>
        </el-button>
      </el-form-item>
    </el-form>
    <div class="login-register-box">
      <div>
        <el-button
          style="width:100%"
          size="large"
          type="success"
          @click="toRegister"
        >
          <template #default> 立即注册</template>
        </el-button>
      </div>
      <el-divider>第三方登录</el-divider>
      <div>
        <el-scrollbar>
          <div class="login-icon-box">
            <div class="login-icon-item-box" v-for="(item,index) in otherLoginDetail" :key="index">
              <el-button circle :title="item.text" @click="doSocialLogin(item.text)">
                <svg-icon :icon-class="item.icon" :style="{color:item.color,fontSize:'20px'}"/>
              </el-button>
            </div>
          </div>
        </el-scrollbar>
      </div>
    </div>
    <m-verification v-model="visibleDialog" :result-method="handleVerify"></m-verification>
    <!--  底部  -->
    <div class="bottom-text-box">
      <span>Copyright © 2018-2023 疯狂的狮子Li All Rights Reserved.</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import {getCodeImg, getTenantList} from '@/api/login';
import {authBinding} from '@/api/system/social/auth';
import {useUserStore} from '@/store/modules/user';
import {LoginData, TenantVO} from '@/api/types';
import {HttpStatus} from "@/enums/RespEnum";
import Trianglify from 'trianglify'
import {ElMessage} from 'element-plus'
import MVerification from "@/components/MVerification/index.vue"
import {ElLoading} from 'element-plus'

type loginIconType = {
  icon: string,
  text: string,
  color?: string,
  data?: object
}
const VITE_APP_TITLE = ref(import.meta.env.VITE_APP_TITLE)
const userStore = useUserStore();
const router = useRouter();
let canvasRef: HTMLElement | null = null

// 租户开关
const tenantEnabled = ref(true);
const redirect = ref(undefined);
const loginRef = ref<ElFormInstance>();
// 租户列表
const tenantList = ref<TenantVO[]>([]);
const visibleDialog = ref<boolean>(false)
const loginForm = ref<LoginData>({
  tenantId: '000000',
  username: 'admin',
  password: 'admin123',
  code: '',
  uuid: ''
} as LoginData);

const commonLoginData = ref(1)

const commonLoginDetails = reactive([
  {text: '用户名登录', value: 1},
  {text: '验证码登录', value: 2},
  {text: '扫码登录', value: 4},
])

const otherLoginDetail = ref<Array<loginIconType>>([
  {
    text: '微信登录',
    icon: 'wechat',
    data: {},
    color: '#18cc21'
  },
  {
    text: 'MaxKey登录',
    icon: 'maxkey',
    data: {}
  },
  {
    text: 'Gitee登录',
    icon: 'gitee',
    data: {}
  },
  {
    text: 'Github登录',
    icon: 'github',
    data: {}
  },
])

const loginFromDetails = reactive([
  {model: 'input', prop: 'username'},
  {model: 'input', prop: 'password'},
])

const loginRules: ElFormRules = {
  tenantId: [{required: true, trigger: "blur", message: "请输入您的租户编号"}],
  username: [{required: true, trigger: 'blur', message: '请输入您的账号'}],
  password: [{required: true, trigger: 'blur', message: '请输入您的密码'}],
  code: [{required: true, trigger: 'change', message: '请输入验证码'}]
};

watch(
  () => router.currentRoute.value,
  (newRoute: any) => {
    redirect.value = newRoute.query && newRoute.query.redirect;
  }, {immediate: true});


const handleLogin = () => {
  loginRef.value?.validate((valid: boolean, fields: any) => {
    if (valid) {
      visibleDialog.value = true
    } else {
      console.log('error submit!', fields);
    }
  });
};

const handleVerify = (codeData: any) => {
  const {code, uuid} = codeData
  if (code && uuid) {
    loginForm.value.uuid = uuid
    loginForm.value.code = code
    toHome()
  }
}

const toHome = async () => {
  const loading = ElLoading.service({
    lock: true,
    text: 'Loading',
    background: 'rgba(0, 0, 0, 0.7)',
  })
  userStore.login(loginForm.value).then(res => {
    setTimeout(() => {
      router.push({path: redirect.value || '/'});
      loading.close()
    }, 1000)
  }).catch(err => {
    ElMessage.error('登录失败，清重试')
  }).finally(() => {
    setTimeout(() => {
      loading.close()
    }, 1000)
  })
}

const toRegister = () => {
  router.push({path: '/register'})
}

/**
 * 获取租户列表
 */
const initTenantList = async () => {
  const {data} = await getTenantList();
  tenantEnabled.value = data.tenantEnabled === undefined ? true : data.tenantEnabled;
  if (tenantEnabled.value) {
    tenantList.value = data.voList;
    if (tenantList.value != null && tenantList.value.length !== 0) {
      loginForm.value.tenantId = tenantList.value[0].tenantId;
    }
  }
}

function initBackground() {
  const div: HTMLElement | null = document.getElementById('init-background');
  const pattern = Trianglify({
    width: window.innerWidth,
    height: window.innerHeight
  });
  canvasRef = pattern.toCanvas();
  if (div && canvasRef) {
    div.appendChild(canvasRef)
  }
}

function listerResize() {
  if (canvasRef === null) {
    return console.log('no element found')
  }
  canvasRef.style.width = window.innerWidth + 'px'
  canvasRef.style.height = window.innerHeight + 'px'
}

//检测租户选择框的变化
watch(() => loginForm.value.tenantId, () => {
  localStorage.setItem("tenantId", String(loginForm.value.tenantId))
});

/**
 * 第三方登录
 * @param type
 */
const doSocialLogin = (type: string) => {
  authBinding(type).then((res: any) => {
    if (res.code === HttpStatus.SUCCESS) {
      // 获取授权地址跳转
      window.location.href = res.data;
    } else {
      ElMessage.error(res.msg);
    }
  });
};

onMounted(() => {
  initBackground()
  initTenantList();
  window.addEventListener('resize', listerResize);
});
</script>

<style lang="scss" scoped>
$login-padding: 40px;

.login-box {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  gap: 20px;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.login-background-box {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  background-size: cover;
}

.login-top-box {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 10px;

  div:nth-child(1) {
    font-size: 20px;
    text-align: center;
  }
}

.login-form-box {
  width: 400px;
  padding: $login-padding;
  border-radius: 20px;
  background-color: #FFFFFF;
  box-shadow: rgba(0, 0, 0, 0.56) 0 22px 70px 4px;

  .input-icon {
    height: 39px;
    width: 14px;
    margin-left: 0px;
  }
}

.login-label-box {
  width: 100%;
}

.login-register-box {
  width: 400px;
  padding: $login-padding;
  border-radius: 20px;
  background-color: #FFFFFF;
  box-shadow: rgba(0, 0, 0, 0.56) 0 22px 70px 4px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.login-tip {
  font-size: 13px;
  text-align: center;
  color: #bfbfbf;
}

.el-login-footer {
  height: 40px;
  line-height: 40px;
  position: fixed;
  bottom: 0;
  width: 100%;
  text-align: center;
  color: #fff;
  font-family: Arial, serif;
  font-size: 12px;
  letter-spacing: 1px;
}


.login-icon-box {
  display: flex;
  overflow-x: auto;
  justify-content: space-between;
  padding: 0 5px;
  gap: 10px;

  .login-icon-item-box {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    height: 50px;

    .el-button {
      width: 35px;
      height: 35px;
    }
  }
}

.login-checkbox-box {
  display: flex;
  flex-wrap: nowrap;

  div:nth-child(2) {
    color: var(--el-color-danger)
  }

  div {
    font-size: 12px;
  }
}

.bottom-text-box {
  position: absolute;
  color: #FFFFFF;
  bottom: 5px;
}

.login-read-box {
  display: flex;
  flex-direction: column;
}
</style>
