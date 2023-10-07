<template>
  <main flex flex-col gap-2 pa-4>
    <form @submit.prevent="onSubmit">
      <section flex flex-col items-start gap-2 w-100>
        <h2> 登录 </h2>

        <label>
          用户名
          <input v-model="userName" :disabled="currentUser !== undefined" />
        </label>

        <button :disabled="processing" type="submit">
          提交
        </button>
      </section>
    </form>

    <section v-if="currentUser !== undefined">
      <h2> 当前用户 </h2>
      <p> ID: {{ currentUser.id }} </p>
      <p> 用户名: {{ currentUser.name }} </p>
      <p> imToken: {{ currentUser.imToken }} </p>
    </section>

    <section v-if="currentUser !== undefined">
      <button :disabled="processing" @click="onClick">
        登出
      </button>
    </section>
  </main>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { getSDK } from '@/utils/lib';

const OpenIM = getSDK();

/** 提交中 */
const processing = ref(false)

/** 用户名 */
const userName = ref('')

/** 是否已经登录 */
const currentUser = ref<User | undefined>(undefined)

/** 表单提交回调 */
async function onSubmit() {
  if (processing.value) {
    throw Error('重复调用')
  }
  try {
    processing.value = true

    if (currentUser.value !== undefined) {
      throw new Error('用户已登录')
    }

    // 1. 调用自己系统的登录接口
    const user = await login(userName.value)

    // 2. 用响应数据调用 SDK 的登录接口。
    await imLogin(user)

    currentUser.value = user;
  } catch (error) {
    alert(error)
  } finally {
    processing.value = false
  }
}

interface User {
  id: number
  name: string
  faceUrl: string
  imToken: string
}

/** 登录 */
async function login(userName: string): Promise<User> {
  const req = new Request('/api/login', {
    method: 'POST',
    body: JSON.stringify({ userName, platform: 5 }),
    headers: {
      'Content-Type': 'application/json',
    },
  })

  const resp = await fetch(req)
  if (resp.status != 200) {
    throw new Error(await resp.text())
  }

  return await resp.json()
}

/** im 登录 */
async function imLogin(user: User) {
  await OpenIM.login({
    platformID: 5,
    apiAddr: import.meta.env.VITE_IM_API_ADDR,
    wsAddr: import.meta.env.VITE_IM_WS_ADDR,
    userID: String(user.id),
    token: user.imToken,
  })
}

/** 点击回调 */
async function onClick() {
  if (processing.value) {
    throw new Error('重复调用')
  }
  try {
    processing.value = true
    if (currentUser.value === undefined) {
      throw new Error('当前用户不存在')
    }

    // 1. 调用 sdk 的登录接口。
    await logout(currentUser.value)

    // 2. 调用 SDK 的登出接口
    await OpenIM.logout()
    currentUser.value = undefined
  } catch (error) {
    alert(error)
  } finally {
    processing.value = false
  }
}

/** 登出 */
async function logout(user: User) {
  const req = new Request('/api/logout', {
    method: 'POST',
    body: JSON.stringify({ id: user.id }),
    headers: {
      'Content-Type': 'application/json',
    },
  })

  const resp = await fetch(req)
  if (resp.status != 200) {
    throw new Error(await resp.text())
  }
}
</script>
