# Tauri-QRCode-Reader 討論紀錄

## 1. _parseQRCode() 解說

`_parseQRCode()` 是用來解析圖片檔案中的 QRCode 文字的函式。  
流程如下：

- 用 FileReader 讀取圖片檔案為 DataURL。
- DataURL 載入到 Image 元素。
- 圖片載入完成後，畫到 canvas。
- 用 canvas 取得像素資料（ImageData）。
- 用 jsQR 解析 QRCode，結果用 callback 回傳。

---

## 2. reader.onload 與 img.onload

- `reader.onload`：檔案讀取完成才觸發。
- `img.onload`：圖片載入完成才觸發，這時才能安全地畫到 canvas。
- canvas 用來取得像素資料，給 jsQR 解析。

---

## 3. 為何要用 canvas 畫一次？

- jsQR 只能解析像素資料（ImageData），不能直接用 img 元素。
- img 只負責顯示，不能直接存取像素。
- canvas 可以操作和分析圖片內容。

---

## 4. _qrcodeImageData() 功能

- 把 video 元素目前的影像畫到 canvas。
- 用 canvas 取得 RGBA 像素資料（ImageData）。
- 回傳給 jsQR 做 QRCode 解析。
- 與解析圖片檔案時流程相同，只是來源不同。

---

## 5. 掃描循環 _tick() 解說

- 取得影像資料（_qrcodeImageData）。
- 用 jsQR 解析 QRCode。
- 掃描到就停止循環，沒掃描到就繼續。
- 用 requestAnimationFrame(_tick) 形成循環，速度約等於螢幕更新率（60fps）。

---

## 6. animationFrameId 用途

- 記錄 requestAnimationFrame 回傳的 ID。
- 用於 stopScan 時取消掃描循環（cancelAnimationFrame）。
- 控制掃描循環的開關。

---

## 7. fakeEvent 說明

- fakeEvent 是手動建立的事件物件，模擬檔案上傳。
- 只要資料結構正確（有 target.files），就能正常運作。
- 常用於單元測試或特殊流程。

---

## 8. 開啟鏡頭的程式段落

- 在 `startScan()` 函式裡開啟鏡頭：

```typescript
stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
```

- 若有多鏡頭，可用 enumerateDevices 取得所有鏡頭，讓使用者選擇：

```typescript
const devices = await navigator.mediaDevices.enumerateDevices();
const cameras = devices.filter(d => d.kind === 'videoinput');
const stream = await navigator.mediaDevices.getUserMedia({
  video: { deviceId: { exact: cameras[0].deviceId } }
});
```

---

## 9. nextTick() 功能

- Vue 的 nextTick() 用來等 DOM 更新完成後再執行程式碼。
- 確保 video 元素已渲染，才能安全操作 srcObject。

---

## 10. nextTick() 就是「等一下」的功用

- 沒錯，就是等 Vue 把畫面更新好，確保要操作的元素已存在。

---

## 11. 支援雙鏡頭

- 只要用 enumerateDevices 取得所有攝影機，讓使用者選擇 deviceId，就能支援雙鏡頭或多鏡頭。

```typescript
const devices = await navigator.mediaDevices.enumerateDevices();
const cameras = devices.filter(d => d.kind === 'videoinput');
const selectedDeviceId = cameras[1].deviceId;
stream = await navigator.mediaDevices.getUserMedia({
  video: { deviceId: { exact: selectedDeviceId