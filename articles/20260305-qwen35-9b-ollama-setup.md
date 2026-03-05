---
title: "Qwen3.5-9BをMac miniに入れてAIエージェントのローカルLLMにした話"
emoji: "🤖"
type: "tech"
topics: ["ollama", "llm", "qwen", "localai", "ai"]
published: true
---

## はじめに

AIエージェントを自宅サーバー（Mac mini M4、16GB）で動かしている。普段のメインLLMはClaudeだが、サブエージェントや軽量タスクにはローカルLLMを使いたい。

これまで使っていたのは `qwen3:8b`（旧世代）。ここに **Qwen3.5-9B** が登場したので、16GB Mac miniに乗せるまでの調査・作業をまとめる。

---

## Qwen3.5とは

**Qwen3.5** はAlibaba Cloudが2025年末にリリースしたQwenシリーズの最新世代。

主な改善点：

| 項目 | Qwen3（旧） | Qwen3.5 |
| --- | --- | --- |
| マルチモーダル | 一部対応 | ✅ 全サイズ対応 |
| コンテキスト長 | 32K〜128K | **262K** |
| 性能 | Qwen3-30B相当 | **9Bで旧30B超え** |
| 量子化 | Q4〜Q8 | Q4〜Q8 |

特筆すべきは「**9BがQwen3-30Bを超える**」という点。パラメータ数が1/3以下なのに性能が上回っているため、VRAM制約のある環境には非常に魅力的だ。

---

## Mac mini 16GBで動かせるサイズの検討

Qwen3.5のラインアップ：

```
Qwen3.5-9B   → Q4_K_M: ~6.6GB  ← ✅ 快適に動く
Qwen3.5-27B  → Q4:    ~14GB   ← ギリギリ、ヘッドルームなし
Qwen3.5-35B-A3B → Q4: ~20GB  ← ❌ 16GBには乗らない
```

27Bは一応ロードできるが、OSとOllamaのオーバーヘッドを考えると実質余裕ゼロ。推論中にスワップが発生して遅くなる可能性が高い。

**結論：Qwen3.5-9B（Q4_K_M、6.6GB）が最適解。**

---

## セットアップ手順

### 1. Ollamaのアップデート

まずOllamaを最新版にする。古いバージョンだと新モデルのmanifestが認識されない場合がある。

```bash
brew upgrade ollama
```

0.15.6 → 0.17.6 にアップデートされた。

### 2. モデルのpull

```bash
ollama pull qwen3.5:9b
```

6.6GBなので、光回線なら5〜10分程度。完了するとこのような出力になる：

```
pulling manifest
pulling dec52a44569a... 6.6 GB
pulling 7339fa418c9a... 11 KB
pulling 9371364b27a5...  65 B
pulling be595b49fe22... 475 B
verifying sha256 digest
writing manifest
success
```

### 3. 登録確認

```bash
ollama list
```

```
NAME          ID              SIZE      MODIFIED
qwen3.5:9b    6488c96fa5fa    6.6 GB    2 minutes ago
qwen3:8b      500a1f067a9f    5.2 GB    2 weeks ago
```

---

## ハマったポイント：pullしたのに一覧に出ない

実は一度目の `ollama pull` では `success` と表示されたにもかかわらず、`ollama list` に出てこないという問題が起きた。

### 原因

`~/.ollama/models/blobs/` を確認すると：

```
sha256-dec52a44569a...-partial   ← 6.6GBのblob（本体）
sha256-dec52a44569a...-partial-0 ← 67バイトのスタブ
sha256-dec52a44569a...-partial-1 ← 67バイトのスタブ
...（17個）
```

**pull中にOllamaデーモンが再起動されていた**ため、blobファイルが `-partial` サフィックスのまま残りmanifestが書き込まれなかった。データ自体は存在しているが、Ollamaがモデルとして認識できない状態。

### 解決方法

再度 `ollama pull qwen3.5:9b` を実行するだけ。

Ollamaは既存のblobファイルをSHA256で検証し、内容が正しければそのままmanifestを書き込む。**6.6GBの再ダウンロードは発生しない。**

```bash
ollama pull qwen3.5:9b
# → pulling dec52a44569a... 6.6 GB ✓（キャッシュ再利用）
# → verifying sha256 digest
# → writing manifest
# → success
```

これで `ollama list` に正常に表示されるようになった。

---

## OpenClaw（AIエージェント）との連携

自宅の自動化基盤にはOpenClawを使っている。OllamaのモデルをOpenClawで使うには、API keyの設定が必要。

```bash
# launchd plistに環境変数を追記
OLLAMA_API_KEY=ollama-local
```

OpenClawはこの環境変数を検出してOllamaプロバイダーを自動で有効化する。

設定後はGatewayを再起動：

```bash
launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl load  ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

サブエージェント呼び出し時に `model: "ollama/qwen3.5:9b"` を指定すれば、ローカルLLMで動く。

---

## 使い分けの方針

```
メインLLM（Claude Sonnet）    → 判断・文章生成・複雑なタスク
Qwen3.5-9B（ローカル）        → 軽量タスク・分類・要約・サブエージェント
```

ローカルLLMのメリット：

- **レイテンシ低い**（ネットワーク往復なし）
- **API費用ゼロ**（電力代のみ）
- **プライバシー**（データが外に出ない）

デメリット：

- 推論速度はClaude Sonnetより遅い（Mac mini M4でトークン/秒は限られる）
- 大規模なコンテキスト処理には向かない

---

## まとめ

- Qwen3.5-9BはQ4_K_Mで6.6GB、16GB Mac miniに余裕で乗る
- 性能は旧Qwen3-30B超え、マルチモーダル、262Kコンテキスト
- pull後に一覧に出ない場合は再pullで解決（再DL不要）
- `OLLAMA_API_KEY` 環境変数でOpenClaw等のエージェントと連携可能

ローカルLLMを自宅の自動化基盤に組み込む際の参考になれば。
