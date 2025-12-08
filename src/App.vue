<template>
  <header>
    <h1>推しふぉと！</h1>
    <button 
      @click="toggleMenu" 
      class="menu-button" 
      :class="{ 'is-open': isMenuOpen }" aria-label="メニュー開閉"
    >
      <span class="material-symbols-rounded icon-menu">menu</span>
      <span class="material-symbols-rounded icon-close">close</span>
    </button>
  </header>

  <main class="main-content">
    
    <div class="main-column-left">
      <div 
        class="canvas-wrapper" 
        ref="canvasWrapperRef" 
        :class="{ 'has-image': canvasHasImage }"
      >
        <div v-if="!canvasHasImage" class="canvas-placeholder">
          背景画像を読み込んでください
        </div>
        <video ref="videoElementRef" v-show="isLivePreviewActive" class="live-video-bg" autoplay playsinline muted></video>
        <EditorCanvas ref="editor" @selection:changed="handleSelectionChange" />

        <!-- 推し画像選択時に表示されるフローティングコントロール -->
        <Transition name="fade-up">
          <div v-if="isOshiSelected" class="floating-controls">
            <div class="preset-grid">
              <button @click="positionOshi('topLeft')" title="左上"><span class="material-symbols-rounded">north_west</span></button>
              <button @click="positionOshi('topCenter')" title="上中央"><span class="material-symbols-rounded">north</span></button>
              <button @click="positionOshi('topRight')" title="右上"><span class="material-symbols-rounded">north_east</span></button>
              <button @click="positionOshi('centerLeft')" title="左中央"><span class="material-symbols-rounded">west</span></button>
              <button @click="positionOshi('center')" title="中央"><span class="material-symbols-rounded">center_focus_strong</span></button>
              <button @click="positionOshi('centerRight')" title="右中央"><span class="material-symbols-rounded">east</span></button>
              <button @click="positionOshi('bottomLeft')" title="左下"><span class="material-symbols-rounded">south_west</span></button>
              <button @click="positionOshi('bottomCenter')" title="下中央"><span class="material-symbols-rounded">south</span></button>
              <button @click="positionOshi('bottomRight')" title="右下"><span class="material-symbols-rounded">south_east</span></button>
            </div>
          </div>
        </Transition>
      </div>

      <div class="canvas-controls">
        <div class="control-group main-actions">
          <button @click="deleteSelectedOshi" class="btn btn-del-selected" title="選択した推しを削除">
            <span class="material-symbols-rounded">delete</span>
            <span class="button-text">削除</span>
          </button>
          <button @click="handleSaveImage" class="btn btn-save" title="画像を保存">
            <span class="material-symbols-rounded">download</span>
            <span class="button-text">保存</span>
          </button>
        </div>
      </div>
    </div>
    <div class="main-column-right">
      <div class="gallery oshi-gallery">
        <div class="gallery-header">
          <div class="gallery-header-info">
            <h3>推し画像</h3> <p>(保存数: {{ oshiGallery.oshiList.value.length }})</p>
          </div>
        </div>
        <div class="gallery-container">
          <ul>
            <li v-for="image in oshiGallery.oshiList.value" :key="image.id">
              <img 
                :src="createBlobUrl(image.data)" 
                class="thumbnail"
                @click="addOshiToCanvas(image.data)"
              />
              <button class="delete-btn" @click="oshiGallery.deleteOshiImage(image.id)" title="推し画像を削除">
                <span class="material-symbols-rounded">close</span>
              </button>
            </li>
              <div class="gallery-input">
              <label for="oshi-upload" class="btn" title="推し画像を追加">
                <span class="material-symbols-rounded">add_photo_alternate</span>追加
              </label>
              <input 
                id="oshi-upload"
                class="file-input"
                type="file" 
                accept="image/png, image/gif" 
                @change="onOshiFileChange"
              />
            </div>
          </ul>
        </div>
      </div>

      <div class="gallery">
        <div class="gallery-header">
          <div class="gallery-header-info">
            <h3>背景画像</h3> <p>(保存数: {{ backgroundGallery.galleryList.value.length }})</p>
          </div>
        </div>
        <div class="gallery-container">
          <ul>
            <li v-for="image in backgroundGallery.galleryList.value" :key="image.id"> 
              <img 
                :src="createBlobUrl(image.data)" 
                class="thumbnail"
                @click="setBackground(image.data)" 
              />
              <button class="delete-btn" @click="backgroundGallery.deleteImage(image.id)" title="背景画像を削除">
                <span class="material-symbols-rounded">close</span>
              </button>
            </li>
            <div class="gallery-input">
              <label for="bg-upload" class="btn" title="背景画像を追加">
                <span class="material-symbols-rounded">add_photo_alternate</span>追加
              </label>
              <input 
                id="bg-upload"
                class="file-input"
                type="file" 
                accept="image/*" 
                @change="onBackgroundFileChange"
              />
              <button @click="toggleLivePreview" class="btn" :class="{ 'is-active': isLivePreviewActive }" title="ライブプレビューを開始/停止">
                <span class="material-symbols-rounded">{{ isLivePreviewActive ? 'videocam_off' : 'photo_camera' }}</span>
                {{ isLivePreviewActive ? 'ライブ停止' : 'ライブ' }}
              </button>
            </div>
          </ul>
        </div>
      </div>
    </div>
  </main>

  <Transition name="fade">
    <div
      v-if="isMenuOpen"
      class="sidebar-overlay"
      @click="toggleMenu"
    ></div>
  </Transition>

  <Transition name="slide">
    <nav v-if="isMenuOpen" class="sidebar-menu">
      <h2>メニュー</h2>
      <ul>
        <li>
          <a href="#">設定</a>
        </li>
        <li>
          <a href="#">利用規約</a>
        </li>
      </ul>
    </nav>
  </Transition>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import EditorCanvas from './components/EditorCanvas.vue';
import { useBackgroundStore } from './composables/useBackgroundStore.ts';
import { useOshiGallery } from './composables/useOshiStore.ts';

const backgroundGallery = useBackgroundStore();
const oshiGallery = useOshiGallery();
const currentBgDataUrl = ref(null);
const currentBgHtmlImage = ref(null);
const editor = ref(null);
const canvasWrapperRef = ref(null);
const canvasHasImage = ref(false);
const isMenuOpen = ref(false);
const isOshiSelected = ref(false);

// カメラ関連のstate
const isLivePreviewActive = ref(false);
const liveStream = ref(null);
const videoElementRef = ref(null);

const toggleMenu = () => {
  isMenuOpen.value = !isMenuOpen.value;
};

const handleSelectionChange = (isSelected) => {
  isOshiSelected.value = isSelected;
}

const handleResize = () => {
  if (!editor.value || !canvasWrapperRef.value) return;

  const containerWidth = canvasWrapperRef.value.clientWidth;

  if (isLivePreviewActive.value && videoElementRef.value && videoElementRef.value.readyState > 0) {
    // ライブプレビュー中のリサイズ
    const video = videoElementRef.value;
    const videoRatio = video.videoHeight / video.videoWidth;
    editor.value.resizeCanvas(containerWidth, containerWidth * videoRatio);
    return;
  }
  
  if (currentBgDataUrl.value && currentBgHtmlImage.value) {
    // 通常の背景画像がある場合のリサイズ
    editor.value.resizeAndSetBackground(currentBgDataUrl.value, currentBgHtmlImage.value, containerWidth);
  } else {
    return;
  }
};

onMounted(() => {
  backgroundGallery.loadImages();
  oshiGallery.loadOshiImages();
  window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
  stopLivePreview(); // コンポーネント破棄時にカメラを閉じる
});

const stopLivePreview = () => {
  if (liveStream.value) {
    liveStream.value.getTracks().forEach(track => track.stop());
  }
  liveStream.value = null;
  isLivePreviewActive.value = false;
  if (editor.value) {
    editor.value.clearBackground();
    canvasHasImage.value = false;
    currentBgDataUrl.value = null;
    currentBgHtmlImage.value = null;
  }
};

const toggleLivePreview = async () => {
  if (isLivePreviewActive.value) {
    stopLivePreview();
    return;
  }

  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    alert('お使いのブラウザはカメラ機能に対応していません。');
    return;
  }

  try {
    const stream = await navigator.mediaDevices.getUserMedia({ 
      video: { facingMode: 'environment' }, 
      audio: false 
    });
    liveStream.value = stream;
    isLivePreviewActive.value = true;
    
    await nextTick();

    if (videoElementRef.value) {
      videoElementRef.value.srcObject = stream;
      videoElementRef.value.onloadedmetadata = () => {
        if (!editor.value || !canvasWrapperRef.value) return;
        const video = videoElementRef.value;
        const containerWidth = canvasWrapperRef.value.clientWidth;
        const videoRatio = video.videoHeight / video.videoWidth;
        
        editor.value.resizeCanvas(containerWidth, containerWidth * videoRatio);
        editor.value.clearBackground();
        canvasHasImage.value = true;
        currentBgDataUrl.value = null;
        currentBgHtmlImage.value = null;
      };
    }
  } catch (err) {
    console.error('カメラの起動に失敗しました:', err);
    let message = 'カメラの起動に失敗しました。';
    if (err.name === 'NotAllowedError') message += '\nカメラへのアクセスが許可されていません。';
    else if (err.name === 'NotFoundError') message += '\n利用可能なカメラが見つかりませんでした。';
    alert(message);
    isLivePreviewActive.value = false;
  }
};

const createBlobUrl = (blob) => {
  if (!blob) return '';
  return URL.createObjectURL(blob);
};

const convertBlobToDataURL = (blob) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result);
    reader.onerror = (error) => reject(error);
    reader.readAsDataURL(blob);
  });
};

const loadHtmlImage = (dataUrl) => {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve(img);
    img.onerror = (err) => reject(err);
    img.src = dataUrl;
  });
};

const setBackground = async (imageBlob) => {
  if (!editor.value || !canvasWrapperRef.value) {
    console.error('setBackground: editor または wrapper が見つかりません');
    return;
  }

  // ライブプレビューがアクティブなら停止する
  if (isLivePreviewActive.value) {
    stopLivePreview();
  }
  
  try {
    const dataUrl = await convertBlobToDataURL(imageBlob);
    const imgElement = await loadHtmlImage(dataUrl);
    const containerWidth = canvasWrapperRef.value.clientWidth;
    currentBgDataUrl.value = dataUrl;
    currentBgHtmlImage.value = imgElement;
    editor.value.resizeAndSetBackground(dataUrl, imgElement, containerWidth);
    canvasHasImage.value = true;
  } catch (error) {
    console.error('背景画像のセットに失敗:', error);
  }
};

const addOshiToCanvas = async (imageBlob) => {
  if (!editor.value) return;
  try {
    const dataUrl = await convertBlobToDataURL(imageBlob);
    editor.value.addOshiImage(dataUrl); 
  } catch (error) {
    console.error('推し画像の変換に失敗:', error);
  }
};

const deleteSelectedOshi = () => {
  if (!editor.value) return;
  editor.value.deleteActiveObject();
};

const positionOshi = (preset) => {
  if (editor.value) {
    editor.value.positionActiveObject(preset);
  }
};

const handleSaveImage = async () => {
  if (!editor.value) {
    console.error('handleSaveImage: editor (ref) が見つかりません');
    return;
  }

  try {
    let dataUrl = null;

    if (isLivePreviewActive.value) {
      // ライブプレビュー中の保存
      if (!videoElementRef.value) {
        alert('ビデオ要素が見つかりません。');
        return;
      }
      dataUrl = await editor.value.exportWithLiveBackground(videoElementRef.value);

    } else {
      // 通常の背景画像での保存
      const originalImage = currentBgHtmlImage.value;
      if (!originalImage) {
        alert('先に背景画像をセットしてください');
        return;
      }
      dataUrl = editor.value.exportOriginalSizeDataURL(originalImage);
    }
    
    if (!dataUrl) {
      console.error('画像の書き出しに失敗しました (dataUrl is null)');
      alert('画像の書き出しに失敗しました。');
      return;
    }
    
    // 1. 現在時刻からDateオブジェクトを生成
    const now = new Date();

    // 2. 各パーツを取得し、padStart(2, '0') で2桁（0埋め）にする
    const year = now.getFullYear().toString().slice(-2); // 年(下2桁)
    const month = (now.getMonth() + 1).toString().padStart(2, '0'); // 月 (0-11なので+1)
    const day = now.getDate().toString().padStart(2, '0'); // 日
    const hours = now.getHours().toString().padStart(2, '0'); // 時
    const minutes = now.getMinutes().toString().padStart(2, '0'); // 分
    const seconds = now.getSeconds().toString().padStart(2, '0'); // 秒

    // 3. ご希望のフォーマットに組み立て
    const fileName = `osph-${year}${month}${day}-${hours}${minutes}${seconds}.png`;

    // 4. リンクを作成してダウンロード
    const link = document.createElement('a');
    link.download = fileName;
    link.href = dataUrl;
    link.click();
    
  } catch (error) {
    console.error('handleSaveImage でエラーが発生しました:', error);
    alert('画像の保存中にエラーが発生しました。');
  }
};

const onBackgroundFileChange = (event) => {
  const file = event.target.files[0];
  if (!file) return;
  backgroundGallery.addImage(file);
  event.target.value = null;
};

const onOshiFileChange = (event) => {
  const file = event.target.files[0];
  if (!file) return;
  oshiGallery.addOshiImage(file);
  event.target.value = null;
};
</script>

<style>
.gallery-input {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.canvas-wrapper {
  position: relative;
}

.live-video-bg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
  z-index: 0; /* キャンバスより後ろに配置 */
}

/* EditorCanvasコンポーネントのラッパーに適用 */
:deep(.canvas-container) {
  z-index: 1;
}

.btn.is-active {
  background-color: #e91e63;
  color: white;
}

.fade-up-enter-active,
.fade-up-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.fade-up-enter-from,
.fade-up-leave-to {
  opacity: 0;
  transform: translateY(10px);
}

.canvas-controls {
  display: flex;
  justify-content: flex-start; /* 左寄せに変更 */
  align-items: center;
  flex-wrap: wrap;
  gap: 10px; /* ギャップを少し詰める */
  padding: 10px;
  background-color: #f0f0f0;
  border-radius: 8px;
}

.floating-controls {
  position: absolute;
  bottom: 15px;
  left: 50%;
  transform: translateX(-50%);
  background-color: rgba(255, 255, 255, 0.85);
  padding: 8px;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
  z-index: 10;
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
}

.control-group {
  display: flex;
  gap: 10px;
  align-items: center;
}

.preset-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 4px;
}

.preset-grid button {
  background-color: rgba(255, 255, 255, 0.5);
  border: none;
  padding: 6px;
  border-radius: 8px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #333;
  transition: background-color 0.2s ease;
}

.preset-grid button:hover {
  background-color: rgba(0, 0, 0, 0.1);
}
</style>
