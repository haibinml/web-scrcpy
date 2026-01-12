<template>
  <v-app>
    <!-- 模式切换按钮 -->
    <v-app-bar v-if="!isDeviceMode" density="compact" color="primary">
      <v-btn icon @click="goBackToDevice">
        <v-icon>mdi-arrow-left</v-icon>
      </v-btn>
      <v-app-bar-title>远程观看</v-app-bar-title>
    </v-app-bar>

    <!-- 设备视图 -->
    <DeviceView v-if="isDeviceMode" :room-name="roomName" :current-user="currentUser">
      <template #remote-button>
        <v-btn
          variant="text"
          size="small"
          class="ml-2"
          @click="isDeviceMode = false"
        >
          <v-icon start>mdi-cast-connected</v-icon>
          远程观看
        </v-btn>
      </template>
    </DeviceView>

    <!-- 远程观看视图 -->
    <RemoteView v-else :initial-peer-id="initialPeerId" />
  </v-app>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { VApp, VIcon, VBtn, VAppBar, VAppBarTitle } from 'vuetify/components';
import DeviceView from './views/DeviceView.vue';
import RemoteView from './views/RemoteView.vue';

const roomName = ref('default-room');
const currentUser = ref({
  id: 'default-user',
  name: 'Default User',
});

const isDeviceMode = ref(true);
const initialPeerId = ref('');

// 检查 URL 参数，如果有 remote 参数则自动进入远程观看模式
onMounted(() => {
  const urlParams = new URLSearchParams(window.location.search);
  const remotePeerId = urlParams.get('remote');
  
  if (remotePeerId) {
    initialPeerId.value = remotePeerId;
    isDeviceMode.value = false;
  }
});

// 返回设备模式时清除 URL 参数
function goBackToDevice() {
  isDeviceMode.value = true;
  initialPeerId.value = '';
  
  // 清除 URL 中的 remote 参数
  const url = new URL(window.location.href);
  url.searchParams.delete('remote');
  window.history.replaceState({}, '', url.toString());
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
}
</style>
