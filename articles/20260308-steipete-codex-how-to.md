---
title: "OpenClaw作者のCodexアプリ活用法——CLIからアプリへ移行した理由と設定のコツ"
emoji: "🦞"
type: "tech"
topics: ["Codex", "OpenAI", "AIエージェント", "開発ツール", "ChatGPT"]
published: true
---

OpenClawの作者として知られるPeter Steinberger（[@steipete](https://x.com/steipete)）がOpenAI Codexの使い方について投稿し、それを[@nukonuko](https://x.com/nukonuko)さんがまとめていました。

個人的に「これは参考にしたい」と思ったのでZennにも残しておきます。

---

## steipeteとは

iOSエンジニア出身のビルダーで、AIパーソナルアシスタントフレームワーク「OpenClaw」の作者です。2026年初頭にOpenAIへ参加し、AIエージェントを誰でも使えるようにすることをミッションとしています。

「OpenClawはGitHubスターでVSCodeを抜いた」とも言われており、AI開発ツールの最前線にいる人物です。

そのsteipeteが日常的にどうCodexを使っているか——というのはそのまま実践的なノウハウになります。

---

## まとめ：steipeteのCodex活用法

### 1. Codex CLI からアプリへ移行した

> "we reached the point where app > cli"

CLIでの操作から、デスクトップアプリ（Codex App）に完全移行しています。理由は**速度向上と、ウィンドウ数の削減**。アプリになったことで複数のタスクを1つのGUI内で並行管理できるようになりました。

Codex Appの主な機能：
- 複数プロジェクトを並列実行
- Built-in Git（diff・commit・revertがアプリ内完結）
- Git worktreeで並行タスクを分離
- スレッドごとに独立したターミナル
- Automations（バックグラウンド自動実行）
- MCPサポート

### 2. GPT-5.4を medium か high で使っている

**推奨設定：medium**
ほとんどのケースでmediumで十分とのこと。

**xhighは不要：**
最高品質設定（xhigh）を使っても体感差が少ない割にコスト・時間がかかるため、推奨していません。

```
日常コーディング → medium
複雑なアーキテクチャ相談・設計 → high
xhigh → 基本的に不要
```

### 3. Codex にエディタ・ファイルツリー機能は搭載されない

Codex Appはあくまで**AIエージェントのコントロールパネル**という設計思想です。VSCodeやCursorのような統合開発環境を目指していません。

コードを書く場所はIDEのまま、Codexはエージェントとして「タスクを渡す・結果を受け取る」役割に徹しています。

### 4. Context Length を1Mに増やすのはまだ非推奨

Codexでは最大1Mトークンまでコンテキストを拡張できますが、steipeteは**コーディング用途では推奨しない**としています。

理由として考えられるのは：
- 大きすぎるコンテキストはモデルの集中力（attention）を分散させる
- 速度低下・コスト増加に対してリターンが薄い
- 通常のタスクはデフォルト設定で十分

### 5. Linux版 Codex アプリはもうすぐ

現在はmacOS（Apple Silicon）とWindowsに対応。Linux版については「まともなポートにこだわっているので、もう少し時間がかかる」とのことです。メーリングリストへの登録で通知を受け取れます。

→ [Codex app sign up | OpenAI](https://openai.com/codex/app/sign-up)

---

## 個人的な所感

「アプリがCLIを超えた」というのは地味に大きな転換点だと思います。

CLIは柔軟で軽量ですが、複数タスクの並行管理・Gitとの統合・結果の可視化といった点でGUIに劣ります。Codex Appがこれらをカバーしたことで、エンジニアの日常ツールとして現実的になってきた印象があります。

**medium設定で十分**というのも重要な知見です。最高品質設定を使いたくなるのは人情ですが、日常的なコーディングでは費用対効果が薄い。steipeteのような使い込んでいる人がそう言うのは、実感として信頼できます。

---

## 参考

- [@steipete のポスト](https://x.com/steipete/status/2030330442520445371)
- [@nukonuko によるまとめ](https://x.com/nukonuko/status/2030533816465682810)
- [Codex App 公式ドキュメント](https://developers.openai.com/codex/app/)
- [steipeteのブログ：OpenClaw, OpenAI and the future](https://steipete.me/posts/2026/openclaw)

---

*本記事は公開情報をもとにまとめたものです。*
