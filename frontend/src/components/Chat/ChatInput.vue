<template>
  <div class="input-area-wrapper">
    <!-- File upload indicator -->
    <div v-if="uploadState.file" class="upload-preview-bar">
      <span class="upload-file-info">
        <i :class="getFileIcon(uploadState.file.name)"></i>
        {{ uploadState.file.name }}
      </span>
      <button v-if="uploadState.status === 'idle'" @click="startUpload" class="btn-upload">
        <i class="fas fa-upload"></i> 上传
      </button>
      <button v-if="uploadState.status === 'idle'" @click="cancelUpload" class="btn-cancel-upload">
        <i class="fas fa-times"></i>
      </button>
      <span v-if="uploadState.status === 'uploading'" class="upload-status">{{ uploadState.message }}</span>
      <span v-if="uploadState.status === 'processing'" class="upload-status processing">{{ uploadState.message }}</span>
      <span v-if="uploadState.status === 'done'" class="upload-status done">{{ uploadState.message }}</span>
      <span v-if="uploadState.status === 'error'" class="upload-status error">{{ uploadState.message }}</span>
    </div>

    <!-- Progress bar -->
    <div v-if="uploadState.status === 'uploading'" class="upload-bar-track">
      <div class="upload-bar-fill" :style="{ width: uploadState.uploadPercent + '%' }"></div>
    </div>
    <div v-if="uploadState.status === 'processing'" class="upload-bar-track">
      <div class="upload-bar-fill processing" :style="{ width: uploadState.processPercent + '%' }"></div>
    </div>

    <div class="input-area">
      <button class="attach-btn" @click="triggerFileSelect" title="上传文档">
        <i class="fas fa-paperclip"></i>
      </button>

      <input
        type="file"
        ref="fileInputRef"
        @change="onFileSelect"
        accept=".pdf,.doc,.docx,.xls,.xlsx,.html,.htm"
        style="display: none"
      />

      <textarea
        v-model="chatStore.userInput"
        @keydown="handleKeyDown"
        @compositionstart="handleCompositionStart"
        @compositionend="handleCompositionEnd"
        @input="autoResize"
        placeholder="和喵喵说点什么吧... (Shift+Enter 换行)"
        rows="1"
        ref="textareaRef"
      ></textarea>

      <button
        v-if="chatStore.isLoading"
        @click="chatStore.handleStop"
        class="send-btn stop-btn"
        title="终止回答"
      >
        <i class="fas fa-stop"></i>
      </button>

      <button
        v-else
        @click="onSend"
        class="send-btn"
        title="发送"
      >
        <i class="fas fa-paper-plane"></i>
      </button>
    </div>
    <div class="footer-text">AI 生成的内容可能包含错误，请仔细甄别。</div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, nextTick } from 'vue';
import { useChatStore } from '@/stores/chat';
import api from '@/utils/api';

const chatStore = useChatStore();
const textareaRef = ref<HTMLTextAreaElement | null>(null);
const fileInputRef = ref<HTMLInputElement | null>(null);
const isComposing = ref(false);

const uploadState = reactive<{
  file: File | null;
  status: 'idle' | 'uploading' | 'processing' | 'done' | 'error';
  message: string;
  uploadPercent: number;
  processPercent: number;
  jobId: string;
  pollTimer: any;
}>({
  file: null,
  status: 'idle',
  message: '',
  uploadPercent: 0,
  processPercent: 0,
  jobId: '',
  pollTimer: null,
});

const getFileIcon = (name: string) => {
  const lower = name.toLowerCase();
  if (lower.endsWith('.pdf')) return 'fas fa-file-pdf';
  if (lower.endsWith('.docx') || lower.endsWith('.doc')) return 'fas fa-file-word';
  if (lower.endsWith('.xlsx') || lower.endsWith('.xls')) return 'fas fa-file-excel';
  return 'fas fa-file';
};

const triggerFileSelect = () => {
  fileInputRef.value?.click();
};

const onFileSelect = (event: Event) => {
  const files = (event.target as HTMLInputElement).files;
  if (files && files.length > 0) {
    stopPolling();
    uploadState.file = files[0];
    uploadState.status = 'idle';
    uploadState.message = '';
    uploadState.uploadPercent = 0;
    uploadState.processPercent = 0;
    uploadState.jobId = '';
  }
};

const cancelUpload = () => {
  stopPolling();
  uploadState.file = null;
  uploadState.status = 'idle';
  uploadState.message = '';
  uploadState.uploadPercent = 0;
  uploadState.processPercent = 0;
  uploadState.jobId = '';
  if (fileInputRef.value) fileInputRef.value.value = '';
};

const stopPolling = () => {
  if (uploadState.pollTimer) {
    clearInterval(uploadState.pollTimer);
    uploadState.pollTimer = null;
  }
};

const startUpload = async () => {
  if (!uploadState.file) return;

  uploadState.status = 'uploading';
  uploadState.message = '正在上传...';
  uploadState.uploadPercent = 0;

  const formData = new FormData();
  formData.append('file', uploadState.file);

  try {
    const response = await api.post('/documents/upload/async', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
      onUploadProgress: (progressEvent: any) => {
        if (!progressEvent.total) return;
        const pct = Math.round((progressEvent.loaded / progressEvent.total) * 100);
        uploadState.uploadPercent = pct;
        uploadState.message = `正在上传... ${pct}%`;
      },
    });

    const data = response.data;
    uploadState.status = 'processing';
    uploadState.uploadPercent = 100;
    uploadState.processPercent = 0;
    uploadState.message = '正在解析与向量化入库...';
    uploadState.jobId = data.job_id;

    // Poll job status
    const poll = async () => {
      try {
        const jobResp = await api.get(`/documents/upload/jobs/${encodeURIComponent(uploadState.jobId)}`);
        const job = jobResp.data;

        // Calculate overall process progress from steps
        if (job.steps && job.steps.length) {
          const totalPct = job.steps.reduce((sum: number, s: any) => sum + (s.percent || 0), 0);
          uploadState.processPercent = Math.round(totalPct / job.steps.length);
        }
        uploadState.message = job.message || '正在处理...';

        if (job.status === 'completed') {
          stopPolling();
          uploadState.status = 'done';
          uploadState.processPercent = 100;
          uploadState.message = `文档 "${uploadState.file?.name}" 已就绪`;
          // Add system message to chat
          chatStore.messages.push({
            text: `📄 文档 **${uploadState.file?.name}** 已上传并完成向量化，共 ${job.total_chunks || 'N'} 个分块。现在可以针对该文档提问了！`,
            isUser: false,
          });
          // Auto clear after 5s
          setTimeout(() => {
            if (uploadState.status === 'done') cancelUpload();
          }, 5000);
        } else if (job.status === 'failed') {
          stopPolling();
          uploadState.status = 'error';
          uploadState.message = `处理失败: ${job.error || '未知错误'}`;
        }
      } catch (e: any) {
        stopPolling();
        uploadState.status = 'error';
        uploadState.message = `查询进度失败: ${e.message}`;
      }
    };

    poll();
    uploadState.pollTimer = setInterval(poll, 1500);
  } catch (error: any) {
    uploadState.status = 'error';
    const errMsg = error.response?.data?.detail || error.message || '上传失败';
    uploadState.message = errMsg;
  }
};

const handleCompositionStart = () => {
  isComposing.value = true;
};

const handleCompositionEnd = () => {
  isComposing.value = false;
};

const handleKeyDown = (event: KeyboardEvent) => {
  if (event.key === 'Enter' && !event.shiftKey && !isComposing.value) {
    event.preventDefault();
    onSend();
  }
};

const autoResize = () => {
  if (textareaRef.value) {
    textareaRef.value.style.height = 'auto';
    textareaRef.value.style.height = textareaRef.value.scrollHeight + 'px';
  }
};

const resetTextareaHeight = () => {
  if (textareaRef.value) {
    textareaRef.value.style.height = 'auto';
  }
};

const onSend = async () => {
  const text = chatStore.userInput.trim();
  if (!text || chatStore.isLoading || isComposing.value) return;

  await chatStore.handleSend();

  await nextTick();
  resetTextareaHeight();
};
</script>