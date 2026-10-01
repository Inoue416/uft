# WebMCP 調査資料

調査日: 2026-10-02（日本時間）。本資料は公開仕様とブラウザ公式資料を確認した時点の情報であり、UFT への導入判断は [Markdown エディタ導入計画](./_plans/webmcp-markdown-editor.md) にまとめる。

## 概要

WebMCP（Web Model Context Protocol）は、Web アプリケーションの操作を AI エージェントから呼び出せる JavaScript のツールとして公開する API である。エージェントはツールの説明と入力スキーマを読み、ページ内の既存処理を実行する。ユーザーとエージェントが同じ画面・アプリケーション状態を使うことを想定している。[仕様](https://webmachinelearning.github.io/webmcp/#intro)

例えば Markdown エディタなら、文書一覧、本文読み取り、文書切り替え、文書作成、本文編集をツール化できる。これは UFT に対する適用案であり、WebMCP にエディタ専用 API があるという意味ではない。

ページを対応させる側と、ツールを発見・実行するエージェント側の両方が必要になる。WebMCP の登録だけで AI チャットやモデルがアプリに付属するわけではない。[Chromium の実験提案](https://groups.google.com/a/chromium.org/g/blink-dev/c/gmYffo5WOE8/m/OJxuQRP3AAAJ)

## 公開・標準化の状況

「開発者が試せる形では公開済み、ブラウザ共通の正式標準としては未完成」と捉える。

| 観点 | 調査時点の状況 | 一次資料 |
| --- | --- | --- |
| 仕様 | Web Machine Learning Community Group のドラフト。W3C Standard でも W3C Standards Track の仕様でもない | [仕様のステータス](https://webmachinelearning.github.io/webmcp/#sotd) |
| Chrome のローカル検証 | `chrome://flags/#enable-webmcp-testing` を有効にして再起動する方法が案内されている | [Chrome の開始手順](https://developer.chrome.com/docs/ai/webmcp#local-webmcp) |
| Chrome の実サイト検証 | Chrome 149 から Origin Trial が公開されている。期限付きの実験機能をサイトで検証する仕組み | [Origin Trial の告知](https://developer.chrome.com/blog/ai-webmcp-origin-trial) |
| Chrome の正式提供予定 | 実験提案の Estimated milestones には試験範囲 149–156、Shipping 157 が記載されている。予定値を正式提供済みの根拠にはしない | [Chromium の実験提案](https://groups.google.com/a/chromium.org/g/blink-dev/c/gmYffo5WOE8/m/OJxuQRP3AAAJ) |
| Edge | Origin Trial の登録ページが公開され、実験機能として提供されている | [Edge の登録ページ](https://developer.microsoft.com/en-us/microsoft-edge/origin-trials/trials/0b76fe60-b266-458e-a285-04e375c0c31a) |

Origin Trial は一般公開されたサイトでの試験に使えるが、恒久的な提供を保証しない。Edge の登録条件も試験終了後の提供を保証していない。期限・対象バージョン・API は変わり得るため、UFT の試験公開直前に登録画面と公式資料を再確認する。[Chrome の説明](https://developer.chrome.com/blog/ai-webmcp-origin-trial)、[Edge の条件](https://developer.microsoft.com/en-us/microsoft-edge/origin-trials/trials/0b76fe60-b266-458e-a285-04e375c0c31a)

本調査では Chrome と Edge の利用経路を確認した。他ブラウザや各 AI 製品について、対応済みとは仮定しない。

## API と実装方式

| 方式 | 公開する対象 | UFT への適性 |
| --- | --- | --- |
| Imperative API | JavaScript で名前・説明・入力スキーマ・実行関数を登録 | 文書モデルや保存処理を扱うため、エディタの主方式に適する |
| Declarative API | HTML の `<form>` に `toolname`、`tooldescription` などを付ける | 通常のフォームには適するが、CodeMirror の文書編集を直接表現するには追加処理が必要 |

両方式は公式に説明されている。UFT での適性評価は現在のコード構成からの判断である。[Imperative API](https://developer.chrome.com/docs/ai/webmcp/imperative-api)、[Declarative API](https://developer.chrome.com/docs/ai/webmcp/declarative-api)

現行の入り口は `document.modelContext`。`registerTool()` で登録し、登録時に渡した `AbortSignal` で解除できる。実行コールバックにもキャンセル用の signal が渡る。古い資料の `navigator.modelContext` や `provideContext()` を実装の基準にせず、対象ブラウザで現行 API を検証する。[公式 API 資料](https://developer.chrome.com/docs/ai/webmcp/imperative-api)

ツールから返す値は JSON として表現できるデータにする。ブラウザの `getTools()`／`executeTool()` は発見・実行側の API であり、UFT の登録処理にエージェント実装まで含める必要はない。[仕様の API](https://webmachinelearning.github.io/webmcp/#api)

## ページと通信の境界

WebMCP は開いたページの JavaScript を実行する仕組みであり、通常のリモート MCP サーバー接続とは利用経路が異なる。UFT のページ側対応だけなら、サーバーの MCP エンドポイントを新設せずに既存のローカル処理を公開できる。[仕様の導入説明](https://webmachinelearning.github.io/webmcp/#intro)

API は secure context を対象とする。Chrome はさらに origin isolation と `tools` Permissions Policy による制限を説明している。origin isolation は SQLite 用の cross-origin isolation と区別する。`document.domain` が有効な構成では利用できず、`tools` の既定は `self` である。[仕様の Document 拡張](https://webmachinelearning.github.io/webmcp/#extensions-to-document)、[Chrome の制限](https://developer.chrome.com/docs/ai/webmcp#security-and-permissions)

他オリジンへの公開は明示的な設定を必要とする。UFT の初期導入では `exposedTo` を指定せず、クロスオリジン iframe への委譲も行わない。[公式の公開範囲の説明](https://developer.chrome.com/docs/ai/webmcp/secure-tools#expose-your-tools-carefully)

**ブラウザ内保存と AI へのデータ提供は別の性質である。** UFT 側が外部通信しなくても、本文を受け取ったエージェントはモデル処理に利用する可能性がある。読み取り公開の説明では、文書がエージェントへ渡ることを明示する。Chrome の検証拡張でも、自然言語テストのプロンプトは既定で Gemini モデルへ送信される。[Inspector の案内](https://developer.chrome.com/docs/ai/webmcp#imitate-agent-chat-with-the-inspector-extension)

## ツールの設計上の注意

- 入力スキーマに加え、実行時にも型・対象文書・サイズを検証する。ツールの登録と解除はページの利用可能な状態に合わせる。[ベストプラクティス](https://developer.chrome.com/docs/ai/webmcp/best-practices)
- 読み取りは `readOnlyHint`、ユーザー作成の本文などを含む出力は `untrustedContentHint` を付ける。これらはヒントであり、アプリ側の入力検証やアクセス範囲の制限を代替しない。[セキュリティ資料](https://developer.chrome.com/docs/ai/webmcp/secure-tools)
- 重要な変更には `consequentialHint` を検討する。確認が必要な操作を実装する際は、ブラウザのヒントだけに依存せず、アプリでも実行前確認を設ける。[セキュリティ資料](https://developer.chrome.com/docs/ai/webmcp/secure-tools)
- 長文は明示した範囲で取得し、切り詰めや続きを示す。Chrome は説明・出力の文字数を抑えることを推奨しているが、推奨値を仕様上の一律制限とは扱わない。[文字数に関する案内](https://developer.chrome.com/docs/ai/webmcp/secure-tools#set-character-budgets)

古い記事にある確認 API をそのまま採用しない。確認した現行ドラフトの IDL に `requestUserInteraction()` はなく、Chrome のセキュリティ資料にはこの名前への言及が残っている。UFT の確認処理はアプリ自身のダイアログで設計し、実装時に仕様との差分を再確認する。[現行 IDL](https://webmachinelearning.github.io/webmcp/#idl-index)、[Chrome の資料](https://developer.chrome.com/docs/ai/webmcp/secure-tools#next-steps)

## 検証方法と UFT の導入判断

Chrome の実験フラグと Model Context Tool Inspector を使い、登録一覧・スキーマ・呼び出し結果を確認する。自然言語での操作試験と、手動で引数を指定する試験を分ける。ヘッドレス環境だけでブラウザの実対応を証明しない。[公式の検証方法と制約](https://developer.chrome.com/docs/ai/webmcp)

UFT は既存の文書モデルと操作処理を公開できるため、試験導入が可能と判断する。まず読み取りと文書切り替えを検証し、編集は保存成功・競合検出・Undo を確認してから公開する。API がないブラウザでも通常の編集を続けられる構成にする。具体的なツール契約、実装順序、受け入れ条件は [導入計画](./_plans/webmcp-markdown-editor.md) を参照する。
