<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted, nextTick } from 'vue';
import { listen } from "@tauri-apps/api/event";
import { readFile, writeFile } from "@tauri-apps/plugin-fs";
import { save } from '@tauri-apps/plugin-dialog';
import QRCode from 'qrcode';
import jsQR from 'jsqr';

const text = ref('https://tauri.app');
const originalText = ref(text.value);
const isError = ref(false);
const qrCanvas = ref<HTMLCanvasElement | null>(null);
const textInput = ref<HTMLInputElement | null>(null);
const video = ref<HTMLVideoElement | null>(null);
const isScanning = ref(false);

let stream: MediaStream | null = null;
let animationFrameId: number | null = null;

// MARK: - public function
/**
 * 開始掃描
 */
async function startScan() {

    if (!navigator.mediaDevices) { return; }
    if (!navigator.mediaDevices.getUserMedia) { return; }

    try {

      stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
      isScanning.value = true;

      await nextTick();
      
      if (!video.value) { setErrorState('無法存取攝影機，請確認已授權。'); return; }
      
      video.value.srcObject = stream;
      video.value.play();
      _tick();

    } catch (error: Error | any) {
      setErrorState(error.message || '無法存取攝影機，請確認已授權。');
    }
}

/**
 * 停止掃描
 */
function stopScan() {
  
  isScanning.value = false;

  if (stream) { stream.getTracks().forEach(track => track.stop()); }
  if (animationFrameId) { cancelAnimationFrame(animationFrameId); }
}

/**
 * 設定錯誤訊息
 * @param message - 錯誤訊息
 */
function setErrorState(message: string) {

  originalText.value = text.value;
  isError.value = true;
  text.value = message;

  textInput.value?.blur();
}

// MARK: - async function
/**
 * 處理文件拖放事件 (上傳圖片)
 */
async function handleFileDragDrop() {

  listen('tauri://drag-drop', async (event: any) => {

    const filePath = event.payload.paths[0] as string;

    if (event?.payload?.paths.length > 0, filePath.length > 0) {
      
      const fileName = filePath.split('/').pop() || 'upload.png';
      const fileType = 'image/png'
      const file = await _fileMaker(filePath, fileName, fileType)
      const fakeEvent = { target: { files: [file] }} as unknown as Event;

      handleFileUpload(fakeEvent);
    }
  });
}

/**
 * 下載圖檔
 * @param defaultPath - 要下載的文件路徑
 * @returns `Promise<string | null>`
 */
async function downloadQRCode() {
  
  if (isScanning.value) { return; }
  if (!qrCanvas.value) { return; }

  const url = qrCanvas.value.toDataURL("image/png");
  const uint8Array = await _downloadFile(url)
  const filePath = await _savePNGImage('QR-Code.png')

  if (!filePath) return;
  await writeFile(filePath, uint8Array);
}

// MARK: - Event Handlers
/**
 * 處理錯誤訊息
 */
function handleFocus() {

  if (!isError.value) { return }
  
  isError.value = false;
  text.value = originalText.value;
}

/**
 * 處理上傳圖片功能 (解析QRCode文字)
 * @param event - Event
 */
function handleFileUpload(event: Event) {
  
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  
  if (isScanning.value) { stopScan(); }
  if (!file) return;
  if (!file.type.startsWith('image/')) { setErrorState('請上傳圖片檔案。'); target.value = ''; return; }

  handleFocus();

  _parseQRCode(file, (code) => {

    if (!code) { setErrorState('無法解析圖片中的 QRCode。'); target.value = ''; return; }

    text.value = code;
    textInput.value?.focus();
  });

  target.value = '';
}

// MARK: - private function
/**
 * 解析圖片上的QRCode文字
 * @param file - File
 * @param callback - 回調函式
 */
function _parseQRCode(file: File, callback: (result?: string) => void) {

  const reader = new FileReader();

  reader.onload = (event) => {

    const img = new Image();

    img.src = event.target?.result as string;
    img.onload = () => {

      const canvas = document.createElement('canvas');
      const context = canvas.getContext('2d');
      
      canvas.width = img.width;
      canvas.height = img.height;
      context?.drawImage(img, 0, 0, canvas.width, canvas.height);

      const imageData = context?.getImageData(0, 0, canvas.width, canvas.height);

      if (imageData) {
        const code = jsQR(imageData.data, imageData.width, imageData.height);
        callback(code?.data);
      }
    };
  };

  reader.readAsDataURL(file);
}

/**
 * 掃描循環
 */
function _tick() {

  const imageData = _qrcodeImageData();

  if (imageData) {
    
    const code = jsQR(imageData.data, imageData.width, imageData.height);
    
    if (code) {
      handleFocus();
      text.value = code.data;
      textInput.value?.focus();
      stopScan();
    }
  }

  if (isScanning.value) { animationFrameId = requestAnimationFrame(_tick); }
}

/**
 * 取得 QRCode 影像資料
 */
function _qrcodeImageData() {

  if (!video.value) { return; }
  if (!isScanning.value) { return; }
  if (video.value.readyState !== video.value.HAVE_ENOUGH_DATA) { return; }

  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');

  canvas.width = video.value.videoWidth;
  canvas.height = video.value.videoHeight;
  context?.drawImage(video.value, 0, 0, canvas.width, canvas.height);

  const imageData = context?.getImageData(0, 0, canvas.width, canvas.height);
  return imageData
}

/**
 * 文件路徑 => File
 * @param filePath - 要讀取的文件路徑
 * @param fileName - 要儲存的文件名稱
 * @param fileType - 要儲存的文件類型
 * @returns `Promise<File>`
 */
async function _fileMaker(filePath: string, fileName: string, fileType: string) {
  const fileData = await readFile(filePath);
  const file = new File([new Uint8Array(fileData)], fileName, { type: fileType });
  return file
}

/**
 * 下載檔案 => Uint8Array
 * @param url - 要下載的文件路徑
 * @returns `Promise<Uint8Array<ArrayBuffer>>`
 */
async function _downloadFile(url: string) {

  const res = await fetch(url);
  const blob = await res.blob();
  const arrayBuffer = await blob.arrayBuffer();
  const uint8Array = new Uint8Array(arrayBuffer);

  return uint8Array
}

/**
 * 儲存PNG圖檔
 * @param defaultPath - 要下載的文件路徑
 * @returns `Promise<string | null>`
 */
async function _savePNGImage(defaultPath: string) {

  const filePath = await save({
    defaultPath: defaultPath,
    filters: [{ name: "PNG Image", extensions: ["png"] }]
  });

  return filePath
}

// MARK: - 生命週期
onMounted(() => {
  handleFileDragDrop();
});

onUnmounted(() => {
  stopScan();
});

watch([text, qrCanvas, isError], async () => {
  
  if (isError.value) { return; }
  if (!text.value) { return; }
  if (!qrCanvas.value) { return; }

  QRCode.toCanvas(qrCanvas.value, text.value, { width: 360, margin: 3, color: { dark: '#333', light: '#FFFFFF' } }, (error) => {
    if (error) { setErrorState(error.message); }
  });
}, { immediate: true });

</script>

<template>
  <div class="app-container">
    <div class="top-section">
      <div class="input-group">
        <input type="text" ref="textInput" v-model="text" placeholder="輸入文字或網址以產生 QRCode" :class="['text-input', { 'error': isError }]" @focus="handleFocus"/>
      </div>
      <div class="button-group">
        <button @click="startScan" :disabled="isScanning" class="btn">
          <i class="icon-camera"></i> 開始掃描
        </button>
        <button @click="stopScan" :disabled="!isScanning" class="btn btn-danger">
          <i class="icon-stop"></i> 停止掃描
        </button>
        <label for="file-upload" class="btn btn-secondary">
          <i class="icon-upload"></i> 上傳圖片
        </label>
        <input id="file-upload" type="file" accept="image/*" @change="handleFileUpload" style="display: none;" />
      </div>
    </div>
    <div class="bottom-section">
      <div class="display-area">
        <div class="display-box">
          <div class="video-preview" v-if="isScanning">
            <video ref="video" autoplay muted playsinline></video>
            <div class="scan-overlay"></div>
          </div>
          <canvas v-else ref="qrCanvas" @click="downloadQRCode" style="cursor: pointer;" title="點擊下載 QRCode"></canvas>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
@import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css');

:root {
  --primary-color: #4F46E5;
  --secondary-color: #6B7280;
  --danger-color: #EF4444;
  --success-color: #059669;
  --background-color: #F9FAFB;
  --card-background: #FFFFFF;
  --text-color: #1F2937;
  --light-text-color: #FFFFFF;
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: var(--background-color);
  color: var(--text-color);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}

.top-section {
  flex: 0.2;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 2rem;
  border-bottom: 1px solid #E5E7EB;
}

.input-group {
  width: 100%;
  max-width: 500px;
  margin-bottom: 1.5rem;
}

.text-input {
  width: 100%;
  padding: 0.75rem 1rem;
  font-size: 1rem;
  border-radius: 0.5rem;
  border: 1px solid #D1D5DB;
  transition: all 0.2s;
}

.text-input:focus {
  outline: none;
  border-color: var(--success-color);
}

.text-input.error {
  color: #EF4444;
  border-color: #EF4444;
}

.text-input.error:focus {
  color: var(--text-color);
  border-color: var(--success-color);
}

.button-group {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  justify-content: center;
}

.btn {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  font-weight: 500;
  border-radius: 0.5rem;
  border: none;
  cursor: pointer;
  transition: all 0.2s;
  background-color: var(--primary-color);
  color: var(--light-text-color);
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.btn:disabled {
  background-color: #9CA3AF;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.btn-danger {
  background-color: var(--danger-color);
}

.btn-secondary {
  background-color: var(--secondary-color);
}

.bottom-section {
  flex: 0.7;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 2rem;
}

.display-area {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.display-box {
  width: 100%;
  max-width: 360px;
  aspect-ratio: 1 / 1;
  background-color: var(--card-background);
  border-radius: 1rem;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

.video-preview {
  position: relative;
  width: 100%;
  height: 100%;
}

.video-preview video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(-1);
}

.scan-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 60%;
  height: 60%;
  border: 3px solid rgba(255, 255, 255, 0.8);
  box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.5);
}

/* Font Awesome Icons */
.icon-camera:before { font-family: FontAwesome; content: '\f030'; }
.icon-stop:before { font-family: FontAwesome; content: '\f04d'; }
.icon-upload:before { font-family: FontAwesome; content: '\f093'; }
</style>
