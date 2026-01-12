<template>
  <div class="share-button-container">
    <!-- 分享按钮 -->
    <v-btn
      v-if="!isSharing"
      color="primary"
      variant="tonal"
      size="small"
      :loading="connectionState === 'initializing'"
      @click="handleStartShare"
    >
      <v-icon start>mdi-share-variant</v-icon>
      分享屏幕
    </v-btn>

    <!-- 分享中状态 -->
    <div v-else class="sharing-info">
      <v-chip
        color="success"
        variant="flat"
        size="small"
        @click="showShareDialog = true"
      >
        <v-icon start size="small">mdi-broadcast</v-icon>
        分享中
        <v-badge
          v-if="viewerCount > 0"
          :content="viewerCount"
          color="info"
          inline
          class="ml-1"
        />
      </v-chip>

      <v-btn
        variant="text"
        size="x-small"
        color="error"
        class="ml-1"
        @click="handleStopShare"
      >
        <v-icon size="small">mdi-stop</v-icon>
      </v-btn>
    </div>

    <!-- 分享对话框 -->
    <v-dialog v-model="showShareDialog" max-width="450">
      <v-card>
        <v-card-title class="d-flex align-center">
          <v-icon start color="success">mdi-broadcast</v-icon>
          屏幕分享中
        </v-card-title>
        
        <v-card-text>
          <div class="text-body-2 mb-3">
            将以下链接分享给观看者，他们可以直接打开链接观看您的设备屏幕。
          </div>
          
          <!-- 分享链接 -->
          <v-text-field
            :model-value="shareLink"
            readonly
            label="分享链接"
            variant="outlined"
            density="compact"
            hide-details
            class="mb-3"
          >
            <template v-slot:append-inner>
              <v-btn
                icon
                variant="text"
                size="small"
                @click="copyShareLink"
              >
                <v-icon>mdi-content-copy</v-icon>
                <v-tooltip activator="parent" location="top">复制链接</v-tooltip>
              </v-btn>
            </template>
          </v-text-field>

          <!-- Peer ID -->
          <v-text-field
            :model-value="peerId"
            readonly
            label="Peer ID"
            variant="outlined"
            density="compact"
            hide-details
            class="mb-3"
          >
            <template v-slot:append-inner>
              <v-btn
                icon
                variant="text"
                size="small"
                @click="copyPeerId"
              >
                <v-icon>mdi-content-copy</v-icon>
                <v-tooltip activator="parent" location="top">复制 ID</v-tooltip>
              </v-btn>
            </template>
          </v-text-field>

          <!-- 观看者数量 -->
          <div class="d-flex align-center text-body-2">
            <v-icon start size="small" color="info">mdi-account-multiple</v-icon>
            当前观看者: {{ viewerCount }} 人
          </div>
        </v-card-text>

        <v-card-actions>
          <v-btn
            color="error"
            variant="text"
            @click="handleStopShare"
          >
            停止分享
          </v-btn>
          <v-spacer />
          <v-btn
            color="primary"
            variant="flat"
            @click="showShareDialog = false"
          >
            确定
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- 错误提示 -->
    <v-snackbar
      v-model="showError"
      color="error"
      timeout="3000"
    >
      {{ error }}
    </v-snackbar>

    <!-- 复制成功提示 -->
    <v-snackbar
      v-model="showCopied"
      color="success"
      timeout="2000"
    >
      已复制到剪贴板
    </v-snackbar>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed } from 'vue';
import { useScreenShare } from '@/composables/use-screen-share';
import scrcpyState from '@/components/Scrcpy/scrcpy-state';

const {
  isSharing,
  peerId,
  viewerCount,
  connectionState,
  error,
  startSharing,
  stopSharing,
} = useScreenShare();

const showError = ref(false);
const showCopied = ref(false);
const showShareDialog = ref(false);

// 生成分享链接
const shareLink = computed(() => {
  if (!peerId.value) return '';
  const baseUrl = window.location.origin + window.location.pathname;
  return `${baseUrl}?remote=${peerId.value}`;
});

// 监听错误
watch(error, (newError) => {
  if (newError) {
    showError.value = true;
  }
});

// 分享成功后自动弹出对话框
watch(isSharing, (sharing) => {
  if (sharing && peerId.value) {
    showShareDialog.value = true;
  }
});

/**
 * 开始分享
 */
async function handleStartShare() {
  const canvas = scrcpyState.getCanvas();
  if (!canvas) {
    showError.value = true;
    return;
  }

  try {
    await startSharing(canvas as HTMLCanvasElement, 30);
  } catch (err) {
    console.error('启动分享失败:', err);
  }
}

/**
 * 停止分享
 */
function handleStopShare() {
  showShareDialog.value = false;
  stopSharing();
}

/**
 * 复制分享链接到剪贴板
 */
async function copyShareLink() {
  if (!shareLink.value) return;

  try {
    await navigator.clipboard.writeText(shareLink.value);
    showCopied.value = true;
  } catch (err) {
    console.error('复制失败:', err);
  }
}

/**
 * 复制 Peer ID 到剪贴板
 */
async function copyPeerId() {
  if (!peerId.value) return;

  try {
    await navigator.clipboard.writeText(peerId.value);
    showCopied.value = true;
  } catch (err) {
    console.error('复制失败:', err);
  }
}
</script>

<style scoped>
.share-button-container {
  display: inline-flex;
  align-items: center;
}

.sharing-info {
  display: inline-flex;
  align-items: center;
}
</style>
