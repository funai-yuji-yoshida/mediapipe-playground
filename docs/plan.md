# MediaPipe お試しアプリ 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** MediaPipe の11機能を、デモ表示と調査表示を切り替えながら試せる単一HTMLを作る。

**Architecture:** `index.html` 1枚にCSS・JSを含める。JSは ES modules として jsDelivr から MediaPipe Tasks を読み込む。機能ごとに「定義オブジェクト（FEATURES 配列の1要素）」を持ち、共通の実行ループ（入力取得 → 推論 → 描画 → 計測）がそれを呼ぶ。

**Tech Stack:** HTML/CSS/素のJS（ビルドなし）、`@mediapipe/tasks-vision` `@mediapipe/tasks-audio` `@mediapipe/tasks-text` いずれも **1.2.2**（jsDelivr）、GitHub Pages。

**Spec:** `docs/design.md`

## Global Constraints
- 単一HTML（`index.html`）。外部読み込みは jsDelivr の MediaPipe 3パッケージ（版は 1.2.2 に固定）と Google のモデル置き場のみ
- 表示テキスト・エラーメッセージはすべて日本語
- PCの Chrome / Edge で動作必須。スマホは動けばよい
- 映像・音声を外部へ送信しない
- モデルURLは `MODEL_BASE` 定数1か所から組み立てる（後でリポジトリ内 `models/` に差し替えられるように）
- 結果の保存・ログ記録はしない。文章分類と端末内LLMは載せない

## Review Focus
1. カメラ／マイクの許可を拒否、またはデバイスが無い → 日本語の案内を表示し、画像ファイル入力に誘導する（Task 2 で確認）
2. 機能を素早く連続で切り替える → 前の機能のループとモデルが止まり、描画が混ざらない（Task 2 で確認）
3. モデル読み込み中・失敗（オフライン） → 「読み込み中…」表示、失敗時は日本語エラーと再試行ボタン（Task 2 で確認）
4. GPU 初期化に失敗する環境 → 自動で CPU にフォールバックし、調査表示にその旨を出す（Task 2 で確認）
5. 縦長・大きな画像ファイル → 枠内に縦横比を保って収まり、重ね描きの座標がずれない（Task 3 で確認）

## ファイル構成
- `index.html`：アプリ本体（全機能）
- `docs/design.md`：設計書（既存）
- `README.md`：使い方と公開URL
- `.claude/launch.json`（ホーム直下）：ローカル確認用サーバー定義（`python -m http.server 8765`、作業フォルダ `mediapipe-playground`）

## 共通インターフェース（index.html 内）
```js
// 機能定義
{
  id: 'hand',                    // 一意
  group: '手',                   // 左一覧の区分
  name: '手のランドマーク',
  inputs: ['camera','image'],    // 'camera'|'image'|'video'|'mic'|'text' の部分集合
  demoText: '指先21点を追跡…',   // デモ表示の説明
  useCase: '非接触での操作',      // 「業務でこう使える」
  options: [{key:'numHands', label:'手の数', min:1, max:4, step:1, value:2}], // 調査表示のスライダー
  models: [{label:'標準', path:'hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task'}],
  create(fileset, modelUrl, delegate, opts) → Promise<task>,
  run(task, source, timestampMs) → result,      // 'image' 入力時は detect、他は detectForVideo
  draw(ctx, result, w, h),
  summarize(result) → string,                    // デモ表示の一行要約（例「手を2つ検出」）
}
```
共通関数：`loadFeature(id)`, `stopCurrent()`, `startLoop()`, `setInput(kind)`, `showError(msg, retryFn)`, `setMode('demo'|'lab')`

---

### Task 1: 土台（画面枠・表示切替・入力切替・ローカル確認）
**Files:** Create `index.html`, `README.md`, `C:\Users\F4237\.claude\launch.json` に設定追加（既存なら追記）
- [ ] 3カラム画面（左：機能一覧、中央：`<video>`＋`<canvas>`重ね、右：説明パネル）と上部の「デモ／調査」切替を作る
- [ ] 入力切替（カメラ／画像／動画／マイク／テキスト）を作り、機能の `inputs` に無いものは無効化する
- [ ] カメラ起動 `getUserMedia({video:{width:1280,height:720}})`、拒否時は `showError('カメラの使用が許可されていません。ブラウザのアドレスバー横のアイコンから許可するか、「画像」を選んでください。')`
- [ ] 確認：preview_start で開き、カメラ映像が中央に出る・切替でパネル表示が変わる・コンソールエラーなし
- [ ] `git init` してコミット `feat: 画面の土台`

### Task 2: 実行ループと手のランドマーク（最初の1機能で共通部を完成させる）
**Interfaces:** Produces 上記の機能定義と共通関数
- [ ] `FilesetResolver.forVisionTasks('https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@1.2.2/wasm')`、`MODEL_BASE='https://storage.googleapis.com/mediapipe-models/'`
- [ ] `loadFeature` は世代番号を持ち、読み込み完了時に世代が古ければ破棄（Review Focus 2）。`stopCurrent` は rAF停止・`task.close()`
- [ ] GPU で `create` 失敗時は CPU で再作成し、調査表示に「GPU不可のためCPUで実行中」（Review Focus 4）
- [ ] 読み込み中表示、失敗時は日本語エラー＋再試行ボタン（Review Focus 3）
- [ ] 調査表示：FPS、処理時間ms（直近30フレーム平均）、options のスライダー、CPU/GPU 切替、モデル選択、結果JSON（先頭2000文字）
- [ ] 手のランドマーク実装（`DrawingUtils` で点と骨格線）
- [ ] 確認：手をかざして骨格が重なる／調査表示でFPSが出る／DevToolsでオフラインにして日本語エラーが出る／機能を連打してもエラーなし
- [ ] コミット `feat: 実行ループと手のランドマーク`

### Task 3: 画像・動画入力
- [ ] 画像ファイル：読み込み後 `runningMode:'IMAGE'` で1回推論し描画。canvas は縦横比を保って枠に収める（Review Focus 5）
- [ ] 動画ファイル：`<video>` に読み込み、再生中ループで推論
- [ ] runningMode の切替は `task.setOptions({runningMode})`
- [ ] 確認：縦長写真で骨格がずれない、動画再生で追従する
- [ ] コミット `feat: 画像・動画入力`

### Task 4: 画像系の残り8機能
各機能は定義オブジェクトの追加のみ。モデルパス（`MODEL_BASE` 以降）：
| id | 機能 | モデル（既定 / 調査用の別モデル） | 主なoption |
|---|---|---|---|
| face | 顔検出 | `face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite` | minDetectionConfidence |
| faceMesh | 顔＋表情 | `face_landmarker/face_landmarker/float16/1/face_landmarker.task`（`outputFaceBlendshapes:true`） | numFaces |
| gesture | ジェスチャー | `gesture_recognizer/gesture_recognizer/float16/1/gesture_recognizer.task` | numHands |
| pose | 姿勢 | `pose_landmarker/pose_landmarker_lite/float16/1/pose_landmarker_lite.task` / `…_full/…_full.task` / `…_heavy/…_heavy.task` | numPoses |
| object | 物体検出 | `object_detector/efficientdet_lite0/float16/1/efficientdet_lite0.tflite` / `efficientdet_lite2/float16/1/efficientdet_lite2.tflite` | scoreThreshold, maxResults |
| classify | 画像分類 | `image_classifier/efficientnet_lite0/float32/1/efficientnet_lite0.tflite` | maxResults |
| selfie | 背景切り抜き | `image_segmenter/selfie_segmenter/float16/latest/selfie_segmenter.tflite` | 背景（ぼかし／単色） |
| magic | クリック切り抜き | `interactive_segmenter/magic_touch/float32/1/magic_touch.tflite` | 入力は画像のみ。クリック位置を keypoint に |
- [ ] 表情は上位5つの blendshape を日本語名（笑顔・まばたき等の対応表）で棒表示。ジェスチャー名・物体名も日本語対応表で表示（未対応名は英語のまま）
- [ ] 確認：各機能をカメラか画像で1回ずつ動かし、描画と要約が出る
- [ ] コミット `feat: 画像系8機能`

### Task 5: 音の分類と言語判定
- [ ] 音：`tasks-audio@1.2.2`、`audio_classifier/yamnet/float32/1/yamnet.tflite`。マイク入力を AudioWorklet か ScriptProcessor で16kHz相当に渡し、上位5ラベルを棒表示。マイク拒否時は日本語案内
- [ ] 言語判定：`tasks-text@1.2.2`、`language_detector/language_detector/float32/1/language_detector.tflite`。テキスト欄入力ごとに判定し、言語コード→日本語名（ja→日本語 など主要20言語）
- [ ] 確認：拍手・声で結果が変わる／「こんにちは」「Hello」で判定が変わる
- [ ] コミット `feat: 音の分類と言語判定`

### Task 6: 仕上げと公開準備
- [ ] デモ表示の説明文・「業務でこう使える」を全11機能分そろえる。初期表示は「手のランドマーク」をカメラで起動（30秒目標）
- [ ] 幅768px未満では機能一覧を上部のプルダウンに
- [ ] README に使い方・公開手順を記載
- [ ] 確認：全機能を通しで切り替えてコンソールエラーなし、スクリーンショットで報告
- [ ] コミット `chore: 仕上げ`
- [ ] GitHub リポジトリ作成と Pages 公開は、**ユーザーの確認を得てから**行う
