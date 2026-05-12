---
status: draft-source
date: 2026-05-12
title_candidate: 営業資料を Claude Code でテンプレ化した話（仮）
slug_candidate: sales-deck-template
purpose: 第10話相当の記事化素材ログ
---

# 営業資料テンプレ作成の素材ログ

note執筆スキルを通す前の生メモ。試行錯誤と気付きを時系列で残す。

## 背景

- 営業資料を案件ごとに1から作っていた状態を見直したかった
- 中身の大部分（サービス紹介・特長・料金・制作フロー）は案件横断で共通
- 案件ごとに差し替えるのは「顧客名・日付・実績ピック・参考記事」だけ
- なら、その差し替え箇所だけYAMLで書けば、HTMLが自動生成される仕組みにできる

## 最終仕様

- 16ページ構成のHTMLスライド（1920×1080）。Ctrl+P → PDF保存で1ファイル化
- ブラウザの矢印キーで操作。VSCodeのプレビューでも見られる
- 顧客プロファイル `clients/<案件>.yaml` → `build.py` → `output/<案件>/index.html`
- works件数で自動分岐：4件版（P12）、3件版（P13・概要付き）、6件汎用（P11）
- 自動ページ再採番：works_variant に応じて P11/P12/P13 のいずれかが削除され、以降のページ番号は繰り上がる
- ブランド：#5DA8D4（cyan）一色＋ニュートラル。緑・赤・オレンジは使わない
- ロゴ：logo-smarvee-ai.png（Smarvee + AI 一体型）を全ヘッダー＆表紙メインに使用
- 全カード共通仕様：背景 #F9FAFB / 上部4px青ライン（::before）/ 角丸24px / overflow hidden

## 試行錯誤と気付き

### 1. ページ番号の認識ずれが、一番ハマった

「P13がおかしい」と言われた時、自分（Claude側）はテンプレP13（Works 3件版＝主要事例3枚を概要付きで載せるページ）を見ていた。

でも実際にユーザーが画面で見ていたのは、自動再採番後の「表示上P13」＝無料ツールページだった。テンプレP14＝無料ツールが、build.pyの再採番で表示上13番に繰り上がっていた。

**学び**：人間とAIの間で「ページ番号」みたいな共有指示子が両者で違うレイヤーを指すことがある。プレビューURLの`#13`を私が指示した時点で、ユーザーは表示上の13を見ていた。「テンプレ番号で言うと」「表示番号で言うと」と毎回明示する習慣をつけるべきだった。

### 2. サムネが16:9で出ない（レターボックス問題）

works のグリッド（3列×2行）で `grid-template-rows: repeat(2, minmax(0, 1fr))` を使うと、grid セルの高さが body 残り全部を2分割になる。各cardはセルに stretch される。

thumb に `aspect-ratio: 16/9` を指定していても、flex shrink が効いてセル高さに合わせて縦が引き伸ばされる。結果、Vimeo Player の中で動画が縦長コンテナにフィットしようとして、上下に黒帯が出る。

**対処**：thumb に `flex: 0 0 auto` を追加してshrinkを禁止。grid自体は `auto auto + align-content: center` に変えて、行高を内容に従わせる。

これは g4 では既に対応していたが、g6 で同じ処理が抜けていた。

### 3. 「水色がフレームの上のレイヤーです」

カード上部に4pxの青ライン（::before）を全カードに追加していたが、無料ツールカード（.tool）だけ `overflow: hidden` がついていなかった。

結果、青ラインが border-radius の角丸でクリップされず、カードの角からはみ出して「フレームの上に浮いた水色の帯」として見える状態に。

ユーザーから「Z-indexの問題ですか？」と聞かれて、最初はz-indexを疑ったが、本当はoverflowの問題だった。

**学び**：他カード（.work、.article）と一見同じ仕様に見えても、コピペし忘れたプロパティがあると微妙な見え方の違いが出る。共通カード仕様は最初から共通CSSクラスとして括っておくべきだった（今は個別記述）。

### 4. excerpt の冒頭タグが残る

WordPressのportfolio excerptに「【制作実績】【記念式典】」みたいなタグ的プレフィックスが入っていることがあった。

そのままtruncate(60)すると「【記念式典】家電製品協会の…来…」みたいに途中で切れて、見た目が「ディスクリプションから引っ張った感」になる。

**対処**：`clean_excerpt` に冒頭【XX】を除去する正規表現を入れる：

```python
s = re.sub(r'^[\s　]*(?:【[^】]*】[\s　]*)+', '', s)
```

その後、truncateは60→90→200と段階的に拡大。最終的にCSSのpを13px/line-height1.8にして、4件版カードでも省略させずに全文表示できるバランスに。

### 5. body padding-bottom 0 で全ページのフッター窮屈

`.body { padding: 36px var(--pad-x) 0; }` だと、bodyのコンテンツがフッターに密接してしまう。

全ページ共通で`padding-bottom: 36px`を入れたら、ほぼ全ページでフッターまで36pxの余白を確保できた。

ただしP6 Featuresだけ、grid 2x2が `flex:1` で残り高さ全部埋めるレイアウトで、内容が大きいと padding-bottom が消費される。対処：`grid-template-rows: repeat(2, 1fr)` → `repeat(2, minmax(0, 1fr))` ＋ `min-height: 0` でgrid min-content制約を外す。

## 記事化の方向（連載の文脈で）

- ひとりDX担当として「営業資料の量産を仕組み化した」話として書ける
- AI（Claude Code）にレイアウトの試行錯誤を任せ、人間はYAMLだけ書く構造
- ターゲット読者：案件ごとに営業資料を作り直していて、しんどい中小企業の担当者
- 連載トーンに乗せるなら、「ページ番号の認識ずれ」「水色レイヤー問題」みたいに具体的なつまずきをそのまま書ける

## 関連ファイル

- 本体：`C:\Users\hseki\smarvee-decks\_template\client-deck.html`
- ビルダー：`C:\Users\hseki\smarvee-decks\_scripts\build.py`
- 顧客プロファイル例：`C:\Users\hseki\smarvee-decks\clients\テスト株式会社.yaml` / `テスト株式会社_3件版.yaml`
- 出力例：`C:\Users\hseki\smarvee-decks\output\テスト株式会社\index.html`
- 引継ぎ：`J:\マイドライブ\AI_support_Projects\Smarvee営業資料\README.md`
