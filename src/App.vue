<template>
  <header>
    <h1>推しふぉと！</h1>
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
        <EditorCanvas ref="editor" />
      </div>

      <div class="canvas-controls">
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
            </div>
          </ul>
        </div>
      </div>
    </div>
    </main>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
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

const handleResize = () => {
  // 必要な要素が揃っていない（背景がまだない等）場合は何もしない
  if (!editor.value || !canvasWrapperRef.value || !currentBgDataUrl.value || !currentBgHtmlImage.value) {
    return;
  }

  console.log('リサイズを検知');
  
  // 1. 新しいコンテナ幅を取得
  const containerWidth = canvasWrapperRef.value.clientWidth;
  
  // 2. EditorCanvasのリサイズ関数を、保持している画像データで呼び出す
  editor.value.resizeAndSetBackground(
    currentBgDataUrl.value, 
    currentBgHtmlImage.value, 
    containerWidth
  );
};

onMounted(() => {
  backgroundGallery.loadImages();
  oshiGallery.loadOshiImages();
  window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

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

const handleSaveImage = () => {
  if (!editor.value) {
    console.error('handleSaveImage: editor (ref) が見つかりません');
    return;
  }
  
  // ★ 保存する元画像のHTMLImageElementを取得
  const originalImage = currentBgHtmlImage.value;
  
  if (!originalImage) {
    console.error('handleSaveImage: 元の背景画像がありません');
    alert('先に背景画像をセットしてください');
    return;
  }

  try {
    const dataUrl = editor.value.exportOriginalSizeDataURL(originalImage);
    
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
