# tono. 技術仕様書

このドキュメントは、`tono.` を保守・拡張するエンジニア向けの仕様メモです。

## 概要

`tono.` は、単一の `index.html` を中心に構成された静的なフロントエンドアプリです。

ビルド工程やnpm依存はありません。ブラウザ上で写真を読み込み、DOM/CSSでカードプレビューを組み立て、`html2canvas` を使ってPNGとして書き出します。

## 技術スタック

- HTML / CSS / Vanilla JavaScript
- `html2canvas` 1.4.1
  - CDN読み込み
  - PNG exportで使用
- Google Fonts
  - `Cormorant Garamond`
- PWA
  - `manifest.json`
  - `sw.js`

## 主要ファイル

```text
index.html
  アプリ本体。CSS、HTML、JavaScriptがすべて含まれる。

manifest.json
  PWA manifest。

sw.js
  service worker。静的アセットのキャッシュを担当。

icon-512.png
  PWA用アイコン。
```

## 画面構造

アプリの大きな構造は以下です。

```text
body
├─ header
├─ .app-shell
│  ├─ .canvas-area
│  │  └─ #storyCanvas.story-canvas
│  │     ├─ #photoFrame.photo-frame
│  │     │  ├─ #canvasPhoto
│  │     │  └─ #noPhoto
│  │     ├─ #storyText
│  │     ├─ #decoLine
│  │     ├─ #watermark
│  │     ├─ #paletteStrip
│  │     ├─ #selvedgeBandH
│  │     ├─ #selvedgeBandV
│  │     └─ #lockOverlay
│  └─ #bottomSheet
│     ├─ .controls
│     └─ .export-area
└─ #proModal
```

スマホでは `#bottomSheet` が固定ボトムシートになり、プレビューカードは `transform: scale(...)` で表示だけ縮小されます。カード内部の基準サイズや座標系は変えない方針です。

## プレビューカード

プレビュー本体は `#storyCanvas` です。

基準サイズ:

```css
.story-canvas {
  width: 280px;
  height: 498px;
}
```

デスクトップの広い画面では以下の上書きがあります。

```css
@media (min-width: 1024px) {
  .story-canvas {
    width: 320px;
    height: 569px;
  }
}
```

## 主要状態

`index.html` 内のJavaScriptでグローバル状態を管理しています。

```js
let isPro = false;
let extractedColors = [];
let currentBgRgb = [255, 255, 255];
let hasPhoto = false;
let photoZoom = 1;
let photoX = 0;
let photoY = 0;
let photoRotation = 0;
let dragMode = 'free';
let canvasScale = 1;
```

### 写真操作

写真は `#canvasPhoto` に読み込まれます。

主な操作状態:

- `photoZoom`: 写真ズーム。下限 `1`、上限 `4`。
- `photoX`: 写真のX方向移動量。
- `photoY`: 写真のY方向移動量。
- `photoRotation`: 写真だけの角度。`-15` から `15` 度。
- `dragMode`: `free` / `x` / `y`。

`applyPhotoTransform()` が、これらの状態をCSS transformへ反映します。

不定形レイアウトでは、フレーム自体が少し回転しています。写真まで傾かないよう、`getFrameRotation()` でフレーム角度を取得し、写真側で打ち消しています。

## 写真プリセット

PHOTOタブの `width` / `height` は `setPhotoPreset()` で処理します。

- `width`
  - 写真の横幅がフレーム幅に届くズーム値へ変更
  - 写真位置を中央へ戻す
  - ドラッグモードを `y` に変更
- `height`
  - 写真の縦幅がフレーム高さに届くズーム値へ変更
  - 写真位置を中央へ戻す
  - ドラッグモードを `x` に変更

ズーム値は `getPhotoPresetZoom()` で、写真の自然サイズと現在の `#photoFrame` サイズから算出します。

## レイアウト

レイアウトは `#storyCanvas` のクラスで切り替えます。

例:

```html
<div class="story-canvas layout-top" id="storyCanvas">
```

`setLayout(el)` は、選択されたサムネイルの `data-layout` を使って `storyCanvas.className` を更新します。

### 現在の主なレイアウト

- `layout-top`
- `layout-center`
- `layout-film`
- `layout-edge`
- `layout-palette`
- `layout-split`
- `layout-frame`
- `layout-circle`
- `layout-organic`
- `layout-organic-2`
- `layout-organic-3`
- `layout-duo`
- `layout-selvedge-h`
- `layout-selvedge-v`
- `layout-land-top`
- `layout-land-center`
- `layout-land-frame`
- `layout-port-top`
- `layout-port-center`
- `layout-port-frame`

### レイアウト追加手順

1. CSSに `.story-canvas.layout-xxx .photo-frame` を追加します。
2. 必要なら `.story-canvas.layout-xxx .story-text`、`.deco-line`、`.watermark` を調整します。
3. サムネイル用に `.t-xxx .lt-photo` などを追加します。
4. Layoutタブの `.layout-grid` に以下のようなHTMLを追加します。

```html
<div class="layout-thumb t-xxx" data-layout="layout-xxx" data-pro="true" onclick="setLayout(this)">
  <div class="lt-photo"></div><div class="lt-line"></div>
  <span class="thumb-name">name</span><span class="ratio-badge">free</span>
</div>
```

5. 古いCanvas合成系の互換関数を残す場合は、`exportGeo()` と `exportText()` にもケースを追加します。

現在の実ExportはDOM撮影なので、最重要なのはCSSとLayoutタブへの追加です。

## 背景色

初期背景は白です。

```js
let currentBgRgb = [255, 255, 255];
```

`renderBgBasePalette()` の最後で `applyBackground(255, 255, 255)` を呼び、初期状態を白にしています。

写真読み込み時に色抽出は行いますが、背景色の自動変更はしません。抽出色は候補としてUIに表示するだけです。

## 色抽出

`extractColors(img)` が写真を小さなcanvasに描画し、簡易k-meansで6色を抽出します。

抽出色は以下に使われます。

- 写真由来の背景色候補
- `layout-palette` のカラーストリップ
- `layout-selvedge-*` の色丸

## テキスト

テキストは以下のDOMに反映されます。

- `#dateLine`
- `#captionDisplay`
- `#subDisplay`

表示 / 非表示は `textVisibility` で管理します。

```js
const textVisibility = { caption: true, sub: true, date: true };
```

`toggleTextItem()` はDOM表示とボタン状態を同期します。

## PNG Export

Exportの入口は `exportImage()` です。

### 方針

ユーザーが編集画面で見ているカードをそのまま書き出すため、`#storyCanvas` を `html2canvas` で撮影します。

ただし、`html2canvas` は `img + object-fit + transform` の組み合わせで、ブラウザ表示と違う切り抜きになることがあります。そのため、Export時のみ写真部分を内部canvasに描画して安定化しています。

### 処理の流れ

1. 写真が読み込まれているか確認。
2. フォントと画像decodeを待つ。
3. モバイル表示用の外側scaleを一時的に外す。
4. `installExportPhotoCanvas()` を呼ぶ。
   - 現在の `photoX` / `photoY` / `photoZoom` / `photoRotation` を使う。
   - `#photoFrame` 内に高解像度canvasを追加する。
   - 元の `#canvasPhoto` は一時的に非表示にする。
5. `html2canvas(storyCanvas, captureOptions)` を実行。
6. PNG Blob化してダウンロード。
7. 一時変更を元に戻す。

### 出力サイズ

カード幅が1080px相当になるようにscaleを計算します。

```js
const targetWidth = 1080;
const scale = targetWidth / storyCanvas.offsetWidth;
```

## PWA

`manifest.json` と `sw.js` があります。

### 注意点

- `manifest.json` は `icon-192.png` を参照していますが、現在のファイル一覧には `icon-192.png` がありません。
- `sw.js` のキャッシュ対象フォントURLには `DM Sans` が含まれていますが、現在の `index.html` では主にシステムフォントと `Cormorant Garamond` を使っています。

PWAとして整える場合は、この2点を見直してください。

## Pro / 課金まわり

現在は「まず自由に使ってもらう」方針のため、課金導線は非表示です。

ただし、将来の課金対応に戻しやすいように、Pro用のDOM、CSS、関数、`data-pro` 属性は残しています。

### 現在非表示にしているもの

- ヘッダーの `upgrade to pro` ボタン
  - `#proBtn`
  - HTML上は `.pro-badge` として残しています。
  - `.visible` クラスを付けると表示できます。
- Layoutタブ下部のPro案内文
  - `.pro-upsell-copy`
  - 現在はCSSで `display: none;` にしています。
- Proモーダル
  - `#proModal`
  - `openModal()` が呼ばれなければ表示されません。
- Proロック用オーバーレイ
  - `#lockOverlay`
  - ロック判定を復活させた時に使います。

### Proボタンを復活させる手順

ヘッダーのボタンに `.visible` を付けます。

```html
<button class="pro-badge visible" id="proBtn" onclick="openModal()">upgrade to pro</button>
```

必要なら、Pro案内文も表示します。

```css
.pro-upsell-copy { display: block; }
```

または、`.pro-upsell-copy` の `display: none;` を削除します。

### テンプレートロックを復活させる手順

各レイアウトサムネイルには `data-pro` が残っています。

```html
<div class="layout-thumb t-fil" data-layout="layout-film" data-pro="true" onclick="setLayout(this)">
```

`setLayout()` の冒頭に、以下の判定を戻します。

```js
if (!isPro && el.dataset.pro === 'true') {
  openModal();
  lockOverlay.classList.add('show');
  return;
}
```

この判定を入れると、`data-pro="true"` のテンプレートを無料状態で選ぼうとした時にProモーダルを開けます。

ロック表示をサムネイルにも出したい場合は、初期化時に `data-pro="true"` の `.layout-thumb` へ `.locked` と `.lock-chip` を付与する処理を追加してください。

例:

```js
function applyProLocks() {
  document.querySelectorAll('.layout-thumb[data-pro="true"]').forEach(t => {
    t.classList.add('locked');
    if (!t.querySelector('.lock-chip')) {
      const chip = document.createElement('span');
      chip.className = 'lock-chip';
      chip.textContent = 'pro';
      t.appendChild(chip);
    }
  });
}
```

`activatePro()` は現在も残っています。課金決済や認証が成功した後に呼び出す想定です。

```js
activatePro();
```

### 課金実装時の注意

- `isPro` は現在メモリ上の状態だけです。リロードすると戻ります。
- 本番化する場合は、Stripeなどの決済結果とユーザー状態をサーバー側で管理してください。
- Pro状態を永続化する場合でも、クライアント側の `localStorage` だけを信頼しないでください。
- ロック対象は `data-pro` で制御するのが一番戻しやすいです。

## 保守時の注意

- Export品質を守るため、写真の表示仕様を変える場合は `applyPhotoTransform()` と `installExportPhotoCanvas()` の両方を確認してください。
- 新しいレイアウトで `photo-frame` を回転させる場合は、`getFrameRotation()` へ角度を追加してください。
- `storyCanvas` の外側scaleはモバイル表示用です。Export時には一時的に外しています。
- `html2canvas` はCSS対応に限界があります。影やマスクを大きく変える場合は、プレビューと書き出しの差分を必ず確認してください。
- `index.html` が大きくなっているため、今後拡張が続く場合はCSS / JS分割を検討してください。

## 確認コマンド

インラインスクリプトの構文チェック:

```powershell
node -e "const fs=require('fs'); const html=fs.readFileSync('index.html','utf8'); const scripts=[...html.matchAll(/<script>([\s\S]*?)<\/script>/gi)].map(m=>m[1]); for (const s of scripts) new Function(s); console.log('inline scripts parse ok:', scripts.length);"
```

ローカル静的サーバー:

```powershell
python -m http.server 4173 --bind 127.0.0.1
```

ブラウザで開く:

```text
http://127.0.0.1:4173/index.html
```
