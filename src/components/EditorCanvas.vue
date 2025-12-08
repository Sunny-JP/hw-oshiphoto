<template>
  <div>
    <canvas ref="canvasElement"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as fabric from 'fabric';

const emit = defineEmits(['selection:changed']);

const canvasElement = ref(null);
let fabricCanvas = null;

// Fabric.jsのコントロールの外観をカスタマイズ
const oshiPink = '#e91e63';
const controlStrokeColor = '#ffffff'; // ハンドルの枠線の色

fabric.Object.prototype.borderColor = oshiPink;
fabric.Object.prototype.cornerColor = oshiPink;
fabric.Object.prototype.cornerSize = 16; // 見やすくするためにサイズを大きく
fabric.Object.prototype.transparentCorners = false;
fabric.Object.prototype.borderScaleFactor = 3; // 枠線を太く
fabric.Object.prototype.cornerStyle = 'circle';
fabric.Object.prototype.cornerStrokeColor = controlStrokeColor; // ハンドルに白い枠線を追加
fabric.Object.prototype.borderOpacityWhenMoving = 1; // 移動中も枠線を不透明に

onMounted(() => {
  if (canvasElement.value) {
    fabricCanvas = new fabric.Canvas(canvasElement.value, {
      width: 1,
      height: 1,
      backgroundColor: 'transparent',
      selectionColor: 'rgba(233, 30, 99, 0.2)' // 選択範囲の背景色
    });
    // オブジェクトの選択状態が変化したときにイベントを発火
    fabricCanvas.on('selection:created', () => emit('selection:changed', true));
    fabricCanvas.on('selection:updated', () => emit('selection:changed', true));
    fabricCanvas.on('selection:cleared', () => emit('selection:changed', false));
    console.log('Fabric.jsのCanvasを初期化しました');
  } else {
    console.error('onMounted: canvasElement が見つかりません');
  }
});

const addOshiImage = (imageUrl) => {
  if (!fabricCanvas) return;
  console.log('addOshiImage が呼ばれました');
  
  const imgElement = new Image();
  imgElement.onload = () => {
    console.log('推し画像の読み込み成功 (Image.onload)');
    const fabricImg = new fabric.Image(imgElement);
    
    // Canvasのサイズに対して、いい感じのサイズに調整
    // (例: Canvasの幅の 1/3 サイズにする)
    const scale = (fabricCanvas.width * 0.33) / imgElement.naturalWidth;
    
    fabricImg.set({
      left: 100,
      top: 100,
      scaleX: scale,
      scaleY: scale,
    });
    
    fabricCanvas.add(fabricImg);
    fabricCanvas.renderAll();
    console.log('「推し」画像を追加しました');
  };
  imgElement.onerror = (err) => { /* ... (省略) ... */ };
  imgElement.src = imageUrl;
};

const resizeAndSetBackground = (dataUrl, imgElement, containerWidth) => {
  if (!fabricCanvas) {
    console.error('resizeAndSetBackground: fabricCanvas が初期化されていません');
    return;
  }
  console.log('resizeAndSetBackground が呼ばれました');

  // 1. 新しいCanvasのサイズを計算
  const imgRatio = imgElement.naturalHeight / imgElement.naturalWidth;
  
  // 親コンテナの幅(containerWidth)をそのままCanvasの幅として使用します
  const newWidth = containerWidth;
  const newHeight = newWidth * imgRatio;

  // 2. Fabric.jsのCanvasにサイズをセット
  fabricCanvas.setDimensions({
    width: newWidth,
    height: newHeight
  });
  
  // 3. <img> から FabricImage を作成
  const fabricImg = new fabric.Image(imgElement);

  // 4. 背景としてセット
  fabricCanvas.backgroundColor = 'transparent'; // 以前の背景色をクリア
  fabricCanvas.backgroundImage = fabricImg;
  
  // 5. 画像をCanvasの幅にフィットさせる
  fabricImg.scaleToWidth(newWidth);

  // 6. 変更をCanvasに反映
  fabricCanvas.renderAll();

  console.log(`Canvasをリサイズしました: ${newWidth} x ${newHeight}`);
};

const resizeCanvas = (width, height) => {
  if (!fabricCanvas) return;
  fabricCanvas.setDimensions({ width, height });
  fabricCanvas.renderAll();
  console.log(`Canvasのサイズのみ変更しました: ${width} x ${height}`);
};

const clearBackground = () => {
  if (!fabricCanvas) return;
  fabricCanvas.backgroundImage = null;
  fabricCanvas.backgroundColor = 'transparent';
  fabricCanvas.renderAll();
};

const captureAndSetBackground = (videoElement, containerWidth) => {
  return new Promise((resolve, reject) => {
    if (!fabricCanvas || !videoElement) {
      const error = new Error('captureAndSetBackground: 引数が不足しています (fabricCanvas or videoElement)');
      console.error(error);
      return reject(error);
    }
    // ビデオの再生が始まっているか確認
    if (videoElement.readyState < videoElement.HAVE_METADATA) {
      const error = new Error('captureAndSetBackground: ビデオのメタデータが読み込まれていません');
      console.error(error);
      return reject(error);
    }

    console.log('captureAndSetBackground が呼ばれました');

    const videoWidth = videoElement.videoWidth;
    const videoHeight = videoElement.videoHeight;

    // 1. 一時的なCanvasを作成してビデオフレームをキャプチャ
    const tempCanvas = document.createElement('canvas');
    tempCanvas.width = videoWidth;
    tempCanvas.height = videoHeight;
    const tempCtx = tempCanvas.getContext('2d');
    tempCtx.drawImage(videoElement, 0, 0, videoWidth, videoHeight);

    // 2. キャプチャしたフレームからData URLを生成
    const dataUrl = tempCanvas.toDataURL('image/png');

    // 3. Data URLからImage要素を作成
    const imgElement = new Image();
    imgElement.onload = () => {
      console.log('カメラキャプチャからの画像読み込み成功');
      // 4. 既存の関数を呼び出して背景を設定
      resizeAndSetBackground(dataUrl, imgElement, containerWidth);
      resolve({ dataUrl, imgElement }); // 成功時に情報を返す
    };
    imgElement.onerror = (err) => {
      console.error('カメラキャプチャからの画像読み込み失敗', err);
      reject(err); // 失敗時にエラーを返す
    };
    imgElement.src = dataUrl;
  });
};

const deleteActiveObject = () => {
  if (!fabricCanvas) return;
  const activeObject = fabricCanvas.getActiveObject();
  if (activeObject) {
    fabricCanvas.remove(activeObject);
    fabricCanvas.discardActiveObject(); // 選択を解除して selection:cleared イベントをトリガー
    fabricCanvas.renderAll();
  }
};

const positionActiveObject = (preset) => {
  if (!fabricCanvas) return;
  const obj = fabricCanvas.getActiveObject();
  if (!obj) return;

  const canvasWidth = fabricCanvas.width;
  const canvasHeight = fabricCanvas.height;
  const objWidth = obj.getScaledWidth();
  const objHeight = obj.getScaledHeight();

  let newLeft, newTop;

  // プリセットに基づいて位置を計算
  switch (preset) {
    case 'topLeft':     newLeft = 0; newTop = 0; break;
    case 'topCenter':   newLeft = (canvasWidth - objWidth) / 2; newTop = 0; break;
    case 'topRight':    newLeft = canvasWidth - objWidth; newTop = 0; break;
    case 'centerLeft':  newLeft = 0; newTop = (canvasHeight - objHeight) / 2; break;
    case 'center':      newLeft = (canvasWidth - objWidth) / 2; newTop = (canvasHeight - objHeight) / 2; break;
    case 'centerRight': newLeft = canvasWidth - objWidth; newTop = (canvasHeight - objHeight) / 2; break;
    case 'bottomLeft':  newLeft = 0; newTop = canvasHeight - objHeight; break;
    case 'bottomCenter':newLeft = (canvasWidth - objWidth) / 2; newTop = canvasHeight - objHeight; break;
    case 'bottomRight': newLeft = canvasWidth - objWidth; newTop = canvasHeight - objHeight; break;
    default: return; // 不明なプリセットの場合は何もしない
  }

  obj.set({
    left: newLeft,
    top: newTop,
  });

  // オブジェクトの位置を更新した後に再描画
  fabricCanvas.renderAll();

  // オブジェクトがキャンバス外にはみ出さないように調整
  // setCoords()を呼ばないと、コントロールの位置が更新されない
  obj.setCoords(); 
};

const exportOriginalSizeDataURL = (originalImageElement) => {
  try {
    if (!fabricCanvas || !originalImageElement) {
      console.error('exportOriginalSizeDataURL: 引数が不足しています');
      return null;
    }

    // 1. オリジナル解像度
    const originalWidth = originalImageElement.naturalWidth;
    const originalHeight = originalImageElement.naturalHeight;

    // 2. 現在の表示サイズ
    const currentWidth = fabricCanvas.width;

    // 3. スケーリング比率
    const ratio = originalWidth / currentWidth;
    
    if (ratio <= 0 || !isFinite(ratio)) {
      console.error('無効なスケーリング比率です', ratio);
      return null;
    }

    // 4. メモリ上に元サイズの StaticCanvas を作成
    const staticCanvas = new fabric.StaticCanvas(null, {
      width: originalWidth,
      height: originalHeight
    });

    // 5. 背景画像を手動で「再構築」
    const bgImage = fabricCanvas.backgroundImage;
    if (bgImage) {
      // getElement() で <img> 要素を取得
      const bgImgElement = bgImage.getElement();
      const newBgScale = bgImage.scaleX * ratio;
      
      // <img> 要素から新しい FabricImage を作成
      const newBg = new fabric.Image(bgImgElement, {
        scaleX: newBgScale,
        scaleY: newBgScale
      });
      staticCanvas.backgroundImage = newBg;
    }

    // 6. オブジェクト（推し）を個別に「再構築」
    const objects = fabricCanvas.getObjects();
    if (objects.length > 0) {
      objects.forEach(obj => {
        // 7. <img> 要素を取得 (FabricImage のみ)
        if (obj.type === 'image' && obj.getElement) {
          const oshiImgElement = obj.getElement();
          
          // 8. <img> とスケーリング済みのプロパティから新しい FabricImage を作成
          const newOshi = new fabric.Image(oshiImgElement, {
            // スケーリングするプロパティ
            left: obj.left * ratio,
            top: obj.top * ratio,
            scaleX: obj.scaleX * ratio,
            scaleY: obj.scaleY * ratio,
            
            // そのままコピーするプロパティ
            angle: obj.angle,
            flipX: obj.flipX,
            flipY: obj.flipY,
            skewX: obj.skewX,
            skewY: obj.skewY,
            originX: obj.originX,
            originY: obj.originY,
            // ... (他にも必要なプロパティがあればここに追加)
          });
          
          // 9. StaticCanvas に追加
          staticCanvas.add(newOshi);
        }
      });
    }

    // 10. 最終描画
    staticCanvas.renderAll();

    // 11. 高解像度のData URLを生成
    const dataUrl = staticCanvas.toDataURL({
      format: 'png',
      quality: 1.0
    });

    // 12. メモリ解放
    staticCanvas.dispose();

    return dataUrl; // ★ 成功パス

  } catch (error) {
    console.error('exportOriginalSizeDataURL 内部でエラー:', error);
    return null; // ★ 失敗パス
  }
};

const exportWithLiveBackground = (videoElement) => {
  return new Promise((resolve, reject) => {
    try {
      if (!fabricCanvas || !videoElement) {
        reject(new Error('exportWithLiveBackground: 引数が不足しています'));
        return;
      }

      const originalWidth = videoElement.videoWidth;
      const originalHeight = videoElement.videoHeight;
      const currentWidth = fabricCanvas.width;
      const ratio = originalWidth / currentWidth;

      if (ratio <= 0 || !isFinite(ratio)) {
        reject(new Error(`無効なスケーリング比率です: ${ratio}`));
        return;
      }

      // 1. ビデオフレームから背景画像を作成
      const tempCanvas = document.createElement('canvas');
      tempCanvas.width = originalWidth;
      tempCanvas.height = originalHeight;
      tempCanvas.getContext('2d').drawImage(videoElement, 0, 0, originalWidth, originalHeight);
      const videoFrameDataUrl = tempCanvas.toDataURL('image/png');

      // 2. メモリ上にStaticCanvasを作成
      const staticCanvas = new fabric.StaticCanvas(null, {
        width: originalWidth,
        height: originalHeight
      });

      // 3. 背景をセット (非同期)
      fabric.Image.fromURL(videoFrameDataUrl, (bgImg) => {
        staticCanvas.backgroundImage = bgImg;

        // 4. オブジェクトを再構築
        const objects = fabricCanvas.getObjects();
        if (objects.length > 0) {
          objects.forEach(obj => {
            if (obj.type === 'image' && obj.getElement) {
              const oshiImgElement = obj.getElement();
              const newOshi = new fabric.Image(oshiImgElement, {
                left: obj.left * ratio,
                top: obj.top * ratio,
                scaleX: obj.scaleX * ratio,
                scaleY: obj.scaleY * ratio,
                angle: obj.angle,
                flipX: obj.flipX,
                flipY: obj.flipY,
                skewX: obj.skewX,
                skewY: obj.skewY,
                originX: obj.originX,
                originY: obj.originY,
              });
              staticCanvas.add(newOshi);
            }
          });
        }

        // 5. 最終描画とエクスポート
        staticCanvas.renderAll();
        const dataUrl = staticCanvas.toDataURL({ format: 'png', quality: 1.0 });
        staticCanvas.dispose();
        resolve(dataUrl);
      });
    } catch (error) {
      console.error('exportWithLiveBackground 内部でエラー:', error);
      reject(error);
    }
  });
};

defineExpose({
  resizeAndSetBackground,
  resizeCanvas,
  clearBackground,
  addOshiImage,
  deleteActiveObject,
  positionActiveObject,
  exportOriginalSizeDataURL,
  exportWithLiveBackground
});
</script>

<style scoped>
/*
  <canvas> タグを囲むラッパー（自動生成）に
  スタイルを適用するため :deep() を使います
*/
:deep(.canvas-container) {
  /*
    親要素 (App.vueの .canvas-wrapper) の
    サイズに追従するようにします
  */
  max-width: 100%;
  max-height: 100%;

  /* スマホ画面で縮小表示されたときもくっきりさせる */
  image-rendering: -webkit-optimize-contrast;
  image-rendering: crisp-edges;
}
</style>
