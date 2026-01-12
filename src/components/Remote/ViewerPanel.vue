<template>
  <div class="viewer-panel">
    <!-- 连接表单 -->
    <div v-if="!isConnected" class="connection-form">
      <v-card flat>
        <v-card-title class="text-h6">
          <v-icon start>mdi-cast-connected</v-icon>
          远程观看
        </v-card-title>
        <v-card-text>
          <v-text-field
            v-model="hostPeerId"
            label="分享 ID"
            placeholder="输入分享端的 Peer ID"
            variant="outlined"
            density="compact"
            :disabled="connectionState === 'connecting'"
            :error-messages="error || undefined"
            hide-details="auto"
            @keyup.enter="handleConnect"
          />
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="primary"
            variant="flat"
            :loading="connectionState === 'connecting'"
            :disabled="!hostPeerId.trim()"
            @click="handleConnect"
          >
            <v-icon start>mdi-connection</v-icon>
            连接
          </v-btn>
        </v-card-actions>
      </v-card>
    </div>

    <!-- 视频显示区域 -->
    <div v-else class="video-container">
      <!-- 状态栏 -->
      <div class="status-bar">
        <v-chip
          color="success"
          variant="flat"
          size="small"
        >
          <v-icon start size="small">mdi-check-circle</v-icon>
          已连接
        </v-chip>
        <v-spacer />
        <v-btn
          color="error"
          variant="text"
          size="small"
          @click="handleDisconnect"
        >
          <v-icon start size="small">mdi-close</v-icon>
          断开
        </v-btn>
      </div>

      <!-- 视频播放器 -->
      <div class="video-wrapper" ref="videoWrapper">
        <video
          ref="videoElement"
          autoplay
          playsinline
          muted
          class="remote-video"
          @pointerdown="touchController.onPointerDown"
          @pointermove="touchController.onPointerMove"
          @pointerup="touchController.onPointerUp"
          @pointercancel="touchController.onPointerCancel"
        />
      </div>

      <!-- 导航按钮 -->
      <div class="navigation-bar">
        <v-btn
          variant="text"
          size="large"
          @click="touchController.sendBackKey"
        >
          <v-icon>mdi-arrow-left</v-icon>
        </v-btn>
        <v-btn
          variant="text"
          size="large"
          @click="touchController.sendHomeKey"
        >
          <v-icon>mdi-circle-outline</v-icon>
        </v-btn>
        <v-btn
          variant="text"
          size="large"
          @click="touchController.sendRecentsKey"
        >
          <v-icon>mdi-square-outline</v-icon>
        </v-btn>
      </div>
    </div>

    <!-- 错误/断开提示 -->
    <v-snackbar
      v-model="showError"
      color="error"
      timeout="5000"
    >
      {{ error }}
      <template v-slot:actions>
        <v-btn
          variant="text"
          @click="showError = false"
        >
          关闭
        </v-btn>
      </template>
    </v-snackbar>

    <!-- 断开连接提示 -->
    <v-snackbar
      v-model="showDisconnected"
      color="warning"
      timeout="3000"
    >
      连接已断开
    </v-snackbar>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, computed } from 'vue';
import { useScreenViewer } from '@/composables/use-screen-viewer';
import { useTouchController } from '@/composables/use-touch-controller';

const props = defineProps<{
  initialPeerId?: string;
}>();

const hostPeerId = ref('');
const showError = ref(false);
const showDisconnected = ref(false);
const videoElement = ref<HTMLVideoElement | null>(null);
const videoWrapper = ref<HTMLDivElement | null>(null);

const {
  isConnected,
  connectionState,
  error,
  remoteStream,
  connect,
  disconnect,
  sendCommand,
} = useScreenViewer();

// 触摸控制器
const touchController = useTouchController(sendCommand, videoElement);

// 如果有初始 Peer ID，自动填充并连接
onMounted(() => {
  if (props.initialPeerId) {
    hostPeerId.value = props.initialPeerId;
    // 延迟一点自动连接，让组件完全挂载
    setTimeout(() => {
      handleConnect();
    }, 500);
  }
});

// 监听错误
watch(error, (newError) => {
  if (newError) {
    showError.value = true;
  }
});

// 监听连接状态
watch(connectionState, (newState, oldState) => {
  if (oldState === 'connected' && newState === 'disconnected') {
    showDisconnected.value = true;
  }
});

// 绑定远程流到视频元素
function bindStreamToVideo() {
  if (videoElement.value && remoteStream.value) {
    console.log('[ViewerPanel] 绑定远程流到视频元素');
    console.log('[ViewerPanel] 视频元素尺寸:', videoElement.value.clientWidth, 'x', videoElement.value.clientHeight);
    
    videoElement.value.srcObject = remoteStream.value;
    
    // 监听视频元素事件
    videoElement.value.onloadedmetadata = () => {
      console.log('[ViewerPanel] 视频元数据已加载:', videoElement.value?.videoWidth, 'x', videoElement.value?.videoHeight);
    };
    
    videoElement.value.onplaying = () => {
      console.log('[ViewerPanel] 视频开始播放');
    };
    
    videoElement.value.onerror = (e) => {
      console.error('[ViewerPanel] 视频错误:', e);
    };
    
    // 确保视频开始播放
    videoElement.value.play().catch((err) => {
      console.warn('[ViewerPanel] 视频自动播放失败:', err);
    });
  }
}

// 监听远程流变化
watch(remoteStream, bindStreamToVideo);

// 监听视频元素挂载（当 isConnected 变为 true 时，视频元素才会渲染）
watch(videoElement, bindStreamToVideo);

/**
 * 连接到分享端
 */
async function handleConnect() {
  if (!hostPeerId.value.trim()) return;

  try {
    await connect(hostPeerId.value.trim());
  } catch (err) {
    console.error('连接失败:', err);
  }
}

/**
 * 断开连接
 */
function handleDisconnect() {
  disconnect();
  hostPeerId.value = '';
}
</script>

<style scoped>
.viewer-panel {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.connection-form {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  padding: 16px;
}

.video-container {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.status-bar {
  display: flex;
  align-items: center;
  padding: 8px 16px;
  background: rgba(0, 0, 0, 0.05);
}

.video-wrapper {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #000;
  overflow: hidden;
  touch-action: none;
}

.remote-video {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  touch-action: none;
}

.navigation-bar {
  display: flex;
  justify-content: space-around;
  padding: 8px;
  background: rgba(0, 0, 0, 0.05);
}
</style>
