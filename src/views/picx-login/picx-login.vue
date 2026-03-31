<template>
  <div
    class="page-container login-container"
    style="align-items: center; justify-content: center; min-height: 80vh"
  >
    <div style="max-width: 360px; text-align: center">
      <div style="margin-bottom: 24px">
        <img
          src="../../assets/logo.png"
          alt="KalePic"
          style="width: 72px; height: 72px; border-radius: 14px"
        />
        <h2 style="margin-top: 14px; color: #333">KalePic 私有图床</h2>
        <p style="color: #999; font-size: 13px">仅限授权用户</p>
      </div>
      <el-button
        type="primary"
        size="large"
        @click="doLogin"
        :loading="loading"
        style="width: 100%; font-size: 16px"
      >
        GitHub 账号登录
      </el-button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, computed, ref } from 'vue'
import axios from 'axios'
import { store } from '@/stores'
import router from '@/router'

const userConfigInfo = computed(() => store.getters.getUserConfigInfo).value
const loading = ref(false)

const redirect_uri = import.meta!.env.VITE_REDIRECT_URI as string
const client_id = import.meta!.env.VITE_CLIENT_ID as string
const authorize_api = 'https://auth-pic.kalejun.com'

const doLogin = () => {
  loading.value = true
  window.location.href = `https://github.com/login/oauth/authorize?client_id=${client_id}&redirect_uri=${redirect_uri}&scope=repo`
}

const handleCallback = async () => {
  const params = new URLSearchParams(window.location.search)
  const code = params.get('code')
  if (!code) return

  loading.value = true
  try {
    const res = await axios.get(`${authorize_api}?code=${code}&redirect_uri=${redirect_uri}`)
    if (res.data?.data) {
      const token = res.data.data

      await store.dispatch('SET_GITHUB_AUTHORIZATION_INFO', {
        authorized: true,
        installed: true,
        token,
        tokenCreateTime: Date.now(),
        isAutoAuthorize: false
      })

      await store.dispatch('SET_USER_CONFIG_INFO', { token })

      const oriUrl = window.location.href.replace(window.location.search, '')
      window.location.href = `${oriUrl}config?auto=1`
    } else {
      ElMessage.error({ message: res.data?.msg || '登录失败', duration: 5000 })
      loading.value = false
    }
  } catch (e) {
    ElMessage.error({ message: '登录失败，请重试', duration: 5000 })
    loading.value = false
  }
}

onMounted(() => {
  // Check if already logged in
  const { token, name, owner, repo, branch, selectedDir: dir, logined } = userConfigInfo
  if (token && name && owner && logined) {
    if (repo && branch && dir) router.push('/upload')
    else router.push('/config')
    return
  }

  // Handle OAuth callback
  handleCallback()
})
</script>

<style scoped lang="stylus">
@import "./picx-login.styl"
</style>
