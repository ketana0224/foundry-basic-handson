# Step 10 — Hosted Agent デプロイ（発展・任意）（15分 / 2:40〜2:55）

⬅️ 前へ: [Step 9 — Microsoft 365 / Teams への公開](step09-m365-publish.md) ｜ 🏠 [目次](README.md) ｜ ➡️ 次へ: [Step 11 — クリーンアップ](step11-cleanup.md)

---

> **この Step は「発展・任意」です。** ここまでのハンズオンは **ポータル UI のみ** で完結しますが、本 Step だけは **VS Code とコード（azd/SDK）** を使う発展課題です。時間が押している場合や、まずはノーコードの流れを一通り体験したい場合は、この Step を飛ばして [Step 11 — クリーンアップ](step11-cleanup.md) に進んで構いません。**必須パス（約180分）はこの Step を含みません。**

## 目的
Step 3〜8 では、ポータル上で Agent を作り、ツール・ナレッジを接続し、トレースと評価まで体験しました。この Step では一歩進んで、**コードで書いた Agent を「Hosted Agent」として Foundry のマネージド ホストにデプロイ** し、Foundry がスケール・認証・監視まで面倒を見る仕組みを体験します。

「ポータルで作る Agent」と「コードで書いて Hosted としてデプロイする Agent」の違いを体感するのがねらいです。

> **（事実）** Foundry Hosted Agents は一般提供（GA）です。出典: <https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents>

## Hosted Agent とは
- **Hosted Agent** = 自分で書いた Agent コードを、Foundry の**マネージド ランタイム**上にデプロイして動かす仕組み。
- サーバーの用意・スケール・認証・監視は Foundry 側が担うため、**インフラ構築なしに** コード資産を公開できる。
- Foundry がデプロイ時に環境変数を自動注入するので、コードにエンドポイントやキーを直書きしなくてよい。
  | 注入される環境変数 | 用途 |
  |---|---|
  | `FOUNDRY_PROJECT_ENDPOINT` | 所属する Foundry プロジェクトのエンドポイント。 |
  | `AZURE_AI_MODEL_DEPLOYMENT_NAME` | 使用するモデル デプロイ名。 |
  | `APPLICATIONINSIGHTS_CONNECTION_STRING` | トレース/メトリクス送信先（Application Insights）。 |

> **（事実）** デプロイ時に上記の環境変数が Foundry から注入されます。出典: <https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent>

## 事前準備
- **VS Code** と **Foundry Toolkit 拡張機能**（プレリリース チャネル）。取得: <https://aka.ms/foundrytk>
- **Azure Developer CLI（azd）1.27.1 以上**。
- Foundry プロジェクトに対して **Contributor** ロールを持っていること。
- **Azure へのサインイン**（未サインインの場合）。VS Code の Foundry Toolkit デプロイ・`azd`・`az` はいずれも Azure 認証が必要です。まず次でサインインしておきます（ブラウザーが開きます）:

  ```powershell
  az login
  ```

  > **（注）** 複数のテナントやサブスクに所属する場合は `az account set --subscription "<サブスク名 or ID>"` で対象を揃えておくと確実です。

- リソース プロバイダーの登録（初回のみ。`az login` 済みであること）:

  ```powershell
  az provider register --namespace Microsoft.CognitiveServices
  ```

> **（推測）** 拡張機能はプレビュー段階のため、メニュー名やコマンド名は更新されることがあります。表示が異なる場合は近い名称を探してください。

## 手順（VS Code Foundry Toolkit を使う推奨フロー）

### A. Agent プロジェクトのひな形を用意する
1. VS Code で作業用フォルダーを開く。
2. コマンド パレット（`Ctrl+Shift+P`）を開き、**`Foundry Toolkit: Create new Hosted Agent`** を実行する（`azd ai agent init` 相当）。
3. **Agent Details** でテンプレートを選ぶ。このハンズオンでは **Basic Hosted Agent**（Python / Agent Framework / Responses）を選ぶ。

   > **（なぜこれか）** 追加のツールや外部接続を必要としない最小構成で、以降の手順（`main.py` / `POST /responses` / `Deployment Method=Code` / `Package Mode=Remote`）とそのまま一致するため。MCP Toolbox や Azure AI Search RAG などのテンプレートは追加リソースや接続設定が必要になるので、任意の発展 Step には過剰です。

4. **Next** → **Create** に進み、**Create Hosted Agent from Sample** 画面で次のとおり設定してひな形を生成する。

   | 項目 | 設定値（このハンズオン） | 補足 |
   |---|---|---|
   | **Workspace Folder** | 作業用フォルダー（例: `c:\GitHub\Foundry_basic_Handson`） | この直下にプロジェクト フォルダーが作成される。 |
   | **Folder Name** | 自動入力のままでOK（例: `my-agent-xxxx`） | プロジェクト フォルダー名。任意で変更可。 |
   | **Environment Setup** | **Skip for now** | まずコードだけ生成する。モデル接続（`.env`）は後で設定する。デプロイ時に Foundry が環境変数を自動注入するため、ここでは接続不要。 |

   > **（補足）** モデル接続を今すぐ構成したい場合は **Setup with Microsoft Foundry** を選ぶと、Foundry のエンドポイントへの接続を今のうちに設定できます。ハンズオンでは Hosted としてデプロイするため **Skip for now** で十分です。

5. **Create** を押すと、`Project will be created at:` に表示されたパスにひな形が生成される。
6. 生成された `main.py`（Agent 本体）を確認する。ローカルで動作を確認したい場合は、次の手順で仮想環境（`.venv`）を作って依存をインストールする。

   > **（注意）** `main.py` はプロジェクト直下ではなく **`src\agent-framework-agent-basic-responses\`** の中にあります。まずそのフォルダーへ移動してください。グローバルの Python に古い `agent_framework` が入っていると `ImportError: cannot import name 'Agent'` が出るため、**必ず `.venv` を作って隔離**します。

   ```powershell
   # 1) main.py のあるフォルダーへ移動（フォルダー名は生成時のものに置き換え）
   cd src\agent-framework-agent-basic-responses

   # 2) 仮想環境を作成して有効化
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   # ↑ 実行ポリシーで弾かれる場合は先に次を実行:
   #   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

   # 3) pip を更新し、依存をインストール（プレリリース版が必要なので --pre 必須）
   python -m pip install --upgrade pip
   python -m pip install --pre -r requirements.txt
   ```

   > **（注）** `pip install` はパッケージのダウンロードとビルドに **数分程度**かかることがあります。途中で止まったように見えても、そのまま完了を待ってください。

   > プロンプト先頭に `(.venv)` が付いていれば有効化成功です。以降このターミナルで `python` を実行すると `.venv` 側が使われます。

7. `.env` に接続情報を設定する（ローカル起動時のみ必要。Hosted デプロイ時は Foundry が自動注入するため不要）。

   ```ini
   FOUNDRY_PROJECT_ENDPOINT=<Foundry プロジェクトのエンドポイント>
   AZURE_AI_MODEL_DEPLOYMENT_NAME=<Step 2 で使ったモデル デプロイ名>
   ```

   **各値を Foundry ポータルで確認する手順:**

   - **`FOUNDRY_PROJECT_ENDPOINT`**（プロジェクト エンドポイント）
     1. Foundry ポータル（<https://ai.azure.com>）で対象プロジェクトを開く。
     2. **ホーム** 画面の下部にある **「プロジェクト エンドポイント」** の欄を確認する（`https://<リソース名>.services.ai.azure.com/...` の形式）。
     3. 右側のコピー アイコンで値をコピーし、`.env` に貼り付ける。
     > **（注）** 隣にある **「Azure OpenAI エンドポイント」**（`...openai.azure.com`）ではありません。必ず **プロジェクト エンドポイント**（`services.ai.azure.com`）を使ってください。

   - **`AZURE_AI_MODEL_DEPLOYMENT_NAME`**（モデル デプロイ名）
     1. 左メニューの **「デプロイ」** を開く。
     2. **「デプロイ済みモデル」** タブの一覧から、使いたいモデルの **「名前」** 列の値をそのまま使う（例: `gpt-4.1-mini`）。
     3. Step 2 のプレイグラウンドで使ったデプロイ名と合わせると確実です。
     > **（注）** 「モデル」列ではなく **「名前」列（デプロイ名）** を使います。デプロイの状態が **Succeeded** のものを選んでください。

   `DefaultAzureCredential` を使うため、事前に `az login` でサインインしておく。

8. ローカルで起動する。

   ```powershell
   python main.py
   ```

   起動後、別ターミナルから疎通確認:

   ```powershell
   curl -X POST http://localhost:8088/responses -H "Content-Type: application/json" -d "{""input"":""こんにちは""}"
   ```

### B. Hosted Agent としてデプロイする

> **⚠️ 本ハンズオン環境について:** 本ハンズオン環境は**セキュリティ制約（社内ネットワーク境界 / プライベート DNS）**により、**VS Code（Foundry Toolkit）からの Hosted Agent デプロイは実行できません**（デプロイ先の azureml ワークスペース ホスト `*.api.azureml.ms` が解決できず `fetch failed` になります）。そのため、本環境では **CLI（`azd`）でデプロイ**してください。手順は本節末尾の [参考：azd（CLI）で進める場合](#参考azdcliで進める場合) を参照。以下の VS Code ウィザード手順は、制約のない一般的な環境での参考手順です。

> **前提:** デプロイは Azure 上で実行されるため、**Azure にサインイン済み**である必要があります（事前準備の `az login`、または VS Code 左側の Azure サインイン）。未サインインだとウィザードのプロジェクト選択やデプロイでエラーになります。

1. コマンド パレットで **`Foundry Toolkit: Deploy Hosted Agent`** を実行する。**Deploy Hosted Agent** ウィザード（**① Foundry Project Setup → ② Basics → ③ Review + Deploy** の3ステップ）が開く。

2. **① Foundry Project Setup**（デプロイ先のプロジェクトを選ぶ）

   > **注意:** このステップは**初回のみ**表示されます。一度プロジェクトを選ぶと既定プロジェクトとして記憶され、次回以降は ② Basics から始まります。**デプロイ先を選び直したい／リセットしたい**場合は、コマンド パレットで **`Foundry Toolkit: Clear Default Project`** を実行してから、もう一度 `Deploy Hosted Agent` を実行してください。

   | 項目 | 選択値（このハンズオン） | 補足 |
   |---|---|---|
   | **Subscription** | 自分のサブスクリプション（例: `ME-M365CPI…-ketana-1`） | Agent をデプロイするサブスク。 |
   | **Project Name** | ハンズオンで使っている Foundry プロジェクト | 既存プロジェクトを選ぶ。無ければ **Create new** で新規作成。 |

   選んだら **Next**。

3. **② Basics**（デプロイ方式と Agent 名）

   > **（画面構成）** 初回のプロジェクト選択が済むと、ウィザードのタブ見出しは **① Basics → ② Review + Deploy** の2枚になります。この **Basics** 画面では、次の3つを設定します（既定値のままでOK）。

   | 項目 | 選択値（このハンズオン） | 補足 |
   |---|---|---|
   | **Deployment Method** | **Code**（Upload source code as a ZIP package） | ソース コードを ZIP でアップロードする方式。`Container`（Docker イメージを ACR 経由）ではなく **Code** を選ぶ。 |
   | **Package Mode** | **Remote**（Recommended） | `requirements.txt` などの依存を **Azure 側がプロビジョニング時にインストール**する。ローカルに依存を含めて丸ごと送る `Bundled` ではなく **Remote** を選ぶ。 |
   | **Deploy to** | **New agent** | 新規に Hosted Agent を作成する。既存を更新する場合は `Existing agent`。 |
   | **Hosted Agent Name** | 自動入力のままでOK（例: `agent-framework-agent-basic-responses`） | 作成される Hosted Agent の名前。任意で変更可。 |

   設定したら **Next**。

4. **③ Review + Deploy**（内容を確認してデプロイ）

   自動検出された内容を確認する。基本的にそのままでOK。

   | 項目 | 値（このハンズオン） | 補足 |
   |---|---|---|
   | **Language** | **Python (Auto-detected)** | コードから自動判定される。 |
   | **Runtime Version** | **Python 3.13** | 実行するランタイム バージョン。 |
   | **Entry Point** | **`python main.py`** | 起動コマンド。ひな形のままでOK。 |
   | **CPU and Memory** | **0.5 CPU / 1.0 Gi** | ハンズオンではこの最小構成で十分。 |

   - 確認できたら **Deploy** を押す（前の画面に戻る場合は **Back**）。
   - リモート ビルド → デプロイが走る（数分かかることがある）。

   > **注意:** 画面下部の注記どおり、Hosted Agent では**データが組織のコンプライアンス境界の外に流れる可能性**があります。実運用では適切なガードレール（セーフガード）を設定し、サードパーティ システムの利用は自己責任で行ってください。

5. デプロイが完了すると、Foundry Toolkit のエクスプローラーの **Hosted Agents** に、作成した Agent が表示される。

> **ヒント:** Hosted デプロイは **Azure 上でビルド・実行**されるため、ローカル実行（A 項）が自宅回線の MTU など**ネットワーク要因でタイムアウトする環境でも、デプロイ自体は影響を受けません**。

### C. デプロイした Agent をテストする
1. Foundry Toolkit で対象の Hosted Agent を開き、**Playground** タブから質問を送って応答を確認する。

## 参考：azd（CLI）で進める場合
VS Code を使わず、ターミナルだけで完結させたい場合は次の流れになります（任意）。

### azd（Azure Developer CLI）のインストール

未インストールの場合は、まず azd をインストールします（インストール済みなら次の「デプロイの流れ」へ）。

```powershell
# Windows（winget を使う場合。推奨）
winget install microsoft.azd

# winget が使えない場合（PowerShell インストール スクリプト）
powershell -ex AllSigned -c "Invoke-RestMethod 'https://aka.ms/install-azd.ps1' | Invoke-Expression"
```

インストール後、**ターミナルを開き直してから**バージョンを確認します（`azd` が PATH に反映されます）。

```powershell
azd version
```

> **（注）** macOS は `brew install azure-dev`、Linux は `curl -fsSL https://aka.ms/install-azd.sh | bash` でインストールできます。詳細は [Install azd](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) を参照。

### デプロイの流れ

```powershell
azd auth login             # Azure にサインイン（未サインインの場合。ブラウザーが開く）
azd ai agent init          # ひな形作成
azd provision              # 必要なリソースをプロビジョニング
azd deploy                 # Hosted Agent をデプロイ
azd ai agent invoke "こんにちは、あなたは何ができますか？"   # 動作確認
azd ai agent monitor --follow   # トレース/ログを追う
```

> **（注）** 片付け時は `azd down` で作成したリソースを削除できます。詳細は [Step 11 — クリーンアップ](step11-cleanup.md) を参照。

## 時間が足りないとき（最小構成）
- 本 Step は**任意**です。デプロイまで行わず、**「Hosted Agent とは何か（マネージドにコードを公開する仕組み）」** を読んで理解するだけでも目的は達成できます。
- ローカル実行（`python main.py`）だけ試して、Foundry へのデプロイはスキップしても構いません。

## UI が違うとき
- コマンド名（`Foundry Toolkit: Deploy Hosted Agent` 等）が見つからない場合は、Foundry Toolkit 拡張機能が**プレリリース チャネル**で最新化されているか確認する。
- ウィザードのステップ名・項目名が異なる場合は、**Foundry Project Setup=デプロイ先プロジェクト選択**、**Basics=Deployment Method（Code）/ Package Mode（Remote）/ Deploy to（New agent）/ Hosted Agent Name の設定**、**Review + Deploy=Language / Runtime Version / Entry Point / CPU の確認**、に相当する箇所を探す（多くは既定値・自動検出のままでOK）。
- 解決しない場合は [Step 12 — トラブルバッファ](step12-troubleshoot.md) を参照。

## 完了チェック
- [ ] Hosted Agent が「マネージド ホストにコードをデプロイして動かす仕組み」だと説明できる
- [ ]（任意）VS Code から Hosted Agent をデプロイし、Foundry Toolkit の Hosted Agents に表示された
- [ ]（任意）Playground から応答を確認できた

---

⬅️ 前へ: [Step 9 — Microsoft 365 / Teams への公開](step09-m365-publish.md) ｜ 🏠 [目次](README.md) ｜ ➡️ 次へ: [Step 11 — クリーンアップ](step11-cleanup.md)
