<template>
  <div class="page-container login-container" style="justify-content: center; align-items: center; min-height: 80vh;">
    <div style="max-width: 360px; text-align: center;">
      <div style="margin-bottom: 24px;">
        <img src="../../assets/logo.png" alt="KalePic" style="width: 72px; height: 72px; border-radius: 14px;" />
        <h2 style="margin-top: 14px; color: #333;">KalePic 私有图床</h2>
        <p style="color: #999; font-size: 13px;">仅限授权用户</p>
      </div>
      <el-button
        v-if="!loading"
        type="primary"
        size="large"
        @click="doLogin"
        style="width: 100%; font-size: 16px;"
      >
        GitHub 账号登录
      </el-button>
      <div v-else style="color: #409eff;">
        <el-icon class="is-loading" :size="24"><IEpLoading /></el-icon>
        <p style="margin-top: 8px;">正在登录...</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, computed, ref } from "vue"
import { store } from "@/stores"
import router from "@/router"
import axios from "axios"

const FIXED_CONFIG = {
  owner: "zahneehk",
  repo: "kalepic-images-hosting",
  branch: "master",
  selectedDir: "/",
  dirMode: "repoDir",
  logined: true,
  repoPrivate: false
}

const userConfigInfo = computed(() => store.getters.getUserConfigInfo).value
const loading = ref(false)

const redirect_uri = import.meta!.env.VITE_REDIRECT_URI as string
const client_id = import.meta!.env.VITE_CLIENT_ID as string
const authorize_api = "https://auth-pic.kalejun.com"

const doLogin = () => {
  loading.value = true
  window.location.href = `https://github.com/login/oauth/authorize?client_id=${client_id}&redirect_uri=${redirect_uri}&scope=repo`
}

const autoConfig = async (token: string) => {
  // Get user info from GitHub
  try {
    const userRes = await axios.get("https://api.github.com/user", {
      headers: { Authorization: `Bearer ${token}` }
    })
    const user = userRes.data

    await store.dispatch("SET_USER_CONFIG_INFO", {
      token: token,
      id: user.id,
      owner: FIXED_CONFIG.owner,
      email: user.email || "",
      name: user.login,
      avatarUrl: user.avatar_url,
      repo: FIXED_CONFIG.repo,
      branch: FIXED_CONFIG.branch,
      selectedDir: FIXED_CONFIG.selectedDir,
      dirMode: FIXED_CONFIG.dirMode,
      logined: true,
      repoPrivate: FIXED_CONFIG.repoPrivate
    })

    await store.dispatch("SET_GITHUB_AUTHORIZATION_INFO", {
      authorized: true,
      installed: true,
      token: token,
      tokenCreateTime: Date.now(),
      isAutoAuthorize: false
    })


    // Auto-set deploy status as deployed
    await store.dispatch("SET_DEPLOY_STATUS_INFO", {
      github: {
        status: true,
        latestTime: Date.now(),
        type: "githubPages"
      }
    })
    router.push("/upload")
  } catch {
    loading.value = false
  }
}

const handleCallback = async () => {
  const params = new URLSearchParams(window.location.search)
  const code = params.get("code")
  if (!code) return

  loading.value = true
  try {
    const res = await axios.get(`${authorize_api}?code=${code}&redirect_uri=${redirect_uri}`)
    if (res.data?.data) {
      // Clean URL
      window.history.replaceState({}, document.title, redirect_uri)
      await autoConfig(res.data.data)
    } else {
      loading.value = false
    }
  } catch {
    loading.value = false
  }
}

onMounted(() => {
  // Already logged in with config? Go straight to upload
  const { token, owner, repo, logined } = userConfigInfo
  if (token && owner && repo && logined) {

    store.dispatch("SET_DEPLOY_STATUS_INFO", { github: { status: true, latestTime: Date.now(), type: "githubPages" } })
    router.push("/upload")
    return
  }

  // Handle OAuth callback
  handleCallback()
})
</script>

<style scoped lang="stylus">
@import "./picx-login.styl"
</style>
