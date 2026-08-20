# Step 9 — Microsoft 365 / Teams への公開（発展・任意）（15分 / 2:25〜2:40）

⬅️ 前へ: [Step 8 — Rubric 評価](step08-rubric.md) ｜ 🏠 [目次](README.md) ｜ ➡️ 次へ: [Step 10 — Hosted Agent デプロイ](step10-hosted-agent.md)

---

> **この Step は「発展・任意」です。** これまで作った Agent を、Microsoft 365 Copilot や Microsoft Teams の中から呼び出せるように**公開（Publish）**します。追加の RBAC（Azure Bot Service 作成権限）や、組織公開の場合は**管理者承認**が必要になるため、権限が足りない場合は読み物として理解するだけでも構いません。時間が押している場合はこの Step を飛ばして [Step 10 — Hosted Agent デプロイ](step10-hosted-agent.md) または [Step 11 — クリーンアップ](step11-cleanup.md) に進んでください。**必須パスにはこの Step を含みません。**

## 目的
Step 3〜8 では、ポータル上で Agent を作り、ツール・ナレッジ・トレース・評価まで体験しました。この Step では、その Agent を **Microsoft 365 Copilot と Microsoft Teams に公開**し、利用者が**普段使っているツールの中から Agent を発見して会話できる**状態にするフローを体験します。

公開されるのは Agent の**安定エンドポイント（stable endpoint）**です。利用者は常に一貫した Agent と対話し、開発者側は裏で新しいバージョンをロールアウトできます。

> **（事実）** ポータルからの公開は Foundry の Microsoft 365 公開 API を呼び出し、Teams アプリ パッケージを自動生成します。出典: <https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot>

## 公開すると何が起きるか
公開時、Foundry は次を自動で行います。

1. 表示名・説明・バージョンなど入力プロパティを検証する。
2. **Teams アプリ マニフェスト**を `.zip` パッケージとしてコンパイルする。
3. マニフェストを **Microsoft 365 Copilot / Teams のエージェント カタログ**に送信する。
4. Agent が Microsoft 365 / Teams とメッセージ交換するための **`activity` プロトコル**を有効化する。
5. 呼び出し権限を制御する**認可スキーム**（`BotServiceRbac` または `BotServiceTenant`）を、選んだ公開範囲に応じて有効化する。

> **（事実）** 公開時に上記 1〜5 が実行されます。出典: <https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot#what-happens-when-you-publish>

## Entra Agent ID・Microsoft Agent 365 との関係（特記）
この Step の「Teams / M365 Copilot への公開」は、単に UI に Agent を出すだけでなく、**エージェントの ID（Entra Agent ID）** と **組織の統制プレーン（Microsoft Agent 365）** の 2 つと密接に関係します。ハンズオンの本筋（Bot Service 経由の配布）とは**別レイヤー**なので、ここで関係を整理しておきます。

### 1. Entra Agent ID（Microsoft Entra Agent ID）＝ エージェント固有の ID
- Foundry は Entra Agent ID を**エージェントのライフサイクルを通じて自動でプロビジョニング・管理**します。プロジェクトで最初の Agent を作った時点で、**既定の「agent identity blueprint」と「agent identity」**が作られ、プロジェクト内の未公開エージェントは**この共有 ID** で認証します。
- **公開（Agent Application 化）すると、そのエージェント専用の agent identity blueprint と agent identity が新たに作られ**、以降はプロジェクト共有 ID ではなく**エージェント固有の Entra Agent ID** で認証します。これにより、MCP / A2A ツールへの接続やリソース アクセスを、資格情報を共有せずに**エージェント単位で認可・監査**できます。
- つまり **Step 4 の MCP 接続や Step 5 の Foundry IQ が「誰の権限で動くか」を決めているのがこの Entra Agent ID** です。公開はこの ID を「プロジェクト共有」から「エージェント専用」に昇格させる操作でもあります。

> **（事実）** Foundry は Entra Agent ID を自動作成・管理し、未公開エージェントはプロジェクト共有 ID、公開エージェントは専用 ID で認証します。出典: <https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity#foundry-integration> / <https://learn.microsoft.com/azure/foundry/agents/how-to/agent-applications#what-is-publishing>

### 2. Microsoft Agent 365 ＝ 組織全体のエージェント統制プレーン
- **Microsoft Agent 365** は、組織内のすべての AI エージェントを**棚卸し・保護・観測**するためのエンタープライズ統制プレーンです。エージェントを **Entra Agent ID を通じて第一級の Entra ID** として扱い、認証・認可・ライフサイクル ガバナンスを ID オブジェクトに直接適用します。
- 主な柱は **Registry（在庫）／ Access control（Entra ベースの認可・RBAC/ABAC・リスク連動の条件付きアクセス）／ Visualization（可視化）／ Interoperability（M365 データ連携）** です。

> **（事実）** Agent 365 は Entra Agent ID を基盤にした統制プレーンで、Registry・Access control・Visualization・Interoperability の柱を持ちます。出典: <https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration>

### 3. Foundry と Agent 365 のつながり方（2 経路）
Foundry のエージェントは、次の **2 通り**で Agent 365 とつながります。

| 経路 | 何が起きるか | この Step との関係 |
|---|---|---|
| **① 自動レジストリ同期（Registry sync）** | 組織が Agent 365 をサブスクライブしていれば、**Foundry で公開したエージェントは自動で Agent 365 レジストリに現れ**、IT 管理者が在庫・所有者・ガバナンスを一元管理できる。 | **この Step の公開がトリガー**。公開すると（サブスクライブ済み組織では）自動で統制対象に入る。 |
| **② Autopilot として公開（Autopilot publishing）** | **Hosted Agent** を「autopilot（ユーザーの代わりに自律動作するエージェント）」として公開すると、**専用の Entra Agent ID を受け取り**、管理者承認後に Agent 365 レジストリに出て Teams などに接続できる。 | **この Step とは別フロー**。[Step 10 — Hosted Agent デプロイ](step10-hosted-agent.md) 系の `azd` ベースの公開（`agent-365` サンプル）で行う。 |

> **（事実）** Foundry↔Agent 365 は「自動レジストリ同期」と「autopilot 公開」の 2 経路で接続します。Prompt / Hosted いずれも両経路をサポートします。出典: <https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration#how-foundry-integrates-with-agent-365>

### まとめ（3 者の役割分担）
- **この Step（Teams / M365 公開）** = 利用者が M365 Copilot / Teams から呼べるようにする**配布**（Azure Bot Service ＋ Teams マニフェスト）。
- **Entra Agent ID** = そのエージェントが「何の権限で動くか」を決める**ID・認可の土台**（公開でエージェント専用 ID に昇格）。
- **Agent 365** = 公開後のエージェントを組織横断で**棚卸し・条件付きアクセス・監査・ブロック**する**ガバナンス層**。

> **（注）** 「公開（Publish＝Agent Application 化）」で作られるのは **Entra Agent ID（agent identity / blueprint）** ですが、Foundry Agent Application 自体は **Entra の agent registry には登録されません**。組織の在庫としての可視化は、上表①の **Agent 365 レジストリ同期**が担います（両者は別物）。出典: <https://learn.microsoft.com/azure/foundry/agents/how-to/agent-applications#what-is-publishing>
>
> **（推測）** ハンズオン用テナントでは Agent 365 のサブスクライブ（M365 Copilot ライセンス等）が無い場合が多く、その場合レジストリ同期や条件付きアクセスは体験できません。ここでは「公開すると ID とガバナンスがこう連動する」という**概念の理解**までを到達目標とします。

## 事前準備
- Step 3〜5 で作成し、**動作確認済みの Agent**（`handson-agent-<自分の名前>`）があること。
- 公開したい **アクティブ バージョン**が選択されていること（既定は「常に最新を使用（Always use latest）」で可）。
- 次のロールを持っていること:
  - **Azure Bot Service 共同作成者（Azure Bot Service Contributor）**ロール。公開時に Azure Bot Service リソースを作成（`Microsoft.BotService/botServices/write`）・チャネル構成（`.../channels/write`）するために必要。**Foundry のロールだけでは付与されない**点に注意（より広い Contributor / Owner でも可）。
  - Foundry プロジェクト スコープの **Foundry User** ロール（Agent の作成・管理・公開）。
- サブスクリプションで **`Microsoft.BotService`** リソース プロバイダーが登録済みであること（初回のみ）:

  ```powershell
  az provider register --namespace Microsoft.BotService
  ```

> **（事実）** RBAC 要件（Azure Bot Service Contributor + Foundry User）と、公開前のバージョン確認が前提です。出典: <https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot#prerequisites>
>
> **（注）** Foundry のロール名は最近リネームされました（旧: Azure AI User など → 新: Foundry User など）。環境によっては旧名が表示されることがあります。

## 手順（ポータル UI）

### A. 公開ダイアログを開く
1. Foundry ポータルで、公開したい Agent（`handson-agent-<自分の名前>`）を開く。
2. 右上の **公開（Publish）** ボタンを押し、**Teams and Microsoft 365 Copilot** を選ぶ。
   - もしくは **詳細（Details）** タブ →**チャネル（Channels）** セクションの **Teams & Microsoft 365 Copilot** からでも開ける。
3. **Publish to Teams and Microsoft 365** ダイアログが開く。

### B. Azure Bot Service とメタデータを入力する
1. **Azure Bot Service** リソースが自動作成される（既存があれば読み取り専用で表示）。
2. 必須メタデータを入力する。

   | 項目 | 内容（このハンズオン） |
   |---|---|
   | **Name（表示名）** | `handson-agent-<自分の名前>`（エージェント ストアに表示される名前） |
   | **Publish version** | `1.0.0`（major.minor.patch の 3 パート） |
   | **Short description** | `根拠付きで社内・最新情報に答えるハンズオン用エージェント` |
   | **Description** | `Web Search と Foundry IQ を使って、出典付きで質問に回答します。` |
   | **Developer（Author）** | 自分の名前 / 組織名 |

3. （任意）**More** を展開すると、Developer website / Terms of use / Privacy statement（いずれも **HTTPS 必須**）を追加できる。ハンズオンでは省略可。

### C. 公開範囲（誰が使えるか）を選ぶ
**Choose who can use this agent** で範囲を選びます。

| 選択肢 | 挙動 | 管理者承認 | 向いている用途 |
|---|---|---|---|
| **Just you（自分だけ）** | 公開後すぐ利用可。エージェント ストアの **Your agents** に表示。リンク共有で他の人にも渡せる。 | 不要 | 個人テスト・少人数・パイロット |
| **People in your organization（組織全体）** | 承認申請が出る。Microsoft 365 管理者が承認・アクセス割り当てを行うと **Built by your org** に表示され、テナント全員が発見・利用できる。 | 必要 | 組織全体への配布・本番 |

> **（推奨）** ハンズオンでは **Just you** を選ぶと、管理者承認を待たずにすぐ結果を確認できます。**People in your organization** を選ぶと、[Microsoft 365 管理センター](https://admin.cloud.microsoft/?#/agents/all/requested) での承認待ちになります。

> **（事実）** Just you = `BotServiceRbac`（承認不要・自分のみ表示）、組織全体 = `BotServiceTenant`（管理者承認・Built by your org 表示）。出典: <https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot#publish-to-microsoft-365-and-teams>

### D. 公開する
1. **Publish** を押す。
2. **Publish successful** ダイアログが表示されれば成功。
3. （任意）マニフェストを配布前に調整したい場合は、**Download & customize** から Teams アプリ マニフェスト（`.zip`）を取得できる。

### E. 動作を確認する
1. **Just you** の場合: Microsoft 365 Copilot / Teams のエージェント ストアの **Your agents** に自分の Agent が表示される。開いて質問し、応答を確認する。
2. **組織全体**の場合: 管理者承認後に **Built by your org** に表示される。承認状況は [Microsoft 365 管理センター](https://admin.cloud.microsoft/?#/agents/all/requested) の **Requests** で確認できる。

## 時間が足りないとき（最小構成）
- 本 Step は**任意**です。公開まで行わず、**「公開すると Teams アプリ マニフェストが生成され、Bot Service 経由で M365 / Teams から呼べるようになる」**という仕組みを理解するだけでも目的は達成できます。
- 公開まで試す場合は、承認待ちの発生しない **Just you** を選ぶと最短で確認できます。

## UI が違うとき
- **公開（Publish）** ボタンが見つからない場合は、Agent の **詳細（Details）** タブ →**チャネル（Channels）** から **Teams & Microsoft 365 Copilot** を探す。
- 「Azure Bot Service を作成できません」等の権限エラーが出る場合は、**Azure Bot Service Contributor** ロールと **`Microsoft.BotService`** プロバイダー登録を確認する。
- **公開ボタンが表示されない / 使えない**場合、プロジェクトが**パブリック ネットワーク アクセスを無効化**している可能性がある（その場合は REST API 経由の公開が必要）。参考: <https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot-virtual-network>
- 解決しない場合は [Step 12 — トラブルバッファ](step12-troubleshoot.md) を参照。

## 完了チェック
- [ ] Microsoft 365 / Teams への公開で「何が起きるか」（マニフェスト生成・カテゴリ登録・activity 有効化・認可スキーム設定）を説明できる
- [ ] **Just you** と **People in your organization** の違い（承認要否・表示先）を説明できる
- [ ] **公開・Entra Agent ID・Agent 365** の役割分担（配布 / ID・認可の土台 / ガバナンス層）と、Foundry↔Agent 365 の 2 経路（レジストリ同期・autopilot 公開）を説明できる
- [ ]（任意）Agent を公開し、**Publish successful** を確認できた
- [ ]（任意）Microsoft 365 Copilot / Teams のエージェント ストアで自分の Agent を確認できた

---

⬅️ 前へ: [Step 8 — Rubric 評価](step08-rubric.md) ｜ 🏠 [目次](README.md) ｜ ➡️ 次へ: [Step 10 — Hosted Agent デプロイ](step10-hosted-agent.md)
