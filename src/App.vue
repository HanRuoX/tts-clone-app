<template>
  <div class="app-container">
    <!-- 导航栏 -->
    <van-nav-bar title="语音克隆与合成" />

    <!-- 语音克隆部分 -->
    <div class="clone-section">
      <h3>语音克隆</h3>

      <!-- 输入音色名称 -->
      <van-field 
        v-model="voiceName" 
        label="音色名称" 
        placeholder="请输入音色名称" 
      />
      <!-- 输入提示文本 -->
      <van-field 
        v-model="promptText" 
        label="提示文本" 
        placeholder="请输入音频对应的文字" 
      />

      <!-- 上传音频文件 -->
      <van-uploader
        accept="audio/*"
        :after-read="handleAudioUpload"
        upload-icon="music-o"
        upload-text="上传音频文件"
        class="mt-3"
      />

      <!-- 开始/停止录音按钮 -->
      <van-button 
        type="primary" 
        block 
        class="mt-3" 
        @click="toggleRecording"
      >
        {{ isRecording ? '停止录音' : '开始录音' }}
      </van-button>

      <!-- 录音回放 -->
      <audio 
        v-if="audioUrl" 
        :src="audioUrl" 
        controls 
        class="mt-3"
      ></audio>

      <!-- 提交克隆按钮：有音频才显示 -->
      <van-button 
        v-if="audioBlob" 
        type="primary" 
        block 
        class="mt-3" 
        @click="submitClone"
      >
        开始克隆
      </van-button>
    </div>

    <!-- 语音合成部分 -->
    <div class="synthesis-section">
      <h3>语音合成</h3>

      <!-- 输入待合成文本 -->
      <van-field
        v-model="inputText"
        rows="3"
        autosize
        type="textarea"
        placeholder="请输入要合成的文本"
      />

      <!-- 选择发音人 -->
      <van-field
        readonly
        clickable
        :model-value="selectedSpeakerText"
        label="选择发音人"
        placeholder="请选择发音人"
        @click="fetchSpeakerList(true)"
      />
      <van-popup 
        v-model:show="showPicker" 
        position="bottom"
      >
        <van-picker
          :columns="speakerOptions"
          @confirm="onPickerConfirm"
          @cancel="showPicker = false"
        />
      </van-popup>

      <!-- 语音合成按钮 -->
      <van-button 
        type="success" 
        block 
        class="mt-3" 
        @click="synthesizeText"
      >
        合成语音
      </van-button>

      <!-- 合成后的语音回放 -->
      <audio 
        v-if="synthesizedAudioUrl" 
        :src="synthesizedAudioUrl" 
        controls 
        class="mt-3"
      ></audio>
    </div>

    <!-- 加载中的遮罩层与自定义样式的加载指示器 -->
    <van-overlay :show="showLoading" z-index="2000" class="custom-overlay">
      <div class="loading-wrapper">
        <van-loading
          type="circular"
          size="40"
          class="loading-icon"
        />
        <div class="loading-text">加载中，请稍候...</div>
      </div>
    </van-overlay>
  </div>
</template>

<script setup>
/**
 * 说明：
 *  Vue 3 <script setup> 语法 + Vant
 */

import { ref, onMounted, computed } from 'vue';
import {
  NavBar,
  Uploader,
  Button,
  Field,
  Popup,
  Picker,
  showToast,
  Overlay,
  Loading,
} from 'vant';
import axios from 'axios';

/* -----------------------------
   录音 / 音频相关的状态管理
----------------------------- */
const isRecording = ref(false);       // 是否在录音中
let audioContext;                     // AudioContext 对象
let mediaStream;                      // 媒体流
let scriptProcessor;                  // ScriptProcessor 用于处理实时音频数据
let audioBuffer = [];                 // 存放录音的 PCM 数据
let sampleRate = 16000;               // 录音采样率，需与服务器保持一致

/* -----------------------------
   音频克隆需要的字段
----------------------------- */
const voiceName = ref('');           // 用户输入的音色名称
const promptText = ref('');          // 用户输入的提示文本
const audioUrl = ref('');            // 回放音频的链接
const audioBlob = ref(null);         // 上传或录音生成的音频文件数据

/* -----------------------------
   语音合成需要的字段
----------------------------- */
const inputText = ref('');           // 待合成的文本
const selectedSpeaker = ref('');      // 选定的发音人 ID
const speakerOptions = ref([]);       // 发音人选项列表 { text, value }
const synthesizedAudioUrl = ref('');  // 合成后的音频链接

/* -----------------------------
   交互相关
----------------------------- */
const showPicker = ref(false);       // 是否显示发音人选择器
const showLoading = ref(false);      // 是否显示“加载中”遮罩

/**
 * 计算属性：根据当前选定的发音人 ID，
 * 从 speakerOptions 找到相应的文字显示
 */
const selectedSpeakerText = computed(() => {
  const found = speakerOptions.value.find(
    (item) => item.value === selectedSpeaker.value
  );
  return found ? found.text : '请选择发音人';
});

/* -----------------------------
   设备 ID 及 axios 默认配置
----------------------------- */
const deviceId = 'abcd';                       // 示例：写死为 abcd
axios.defaults.baseURL = 'http://localhost:9800/api/v1';

/* -----------------------------
   获取发音人列表
----------------------------- */
const fetchSpeakerList = async (isShow) => {
  // 是否在获取列表后显示选择器
  showPicker.value = isShow;
  // 显示加载
  showLoading.value = true;

  try {
    const response = await axios.get(`/voices/${deviceId}`);
    // 拼接 general 与 user 的发音人
    speakerOptions.value = response.data.general.concat(response.data.user).map((v) => ({
      text: v.voice_name,
      value: v.voice_id,
    }));
    // 默认选第一个
    if (speakerOptions.value.length > 0) {
      selectedSpeaker.value = speakerOptions.value[0].value;
    }
  } catch (error) {
    showToast('获取发音人列表失败');
  } finally {
    showLoading.value = false;
  }
};

/* -----------------------------
   onMounted：初始化时先获取一次发音人列表
----------------------------- */
onMounted(async () => {
  await fetchSpeakerList(false);
});

/* -----------------------------
   上传本地音频文件回调
----------------------------- */
const handleAudioUpload = (file) => {
  audioBlob.value = file.file;
  audioUrl.value = URL.createObjectURL(file.file);
};

/* -----------------------------
   开始 / 停止录音
----------------------------- */
const toggleRecording = async () => {
  if (isRecording.value) {
    // 如果正在录音，则停止
    stopRecording();
    isRecording.value = false;
  } else {
    // 否则开始录音
    await startRecording();
    isRecording.value = true;
  }
};

/* -----------------------------
   开始录音
----------------------------- */
const startRecording = async () => {
  audioContext = new (window.AudioContext || window.webkitAudioContext)({
    sampleRate,
  });

  mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true });
  const source = audioContext.createMediaStreamSource(mediaStream);
  scriptProcessor = audioContext.createScriptProcessor(4096, 1, 1);

  scriptProcessor.onaudioprocess = (event) => {
    const inputBuffer = event.inputBuffer.getChannelData(0);
    audioBuffer.push(new Float32Array(inputBuffer));
  };

  source.connect(scriptProcessor);
  scriptProcessor.connect(audioContext.destination);
};

/* -----------------------------
   停止录音
----------------------------- */
const stopRecording = () => {
  scriptProcessor.disconnect();
  mediaStream.getTracks().forEach((track) => track.stop());

  // 转成 WAV
  const wavBlob = encodeWAV(audioBuffer, sampleRate);
  const randomFileName = `record_${Date.now()}_${Math.floor(Math.random() * 10000)}.wav`;
  audioBlob.value = new File([wavBlob], randomFileName, { type: 'audio/wav' });
  audioUrl.value = URL.createObjectURL(audioBlob.value);

  audioBuffer = [];
};

/* -----------------------------
   提交克隆
----------------------------- */
const submitClone = async () => {
  if (!voiceName.value || !promptText.value || !audioBlob.value) {
    showToast('请完善音色名称、提示文本及音频文件');
    return;
  }
  showLoading.value = true;

  const formData = new FormData();
  formData.append('audio_file', audioBlob.value);

  try {
    await axios.post(
      `/voices/clone?device_id=${deviceId}&voice_name=${voiceName.value}&prompt=${promptText.value}`,
      formData,
      {
        headers: { 'Content-Type': 'multipart/form-data' },
      }
    );
    showToast('音色克隆成功');
  } catch (error) {
    showToast('音色克隆失败');
  } finally {
    showLoading.value = false;
  }
};

/* -----------------------------
   PCM 转 WAV
----------------------------- */
const encodeWAV = (samples, sampleRate) => {
  const numChannels = 1;
  const bytesPerSample = 2;
  const length = samples.reduce((acc, val) => acc + val.length, 0);
  const wavBuffer = new ArrayBuffer(44 + length * bytesPerSample);
  const view = new DataView(wavBuffer);

  let offset = 0;
  // RIFF 头
  writeString(view, offset, 'RIFF'); offset += 4;
  view.setUint32(offset, 36 + length * bytesPerSample, true); offset += 4;
  writeString(view, offset, 'WAVE'); offset += 4;

  // fmt 子块
  writeString(view, offset, 'fmt '); offset += 4;
  view.setUint32(offset, 16, true); offset += 4;  
  view.setUint16(offset, 1, true); offset += 2;  // PCM
  view.setUint16(offset, numChannels, true); offset += 2;
  view.setUint32(offset, sampleRate, true); offset += 4;
  view.setUint32(offset, sampleRate * numChannels * bytesPerSample, true); offset += 4;
  view.setUint16(offset, numChannels * bytesPerSample, true); offset += 2;
  view.setUint16(offset, bytesPerSample * 8, true); offset += 2;

  // data 子块
  writeString(view, offset, 'data'); offset += 4;
  view.setUint32(offset, length * bytesPerSample, true); offset += 4;

  let pos = offset;
  samples.forEach((sample) => {
    for (let i = 0; i < sample.length; i++, pos += bytesPerSample) {
      const s = Math.max(-1, Math.min(1, sample[i]));
      view.setInt16(pos, s < 0 ? s * 0x8000 : s * 0x7fff, true);
    }
  });

  return new Blob([view], { type: 'audio/wav' });
};

/* -----------------------------
   写字符串的工具函数
----------------------------- */
const writeString = (view, offset, string) => {
  for (let i = 0; i < string.length; i++) {
    view.setUint8(offset + i, string.charCodeAt(i));
  }
};

/* -----------------------------
   文本合成
----------------------------- */
const synthesizeText = async () => {
  if (!inputText.value) {
    showToast('请输入文本内容');
    return;
  }
  showLoading.value = true;

  try {
    const response = await axios.post(
      '/voices/synthesize',
      {
        device_id: deviceId,
        text: inputText.value,
        voice_id: selectedSpeaker.value,
      },
      { responseType: 'blob' }
    );

    synthesizedAudioUrl.value = URL.createObjectURL(response.data);
    // 播放合成音频
    const audio = new Audio(synthesizedAudioUrl.value);
    audio.play();

    showToast('语音合成成功');
  } catch (error) {
    showToast('语音合成失败');
  } finally {
    showLoading.value = false;
  }
};

/* -----------------------------
   选择器确认事件
----------------------------- */
const onPickerConfirm = ({ selectedOptions }) => {
  selectedSpeaker.value = selectedOptions[0].value;
  showPicker.value = false;
};
</script>

<style scoped>
.app-container {
  padding: 20px;
  padding-bottom: 80px;
  overflow-y: auto;
  height: 100vh;
}

.clone-section,
.synthesis-section {
  margin-bottom: 20px;
  padding: 12px;
  border-radius: 8px;
  background-color: #f5f7fa;
}

.mt-3 {
  margin-top: 12px;
}

/* 自定义加载遮罩层外观 */
.custom-overlay {
  background-color: rgba(0, 0, 0, 0.5) !important; /* 半透明深色背景 */
}

/* 加载容器：居中放置 */
.loading-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
}

/* 调整loading本体外观 */
.loading-icon {
  /* 你可以根据需求自行定义颜色或大小 */
}

/* 提示文字 */
.loading-text {
  margin-top: 8px;
  font-size: 16px;
  color: #ffffff;
  text-align: center;
}
</style>