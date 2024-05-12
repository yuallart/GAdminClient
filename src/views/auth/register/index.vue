<template>
  <div class="register-background-box" id="init-background"></div>
  <div class="register-box">
    <el-form
      ref="registerRef"
      :model="registerForm"
      :rules="registerRules"
      class="register-form-box"
    >
      <h3 class="title">RuoYi-Vue-Plus多租户管理系统</h3>
      <el-form-item prop="tenantId" v-if="tenantEnabled">
        <el-select
          v-model="registerForm.tenantId"
          filterable
          placeholder="请选择/输入公司名称"
          style="width: 100%"
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
          v-model="registerForm.username"
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
          v-model="registerForm.password"
          type="password"
          size="large"
          auto-complete="off"
          placeholder="密码"
          @keyup.enter="handleRegister"
        >
          <template #prefix>
            <svg-icon icon-class="password"/>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item prop="confirmPassword">
        <el-input
          v-model="registerForm.confirmPassword"
          type="password"
          size="large"
          auto-complete="off"
          placeholder="确认密码"
          @keyup.enter="handleRegister"
        >
          <template #prefix>
            <svg-icon icon-class="password"/>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item prop="code" v-if="captchaEnabled">
        <el-input
          size="large"
          v-model="registerForm.code"
          auto-complete="off"
          placeholder="验证码"
          style="width: 63%"
          @keyup.enter="handleRegister"
        >
          <template #prefix>
            <svg-icon icon-class="validCode"/>
          </template>
        </el-input>
        <div class="register-code">
          <img :src="codeUrl" @click="getCode" class="register-code-img"/>
        </div>
      </el-form-item>
      <el-form-item style="width:100%;">
        <el-button
          :loading="loading"
          size="large"
          type="primary"
          style="width:100%;"
          @click.prevent="handleRegister"
        >
          注册
        </el-button>
      </el-form-item>
    </el-form>
    <div class="register-form-box">
      <el-button type="success" size="large" style="width: 100%" @click="toLogin">返回登录</el-button>
    </div>
    <!--  底部  -->
    <div class="el-register-footer">
      <span>Copyright © 2018-2023 疯狂的狮子Li All Rights Reserved.</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import {getCodeImg, register, getTenantList} from '@/api/login';
import {RegisterForm, TenantVO} from '@/api/types';
import {to} from 'await-to-js';
import Trianglify from "trianglify";

const router = useRouter();

const registerForm = ref<RegisterForm>({
  tenantId: "",
  username: "",
  password: "",
  confirmPassword: "",
  code: "",
  uuid: "",
  userType: "sys_user"
});

// 租户开关
const tenantEnabled = ref(true);
let canvasRef: HTMLElement | null = null

const equalToPassword = (rule: any, value: string, callback: any) => {
  if (registerForm.value.password !== value) {
    callback(new Error("两次输入的密码不一致"));
  } else {
    callback();
  }
};

const registerRules: ElFormRules = {
  tenantId: [
    {required: true, trigger: "blur", message: "请输入您的租户编号"}
  ],
  username: [
    {required: true, trigger: "blur", message: "请输入您的账号"},
    {min: 2, max: 20, message: "用户账号长度必须介于 2 和 20 之间", trigger: "blur"}
  ],
  password: [
    {required: true, trigger: "blur", message: "请输入您的密码"},
    {min: 5, max: 20, message: "用户密码长度必须介于 5 和 20 之间", trigger: "blur"}
  ],
  confirmPassword: [
    {required: true, trigger: "blur", message: "请再次输入您的密码"},
    {required: true, validator: equalToPassword, trigger: "blur"}
  ],
  code: [{required: true, trigger: "change", message: "请输入验证码"}]
};
const codeUrl = ref("");
const loading = ref(false);
const captchaEnabled = ref(true);
const registerRef = ref<ElFormInstance>();
// 租户列表
const tenantList = ref<TenantVO[]>([]);

const handleRegister = () => {
  registerRef.value?.validate(async (valid: boolean) => {
    if (valid) {
      loading.value = true;
      const [err] = await to(register(registerForm.value));
      if (!err) {
        const username = registerForm.value.username;
        await ElMessageBox.alert(
          "<div style='color:red'>恭喜你，您的账号 " + username + " 注册成功！</div>", "系统提示",
          {
            dangerouslyUseHTMLString: true,
            type: "success",
          }
        )
        await router.push("/login");
      } else {
        loading.value = false;
        if (captchaEnabled) {
          getCode();
        }
      }
    }
  });
}

const toLogin = () => {
  router.push({path: '/login'})
}

const getCode = async () => {
  const res = await getCodeImg();
  const {data} = res;
  captchaEnabled.value = data.captchaEnabled === undefined ? true : data.captchaEnabled;
  if (captchaEnabled.value) {
    codeUrl.value = 'data:image/gif;base64,' + data.img;
    registerForm.value.uuid = data.uuid;
  }
};

const initTenantList = async () => {
  const {data} = await getTenantList();
  tenantEnabled.value = data.tenantEnabled === undefined ? true : data.tenantEnabled;
  if (tenantEnabled.value) {
    tenantList.value = data.voList;
    if (tenantList.value != null && tenantList.value.length !== 0) {
      registerForm.value.tenantId = tenantList.value[0].tenantId;
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

onMounted(() => {
  getCode();
  initTenantList();
  initBackground()
  window.addEventListener('resize', listerResize);
})
</script>

<style lang="scss" scoped>
$login-padding: 40px;

.register-box {
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

.title {
  margin: 0 auto 30px auto;
  text-align: center;
  color: #707070;
}

.register-form-box {
  width: 400px;
  padding: $login-padding;
  border-radius: 20px;
  background-color: #FFFFFF;
  box-shadow: rgba(0, 0, 0, 0.56) 0 22px 70px 4px;
}


.register-background-box {
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

.register-tip {
  font-size: 13px;
  text-align: center;
  color: #bfbfbf;
}

.register-code {
  width: 33%;
  height: 40px;
  float: right;

  img {
    cursor: pointer;
    vertical-align: middle;
  }
}

.el-register-footer {
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

.register-code-img {
  height: 40px;
  padding-left: 12px;
}
</style>
