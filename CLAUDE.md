# CLAUDE.md

このリポジトリで作業するときの、Claude Code 向けガイドです。

## プロジェクト概要

**うつしみ**（`utsushimi.html`）は、単一のHTMLファイルで完結するWebアプリです。
AIがランダムに架空の村人を生成し、ユーザーがその村人たちの物語を小説として書き溜めていく道具です。

- サブタイトル: 「物語で間合いをはかる」
- 元になった課題: 「人間とテクノロジーの望ましい関係を探る」というテーマの問いマップ（`問いマップ_吉村.pdf`）。
- 発想の核: 問いに向き合う方法としての「**フィクションに仮託する**」——物語の登場人物に感情的に憑依し、その周辺の出来事・感情・関係を想像して描くことで、自分の思考や感情をフィクションに預ける営み。
- 任天堂「トモダチコレクション」の逆。人間が設定を作るのではなく、AIが設定をランダムに与え、人間が物語を描く。

## 設計思想（変更前に必ず読む）

この道具の仕様は美観ではなく思想から来ています。実装判断はここに従ってください。

1. **設定はランダム（制御不能）であること。** 制御を感じるのは「設定」の部分。だから村人の性格・過去・容姿・関係はAIがランダムに与える。物語の続きに都合よく寄せた人物を作ってはいけない。整合性より偶然性を優先し、「大きな矛盾」だけ避ける。
2. **物語を描く行為は制御ではなく「間合いをはかる」こと。** ユーザーが物語を書く部分は自由。ここに制約を足さない。
3. **一度与えた設定は変更不可。** ユーザーもAIも村人の設定を書き換えられない（実装では `Object.freeze`）。※例外として「立ち絵の画像」だけはユーザーが差し替え／削除できる。
4. **設定はパラメータの数値ではなく、エピソードを含む散文で提示する。**

## ファイル構成

- `utsushimi.html` — 本体（HTML/CSS/JSすべて内包、約3.2MB。大半は埋め込み背景画像）。**成果物はこの1ファイルのみ。**
- `背景プロンプト16枚.md` — 村の背景（季節×時間帯16枚）をAI画像生成ツールで作るためのプロンプト集。
- `問いマップ_吉村.pdf` — 元課題の資料。
- `CLAUDE.md` — このファイル。

## 動作確認・開発

- **実行**: ブラウザで `utsushimi.html` を開くだけ。ビルド不要。
- **このリポジトリにはブラウザ実行環境がない場合が多い。** 目視確認は以下で代替してきた:
  - JS構文チェック: HTMLから `<script>` を抽出して `node --check`。
  - SVG（立ち絵・フォールバック風景）の描画確認: **librsvg** を ctypes 経由で叩いて PNG 化（`librsvg-2.so` + libcairo）。ImageMagick(`convert`)はグラデーション/フィルタ/clipPathを正しく描けないので使わない。
  - 立ち絵は表示上 82px 程度に縮むので、拡大レンダーで太く見えても実機では細い。
- **ファイルの上書きに注意**: 既存 `utsushimi.html` を直接 `open(...,'w')` で上書きできないことがある（権限）。その場合は一時ファイルに書き出し → 削除 → コピーで置き換える。

## アーキテクチャ

単一 `<body>` に、`<style>` 1つと `<script>` 2つ:

1. `<script>window.SCENE_IMG={...}</script>` — 16枚の背景JPEGを `"季節|時間帯" → dataURI` で持つ。**アプリ本体スクリプトより前に置く**こと（起動時に参照するため）。
2. アプリ本体スクリプト（状態管理〜起動）。

### 状態と永続化

- localStorageキー: `yosuga_village_v1`
- `state = { mode, apiKey, villageName, villageDescription, villagers[], novels[], portraits{}, createdAt, lastVillagerAt }`
  - `villagers[]`: 生成された村人（`appearance` などを含む。`arrivedAt` で識別）。
  - `novels[]`: `{ id, title, body, createdAt, updatedAt? }`
  - `portraits{}`: `"name|arrivedAt" → PNG dataURI`（ユーザーがアップした立ち絵）。
- `save()` / `load()` で読み書き。`load()` は旧データも `mode="api"`・`portraits={}` に補正する。

### 生成（AI呼び出し）

- `callClaude(prompt, maxTokens)` が `PROXY_URL`（Cloudflare Worker中継）へPOST。**APIキーはこのファイルに書かず、プロキシ側のSecretに保管**。`SYSTEM_PROMPT` は user 発言の先頭に折り込む。
- `makeConstraints()` がクライアント側で乱数制約（髪の長さ/結い方/色/年齢/発想の種）を作り、プロンプトに埋め込む。AIはこの制約に従う。
- `buildInitialPrompt()`（村＋最初の3人）／`buildNewVillagerPrompt()`（増員）。出力は有効なJSONのみ。
- 生成ルール要点: 名前は人種を固定しない（日本語風でない名はカタカナ）。年齢は人外もOK（その場合 `apparentAge` に「人間でいうと何歳か」を書く）。`hairStyle` を立ち絵に反映。既存物語との「大きな矛盾」だけ避ける。

### 主な関数（役割 / だいたいの位置）

- `makeConstraints` / `constraintText` — 乱数制約。
- `buildInitialPrompt` / `buildNewVillagerPrompt` — プロンプト組み立て。
- `callClaude` / `generate` — 生成呼び出し。
- `portraitSVG(v)` — 既定の立ち絵（**素朴な絵本調**、viewBox `0 0 120 150`、名前でシード）。
- `portraitHTML(v)` — カスタム画像があれば `<img class="pfp">`、なければ `portraitSVG`。**表示は必ずこれ経由**（村・顔ぞろえ・詳細・来訪）。
- `sceneSVG()` — 背景画像が無いときのフォールバック風景（湖畔の村・季節時間で変化）。
- `renderScene()` — `#villageBg`(`<img>`)の `src` に `SCENE_IMG[季節|時間]`（無ければ sceneSVG）をセット。
- `applyTheme()` — 季節×時間からUI配色（CSS変数）を切り替え。
- `currentST()` / `seasonOf` / `timeOf` / `tickClock` — 時刻判定と「村の時間」時計、季節時間が変わったら theme+scene 更新。
- `weeklyCheck` / `welcome` / `invite` — 週次増員（最大25）と手動招待。
- 物語: `renderLibrary` / `openReader` / 保存・上書き（`saveNovelBtn`）・編集（`readerEdit`/`cancelEditBtn`）。

## 機能ごとのポイント

- **村人増員**: `WEEK = 7日`ごとに1人、`MAX_AUTO_POP = 25` まで自動。到達後は「新しい村人を招く」ボタン。
- **立ち絵の絵柄**: 細い茶色の輪郭・丸い顔・ハイライトや頬紅なし・目は小さめ・口もとは無表情/ひかえめな笑み/口をあけた笑顔を人物ごとに出し分け。髪型 enum は `down|ponytailLow|ponytailHigh|bun|braid|twin`、長さ enum は `bald|veryShort|short|medium|long|veryLong`。`otherFeatures`/`personality` のテキストから髭・眼鏡・そばかす・スカーフを検出して描画。
- **立ち絵の差し替え**: 村人詳細モーダルの「立ち絵を変更」。透過PNG（横400×縦500目安、4:5）を推奨。アップ画像は canvas で最大440×560に縮小しPNG dataURIで `state.portraits` に保存。「既定のイラストにもどす」で削除→SVGに戻る。保存失敗（容量超過）時はロールバックして警告。
- **背景（16枚）**: `SCENE_IMG` の dataURI。`<img id="villageBg">` + `object-fit:cover` で全面表示（比率ズレで端に隙間が出ない）。差し替えは `背景プロンプト16枚.md` で画像を作り直し、再エンコードして `SCENE_IMG` を更新。
- **UIテーマ（6案）**: `THEME_PALETTES`（`beige`=01 / `sepia`=02 / `bluegrey`=04 / `pastel`=05 / `navy`=06）、`PALETTE_MAP` で `季節|時間 → パレット` を割り当て（夕=セピア, 夜=ネイビー 等）。`applyTheme` がCSS変数をセットし、`navy` のとき `<html class="dark">`。

## 設計上まもること

- **単一HTMLファイルを維持。** 画像・スクリプトは外部ファイル化せず dataURI/インラインで内包する。
- **永続化はブラウザの localStorage のみ。**
- **村人の設定は不変**（立ち絵画像を除く）。編集/カスタマイズ機能を足さない。
- **偶然性を優先**：物語の内容から村人を逆算しない。
- **この環境ではAI画像生成（拡散モデル）は使えない**（コネクタ/スキル無し）。ラスタ画像が必要ならユーザーに生成を依頼して受け取り、埋め込む。
- UI文言・生成物は日本語。トーンは静かでやわらかく。

## よくある変更のレシピ

- **髪型を増やす**: `HAIR_STYLE_JP` に追加 → `makeConstraints` の重み → `portraitSVG` の描画分岐 → プロンプトの `hairStyle` enum 説明。
- **テーマ割り当てを変える**: `PALETTE_MAP` を編集（必要なら `THEME_PALETTES` に色案を追加）。暗いテーマを足すときは、白固定の背景（`#fff`）を `var(--card)` に、名前ピルの文字色などが読めるか確認。
- **背景を差し替える**: 16枚を再生成 → 幅1500・JPEG q80程度に最適化 → base64化して `SCENE_IMG` を置換。`"季節|時間帯"` キーは `spring|morning` 形式。
- **立ち絵の絵柄調整**: `portraitSVG` 内。太さは `S`/`St` の stroke-width、目位置は `eyes` の cx、口は `mouth` の分岐。

## 落とし穴

- **localStorage 容量**: 立ち絵画像を多数保存すると上限（〜5MB）に近づく。保存は try/catch でロールバックしている。
- **暗テーマの可読性**: `navy` 時に白背景＋濃文字が反転して読めなくなる箇所に注意。テーマ変数（`--card`/`--ink` 等）で組む。
- **`SCENE_IMG` の読み込み順**: 本体スクリプトより前に定義されていること。
- **プロキシ依存**: 生成は `PROXY_URL` 前提。落ちると生成できない（エラーバナー＋再試行あり）。
