---
title: "LINE WORKSの引き継ぎチャットをAIが自動でGoogle Docsに転記する仕組みを作った"
emoji: "📋"
type: "tech"
topics: ["lineworks", "googleworkspace", "playwright", "python", "automation"]
published: false
---

## はじめに

複数拠点を持つサービス業では、拠点間の引き継ぎ情報管理が地味に大変です。

うちのケースは「LINE WORKSで各拠点スタッフが引き継ぎ事項を投稿 → 担当者がドキュメントに手動でコピペ」というフローでした。これを毎日やるのは手間だし、コピペミスも起きる。

そこで **Playwright常駐サーバー + Python + Google Docs API** を組み合わせて、引き継ぎ情報の自動転記を実装しました。

この記事ではその構成と実装のポイントを解説します。

## システム構成

```
LINE WORKS（チャット）
    ↓ Playwright（ブラウザ自動化）
メッセージ抽出（テキスト＋画像OCR）
    ↓ Python
Google Docs API（拠点ごとのドキュメントに追記）
    ↓ launchd（macOS）
1日2回 定時自動実行（8:00 / 20:00）
```

### 使用技術

- **Playwright（Node.js）**: LINE WORKSはAPIが限定的なため、ブラウザ自動化で対応
- **Python**: メッセージの解析・Google Docs API呼び出し
- **macOS Vision OCR（Swift）**: 画像メッセージのテキスト抽出
- **launchd**: macOSのサービス管理。常駐＋自動再起動

## なぜPlaywrightを常駐させるのか

LINE WORKSは公式APIでのメッセージ取得に制限があります。そのため、ブラウザ（Chromium）でLINE WORKSにログインした状態を維持し、HTTPサーバーとして常駐させる構成にしました。

```
http://127.0.0.1:18800/status   → 稼働確認
http://127.0.0.1:18800/extract  → メッセージ取得
http://127.0.0.1:18800/download-image?url=... → 画像DL（認証済みセッションで）
```

Playwrightサーバーは `launchd` で管理し、Mac再起動後も自動起動・クラッシュ時も自動再起動します。

```xml
<!-- com.myapp.lineworks-pw.plist -->
<key>RunAtLoad</key><true/>
<key>KeepAlive</key><true/>
```

セッションのCookieは永続化しており、再ログイン不要で長期運用できます。

## メッセージ抽出の実装

Playwrightで各ルーム（拠点チャンネル）にアクセスし、DOMからメッセージを取得します。

```javascript
// extract.js（抜粋）
const messages = await page.evaluate(() => {
  const results = [];
  document.querySelectorAll('.msg_wrap').forEach(el => {
    const text = el.querySelector('.msg_box')?.innerText?.trim() || '';
    const imgEl = el.querySelector('.msg_box img.img_msg');
    const hasImg = !!imgEl;
    const imgUrl = imgEl?.src || imgEl?.getAttribute('data-src') || '';

    if (text || hasImg) {
      results.push({
        name: currentName,
        time: timeEl?.innerText?.trim() || '',
        text,
        img: hasImg && !text,
        imgUrl: hasImg ? imgUrl : ''
      });
    }
  });
  return results;
});
```

重複取得を避けるため、最終処理日時をJSONファイルで管理し、差分のみGoogle Docsに追記します。

## 画像メッセージのOCR

LINE WORKSには手書きメモや写真を貼り付けるケースがあります。これをテキストに変換するため、**macOS標準のVision Framework**をSwiftで呼び出します。

```swift
// ocr-vision.swift（抜粋）
import Vision
import Foundation

let imageURL = URL(fileURLWithPath: CommandLine.arguments[1])
let requestHandler = VNImageRequestHandler(url: imageURL)
let request = VNRecognizeTextRequest { request, _ in
    guard let observations = request.results as? [VNRecognizedTextObservation] else { return }
    let text = observations.compactMap { $0.topCandidates(1).first?.string }.joined(separator: "\n")
    print(text)
}
request.recognitionLanguages = ["ja", "en"]
try? requestHandler.perform([request])
```

追加ライブラリ不要・macOSに標準搭載なのでインストール不要です。

Playwrightの認証済みセッションを使って画像をダウンロードし、Swiftに渡してOCRします。

```python
def download_and_ocr(img_url: str) -> str:
    # Playwrightサーバー経由でDL（認証が必要な画像URL）
    resp = requests.get(
        f"http://127.0.0.1:18800/download-image?url={urllib.parse.quote(img_url)}"
    )
    with tempfile.NamedTemporaryFile(suffix=".png", delete=False) as f:
        f.write(resp.content)
        tmp_path = f.name

    result = subprocess.run(
        ["swift", "scripts/ocr-vision.swift", tmp_path, "ja"],
        capture_output=True, text=True, timeout=30
    )
    return result.stdout.strip()
```

## 拠点ごとのドキュメント振り分け

メッセージ内のキーワードや送信元チャンネルを見て、対応するGoogle Docsドキュメントに追記します。

```python
ROOM_TO_DOC = {
    "拠点A": "1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "拠点B": "1yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy",
    "拠点C": "1zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz",
}
```

Google Docs APIへの書き込みはサービスアカウントで認証します。OAuth2より管理がシンプルです。

```python
from googleapiclient.discovery import build
from google.oauth2 import service_account

creds = service_account.Credentials.from_service_account_file(
    "config/service-account.json",
    scopes=["https://www.googleapis.com/auth/documents"]
)
service = build("docs", "v1", credentials=creds)

# ドキュメントの末尾に追記
service.documents().batchUpdate(
    documentId=doc_id,
    body={"requests": [{"insertText": {"location": {"index": end_index}, "text": text}}]}
).execute()
```

## 定時自動実行（launchd）

macOSのlaunchdでcron的な定時実行を設定します。Pythonスクリプトを直接呼び出すのではなく、AIエージェント（OpenClaw）経由で実行しているため、エラー時の通知なども柔軟に対応できます。

```
8:00 JST  → Playwrightで前夜〜朝のメッセージを抽出 → Google Docs更新
20:00 JST → Playwrightで日中のメッセージを抽出   → Google Docs更新
```

## 運用してみて

- 1日平均50〜80件のメッセージを処理
- 画像OCR精度はmacOS Vision Frameworkで十分（日本語対応良好）
- セッション維持はCookieで安定運用中（数ヶ月無再認証）
- エラー時はTelegramに通知が来る構成にしている

手動コピペ作業がゼロになり、転記ミスも解消されました。

## 応用

同じ構成は、記録管理が重要な業界全般に応用できます。

- **医療・介護**: スタッフ間の申し送り・ケア記録の自動アーカイブ
- **警備・施設管理**: 巡回日誌・異常記録の自動集約
- **飲食チェーン**: 各店舗の日報・衛生管理記録の一元管理
- **物流・倉庫**: 作業日報・引き継ぎ情報の集約

LINE WORKSに限らず、Playwrightが動くWebアプリであれば同様の構成が組めます。

## まとめ

- LINE WORKSのAPI制限をPlaywright常駐サーバーで回避
- macOS Vision OCRで画像メッセージにも対応
- Google Docs APIで拠点別ドキュメントに自動追記
- launchdで1日2回の定時実行

コードベースはNode.js（Playwrightサーバー）＋Python（処理ロジック）のハイブリッド構成ですが、役割が明確に分かれているので意外とシンプルです。

似たような「チャット→ドキュメント自動化」を検討している方の参考になれば。
