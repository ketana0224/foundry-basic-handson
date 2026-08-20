# Microsoft Foundry 初学者向け 180分ハンズオン（Lab 目次）

> 本ハンズオンは [final-report.md](report/20260811-1514-microsoft-foundry-handson/final-report.md) の「初学者向け180分ハンズオン」スケジュールを、当日そのまま進行できる **ステップ別の Lab** に分割したものです。各 Lab の末尾から次の Lab へ進めます。
>
> - **対象**: Microsoft Foundry を初めて触る受講者
> - **所要時間**: 180分（14ステップ + バッファ / Step 9（M365 公開）と Step 10（Hosted Agent）は発展・任意で必須パスには含まない）
> - **ゴール**: ポータル上で単一 Agent を作成し、Web Search と Foundry IQ で根拠付き回答を行い、トレース閲覧・手動評価・Rubric 評価で運用の入口を体験する
> - **使用サービス**: Microsoft Foundry ポータル / Foundry Agent Service（GA）
> - **使用しないもの**: Assistants API（2026-08-26 廃止予定のため不使用）、SDK、正式 Evaluation、MCP/A2A の自作、ネットワーク分離

---

## 事実 / 推測の区別について

各 Lab のうち、**製品仕様・公式ドキュメントに基づく記述は「事実」**、**当日の UI 操作フローやカリキュラム設計上の判断は「推測（設計判断）」** です。UI はアップデートで変わることがあるため、画面名やボタン名が一致しない場合は各 Lab の「UI が違うとき」と [Step 12 トラブルバッファ](lab/step12-troubleshoot.md) を参照してください。

- （事実）Foundry Agent Service は GA。出典: <https://devblogs.microsoft.com/foundry/foundry-agent-service-ga/>
- （事実）Assistants API は非推奨・2026-08-26 廃止予定。出典: <https://learn.microsoft.com/en-us/azure/foundry-classic/openai/concepts/assistants>
- （事実）Web Search ツールが提供されている。出典: <https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/web-search>
- （事実）Foundry IQ の概念。出典: <https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq>
- （事実）Observability / トレース。出典: <https://learn.microsoft.com/en-us/azure/foundry/concepts/observability>

---

## Lab 一覧（タイムテーブル）

| # | 時間 | 分 | Lab |
|---|---|---:|---|
| 0 | 0:00〜0:10 | 10 | [Step 0 — オリエンテーション](lab/step00-orientation.md) |
| 1 | 0:10〜0:30 | 20 | [Step 1 — サインイン・権限・プロジェクト確認](lab/step01-signin.md) |
| 2 | 0:30〜0:50 | 20 | [Step 2 — モデルとプレイグラウンド](lab/step02-model-playground.md) |
| 3 | 0:50〜1:15 | 25 | [Step 3 — 単一 Agent 作成（Web Search 既定有効）](lab/step03-create-agent.md) |
| — | 1:15〜1:25 | 10 | ☕ 休憩（Step 3 の末尾に記載） |
| 4 | 1:25〜1:35 | 10 | [Step 4 — MCP サーバーの有効化・接続（事前準備済）](lab/step04-mcp.md) |
| 5 | 1:35〜1:50 | 15 | [Step 5 — Foundry IQ の有効化・接続（事前準備済）](lab/step05-foundry-iq.md) |
| 6 | 1:50〜2:00 | 10 | [Step 6 — トレース閲覧](lab/step06-trace.md) |
| 7 | 2:00〜2:10 | 10 | [Step 7 — 手動評価](lab/step07-eval.md) |
| 8 | 2:10〜2:25 | 15 | [Step 8 — Rubric 評価](lab/step08-rubric.md) |
| 9 | 2:25〜2:40 | 15 | [Step 9 — Microsoft 365 / Teams への公開（発展・任意）](lab/step09-m365-publish.md) |
| 10 | 2:40〜2:55 | 15 | [Step 10 — Hosted Agent デプロイ（発展・任意）](lab/step10-hosted-agent.md) |
| 11 | 2:55〜3:05 | 10 | [Step 11 — クリーンアップ](lab/step11-cleanup.md) |
| 12 | 3:05〜3:20 | 15 | [Step 12 — トラブルバッファ](lab/step12-troubleshoot.md) |
| 13 | 3:20〜3:25 | 5 | [Step 13 — まとめ](lab/step13-summary.md) |

➡️ **最初の Lab へ: [Step 0 — オリエンテーション](lab/step00-orientation.md)**

---

## 講師向け 事前準備チェックリスト（当日までに完了）

> **注**: MCP サーバーと Foundry IQ の構築手順は本ハンズオンとは別に案内されます。ここでは「用意済みであること」を確認します。

- [ ] Azure サブスクリプションを用意し、受講者に必要な RBAC を付与済み（Foundry アカウント スコープに **Cognitive Services User** ロール。エージェント構築・モデル推論に必要な dataActions を含む）
- [ ] Microsoft Foundry プロジェクトを作成済み（受講者ごと or 共有）
- [ ] 利用するチャットモデル（例: GPT 系）をデプロイ済み、対象リージョンとクォータを確認済み
- [ ] **Foundry IQ を事前構築済み**、Agent への接続情報（ナレッジソース名など）を控えてある
- [ ] **MCP サーバーを事前構築済み**、Agent への接続情報（エンドポイント URL / 認証情報）を控えてある
- [ ] トレース閲覧用のリソースとサンプルデータを準備済み
- [ ] 受講者へ配布する「接続情報シート」（プロジェクト名・モデル名・IQ 接続名・MCP 接続情報）を用意

### 受講者が当日持参・確認するもの

- [ ] サインインに使う組織アカウント（Microsoft Entra ID）
- [ ] ブラウザ（Microsoft Edge または Google Chrome の最新版）
- [ ] 講師から配布された「接続情報シート」

---

## 付録

- 受講者成果物・前提・参照元は [Step 13 — まとめ](lab/step13-summary.md) の付録を参照してください。
- 主要参照元:
  - [What is Microsoft Foundry?](https://learn.microsoft.com/en-us/azure/foundry/what-is-foundry)
  - [Foundry Agent Service overview](https://learn.microsoft.com/en-us/azure/foundry/agents/overview)
  - [Web Search tool](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/web-search)
  - [MCP tool](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol)
  - [Foundry IQ](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq)
  - [Observability](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability)
