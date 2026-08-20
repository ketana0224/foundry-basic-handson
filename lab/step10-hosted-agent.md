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
- **既存の Foundry プロジェクトに対して `Foundry Project Manager` ロール**を持っていること（Hosted Agent の登録＝データプレーン操作に必要）。**サブスクリプション レベルのロールは不要**です（既存プロジェクトへのデプロイは `azd deploy` だけで完結し、`azd provision` は実行しません）。
- **Azure へのサインイン**（未サインインの場合）。VS Code の Foundry Toolkit デプロイ・`azd`・`az` はいずれも Azure 認証が必要です。まず次でサインインしておきます（ブラウザーが開きます）:

  ```powershell
  az login
  ```

  > **（注）** 複数のテナントやサブスクに所属する場合は `az account set --subscription "<サブスク名 or ID>"` で対象を揃えておくと確実です。

- リソース プロバイダーの登録（初回のみ。`az login` 済みであること）:

  ```powershell
  az provider register --namespace Microsoft.CognitiveServices
  ```

  > **（注）** これはサブスクリプション レベルの操作です。**共有ハンズオン環境では管理者が登録済み**のため、参加者は実行不要（権限が無い場合はスキップして構いません）。

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

> **⚠️ インストール直後に `azd: 用語 ... が認識されません` になる場合（重要）**
> winget は azd を `%LOCALAPPDATA%\Programs\Azure Dev CLI\` に入れ、**ユーザー PATH に追記**します。ところが**すでに起動している VS Code プロセスはインストール前の PATH を保持**しているため、**ターミナルのタブを開き直しても認識されません**（同じ VS Code プロセスの子プロセスなので古い環境を引き継ぐ）。次のどちらかで解決します。
>
> **方法1（再起動不要・確実）— 現在のセッションの PATH をレジストリから再読込:**
> ```powershell
> $env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')
> ```
>
> **方法2 — VS Code 本体を完全に再起動**（ターミナル タブを閉じるだけでは不十分。ウィンドウごと閉じて開き直す）。

インストール後、バージョンを確認します。

```powershell
azd version
```

> **（注）** macOS は `brew install azure-dev`、Linux は `curl -fsSL https://aka.ms/install-azd.sh | bash` でインストールできます。詳細は [Install azd](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) を参照。

### デプロイの流れ（ハンズオン手順：Toolkit で作成 → デプロイのみ CLI）

このハンズオン環境では、**VS Code（Foundry Toolkit）からの Hosted Agent デプロイがネットワーク制約で失敗**します。そこで、**エージェントの作成（Create agent）は Toolkit で行い、デプロイだけを CLI（`azd`）に切り替えます**。

Toolkit の **Create agent** を実行すると、エージェント用フォルダー（例 `my-agent-xxxxxx`）が作成され、**その中に `azure.yaml`（azd 用の統合マニフェスト）も一緒に生成されます**。この `azure.yaml` がそのまま `azd` のデプロイ設定になるため、**`azd ai agent init` は実行不要**です。

> **前提:** Toolkit の Create agent が完了し、フォルダー内に `azure.yaml` と `src/<agent-name>/`（`main.py`・`requirements.txt`・`Dockerfile` など）がある状態。まだの場合は先に Toolkit でエージェントを作成してください。

> **✅ ポイント：`azd provision` は実行しません（既存プロジェクトへは不要）。**
> このハンズオンは **Step 3 で作った既存の Foundry プロジェクトに Hosted Agent を登録するだけ**です。これは **データプレーンの操作**で、`azd deploy` だけで完結します。
> `azd provision` は「リソース グループや Application Insights などのインフラを**新規作成**する」サブスクリプション スコープの操作で、実行すると **サブスクリプション レベルの権限**（`Microsoft.Resources/deployments/write` など）が要求され、権限のない参加者は `AuthorizationFailed`（403）になります。既存インフラを使う本ハンズオンでは**不要**なので実行しません。
>
> **必要な権限（参加者）:** 対象 Foundry プロジェクトに **Foundry Project Manager**（エージェント登録に必要なデータプレーン権限）。サブスクリプション レベルのロールは不要です。

#### 手順 1. エージェント フォルダーでターミナルを開く

Toolkit が作成したフォルダー（`azure.yaml` があるフォルダー）をカレントにします。VS Code でそのフォルダーを開いていれば、ターミナルはそこがカレントになっています。

```powershell
cd <Toolkit が作成した agent フォルダー>   # 例: C:\Users\user02\AIFoundryProjects\my-agent-xxxxxx
```

#### 手順 2. Azure にサインイン

```powershell
azd auth login
```

ブラウザーが開くので、ハンズオンで使うアカウント（例: `admin@M365CPI65139919.onmicrosoft.com`）でサインインします。すでにサインイン済みならスキップされます。

#### 手順 3. デプロイ先の既存プロジェクトを指定

まず azd 環境を作成し、Step 3 で使った**既存の Foundry プロジェクト**を 2 つの環境変数で指定します。**両方とも `azd deploy` に必須**です。

```powershell
azd env new dev                                                   # azd 環境を作成（名前は任意。既にあればスキップ）

# ① プロジェクトの ARM リソース ID（azd deploy が登録先プロジェクトを特定するのに必須）
azd env set AZURE_AI_PROJECT_ID "/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>"

# ② プロジェクト エンドポイント（★ホストは「カスタム サブドメイン」を使う。リソース名ではない）
azd env set FOUNDRY_PROJECT_ENDPOINT "https://<customSubDomain>.services.ai.azure.com/api/projects/<project>"

# ③ エージェントが使うモデル デプロイ名（★このプロジェクトの既存チャット モデルのデプロイ名）
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5-mini

azd env get-values                                                # 3 つの値が既存プロジェクトを指しているか確認
```

> **★★ 最重要：エンドポイントのホスト名は「リソース名」ではなく「カスタム サブドメイン」を使う ★★**
> Foundry（Cognitive Services）アカウントは、**アカウント リソース名**と**データプレーンのカスタム サブドメイン**が**異なる**ことがあります。`azd deploy` はエンドポイントの**ホスト名をそのまま DNS で解決**するため、リソース名を使うと `dial tcp: lookup ...: no such host`（名前解決失敗）で失敗します。
>
> | 形式 | 例 | 使えるか |
> |---|---|---|
> | ❌ リソース名 | `aif-foundryobs-jyenh.services.ai.azure.com` | **不可（DNS 解決できない）** |
> | ✅ カスタム サブドメイン | `foundryobsjyenh.services.ai.azure.com` | **これを使う** |
>
> **正しいホスト名の調べ方（どちらか）:**
> - **Foundry ポータル:** 対象プロジェクトの **Overview（概要）** にある **Foundry project endpoint** をそのままコピーする（ここには正しいカスタム サブドメインが入っています）。
> - **CLI:** `az cognitiveservices account show -n <account> -g <rg> --query "properties.endpoints" -o json` を実行し、**`"AI Foundry API"`** の値（`https://<customSubDomain>.services.ai.azure.com/`）のホスト名を使う。`properties.customSubDomainName` がカスタム サブドメイン名です。

> **AZURE_AI_PROJECT_ID の調べ方:** `az cognitiveservices account show` などで確認するか、Foundry ポータルのプロジェクト設定から ARM リソース ID をコピーします。形式は `/subscriptions/.../resourceGroups/.../providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>` です。

> **（このハンズオン環境の実際の値・参考）**
> - `AZURE_AI_PROJECT_ID` = `/subscriptions/d1bf4d07-2dac-43a8-9060-4d5274fc7e33/resourceGroups/rg-foundryobs-eastus2/providers/Microsoft.CognitiveServices/accounts/aif-foundryobs-jyenh/projects/proj-foundryobs-jyenh`
> - `FOUNDRY_PROJECT_ENDPOINT` = `https://foundryobsjyenh.services.ai.azure.com/api/projects/proj-foundryobs-jyenh` ← **ホストは `foundryobsjyenh`（カスタム サブドメイン）。`aif-foundryobs-jyenh` ではない。**

#### 手順 4. デプロイ（★ここだけ CLI・`provision` は実行しない）

```powershell
azd deploy
```

`azure.yaml` の Hosted Agent 構成に沿って、**手順 3 で指定した既存の Foundry プロジェクトに Hosted Agent が登録**されます。`ai-project` と Hosted Agent の 2 つが `Done` になり、最後に `SUCCESS: Your application was deployed to Azure ...` が表示されれば完了です。

> **（注）** `azd provision` は実行しません（手順冒頭のポイント参照）。既存プロジェクトへの登録は `azd deploy` だけで完結します。
>
> **デプロイ成功時の出力例:**
> ```text
>   ● ai-project                                    Done   1s
>   ● agent-framework-agent-basic-responses         Done   1m37s
> - Agent playground (portal): https://ai.azure.com/...
> - Agent endpoint (responses): https://<customSubDomain>.services.ai.azure.com/api/projects/<project>/agents/.../endpoint/protocols/openai/responses?api-version=v1
> SUCCESS: Your application was deployed to Azure in 1 minute 37 seconds.
> ```

#### 手順 5. 動作確認

```powershell
azd ai agent invoke "こんにちは、あなたは何ができますか？"   # 応答が返れば成功
azd ai agent monitor --follow                                # トレース/ログを追う（Ctrl+C で終了）
```

Foundry ポータルの対象プロジェクトの **Agents** 一覧にも、デプロイした Hosted Agent が表示されます。

> **（注）** 片付け時は `azd down` で作成した付随リソースを削除できます（既存 Foundry プロジェクト自体は別管理）。詳細は [Step 11 — クリーンアップ](step11-cleanup.md) を参照。

#### （参考）Toolkit を使わず CLI だけで作成する場合

Toolkit を使わずゼロから CLI で作りたい場合は、空フォルダーで `azd ai agent init` を実行してひな形を作成します（ウィザードでテンプレート／サブスク／Foundry プロジェクト／モデルを選択）。既存プロジェクトに入れたい場合は **`Use an existing Foundry project`** を選びます（`Create a new...` を選ぶと新規プロジェクトが作られます）。`init` の時点でデプロイ先プロジェクトが決まり、環境変数（`AZURE_AI_PROJECT_ID` / `FOUNDRY_PROJECT_ENDPOINT`）も設定されます。その後は上記の手順 4〜5（`azd deploy` → 動作確認）と同じです（**`provision` は不要**）。

#### （参考）`azd ai agent init` で何が作られるか（詳しく）

`azd ai agent init` は「Hosted Agent を**ビルド・ローカル実行・デプロイ**するための一式（ひな形プロジェクト）」をカレント ディレクトリに生成します。対話ウィザードで次を順に選びます。

| プロンプト | 内容 |
|---|---|
| **Agent template** | フレームワーク・言語（Python / .NET）別のテンプレートから選択。エージェント名はテンプレート既定値になる（`--agent-name` で変更可）。 |
| **Azure subscription** | Foundry プロジェクトを探す／作るためのサブスク。 |
| **Foundry project** | 既存プロジェクトを選ぶ、または新規作成（新規時はリージョンも選ぶ）。 |
| **Model deployment** | 既存のモデル デプロイを選ぶ、または未指定ならテンプレート既定で作成。 |

ウィザード完了後、次のファイル／フォルダーが生成されます（Azure リソースはまだ**作られません**。`init` は**ローカルにファイルを置くだけ**）。

| 生成物 | 場所 | 役割 |
|---|---|---|
| **`azure.yaml`** | プロジェクト ルート | **統合プロジェクト マニフェスト**。`azd` プロジェクト設定と Hosted Agent 構成を1本に集約する。`services:` 配下に、モデルを持つ `azure.ai.project` サービスと、それを `uses:` で参照する `azure.ai.agent` サービスを定義し、`azd` がプロビジョニング／デプロイ時に依存関係を解決する。**現行はこの1ファイルに集約**（旧 `agent.manifest.yaml` / `agent.yaml` は非推奨）。 |
| **エージェント ソース** | `src/<agent-name>/` | `main.py`（ホスティング ラッパー付きのエージェント本体。`responses` などのプロトコルで HTTP を受ける）、`requirements.txt`（依存）、`.dockerignore` など。ローカル実行時はこの `main.py` が `http://localhost:8088` で起動する。 |
| **`Dockerfile`** | `src/<agent-name>/` | コンテナ デプロイ（`--deploy-mode container`）用。**Code（ZIP）デプロイでは未使用**だが同梱される。 |
| **azd 環境** | `.azure/<ディレクトリ名>-dev/` | `<ディレクトリ名>-dev` という azd 環境を作成し、選んだ Foundry プロジェクトの情報（`FOUNDRY_PROJECT_ENDPOINT`、`AZURE_AI_MODEL_DEPLOYMENT_NAME` など）を格納。以降の `azd provision` / `azd deploy` / `azd ai agent run` がこの環境値を参照する。 |

補足:
- **既定は bicep-less**（インフラ用 Bicep は生成されない。必要になれば後で eject 可能）。`azd provision` はインフラ（リソース グループや Application Insights など）を**新規作成**する操作で、**既存プロジェクトへデプロイする本ハンズオンでは不要**（サブスクリプション権限も要求される）。既存プロジェクトへの登録は `azd deploy` だけで完結する。
- 既存のエージェント コードがあるディレクトリで `init` すると、丸ごとのひな形ではなく **`azure.yaml` のサービス エントリと（コンテナ時のみ）`Dockerfile`** だけが追加される（Bring your own code）。
- 特定サンプルから始めたい場合は `azd ai agent init -m <azure.yaml の URL>` でそのサンプルを取り込める。

> **（注）** 片付け時は `azd down` で作成したリソースを削除できます。詳細は [Step 11 — クリーンアップ](step11-cleanup.md) を参照。

## トラブルシュート：デプロイまわりで失敗する

### 症状 A：起動できない（`ModuleNotFoundError: No module named 'agent_framework_foundry_hosting'`）

Playground が `session_not_ready`（`/readiness` が 200 にならない）になり、ログ末尾に次が出る:

```text
File "/app/main.py", line 7, in <module>
  from agent_framework_foundry_hosting import ResponsesHostServer
ModuleNotFoundError: No module named 'agent_framework_foundry_hosting'
```

**原因:** `requirements.txt` に **`agent-framework-foundry-hosting`（＝`main.py` が import する `agent_framework_foundry_hosting` モジュールの提供元）が入っていない**。この 1 行が欠けると、他がすべて正しくてもコンテナは起動できません。

**対処:** `requirements.txt` に `agent-framework-foundry-hosting` を含める。動作する最小構成は次のとおり:

```text
agent-framework-foundry
agent-framework-foundry-hosting
azure-identity
python-dotenv
debugpy
```

> **（重要）`azure-ai-agentserver-core` / `azure-ai-agentserver-responses` を手でバージョン固定しないこと。** これらは `agent-framework-foundry-hosting` が**依存として自動で解決**します。`agent-framework-foundry-hosting`（最新 `1.0.0b260813`）は **`azure-ai-agentserver-core >=2.1.0b1` / `-responses >=2.1.0b1`（beta 版）を要求**するため、`==2.0.0`（GA）を手で固定すると依存解決が衝突し、インストール失敗やモジュール欠落を招きます（＝症状 A の再発）。

修正後、`requirements.txt` の変更で確実に再ビルドが走ります:

```powershell
azd deploy
```

### 症状 B：起動は成功するが Playground 実行時に失敗する（`store=True` / `Resilient task subsystem missing`）

`/readiness` は 200 だが、質問を送ると実行時に落ちる。ログに `Creating response ... store=True` と `Resilient task subsystem missing in hosted environment ...` が出る。

**背景（事実）:** 上記のとおり `agent-framework-foundry-hosting` は **beta 版の `azure-ai-agentserver-*`（>=2.1.0b1）を要求**します。リモート ビルドの `pip install --pre` はこの beta を取得します。**agentserver を GA `2.0.0` に手で固定して回避することはできません**（hosting が `>=2.1.0b1` を要求するため。前述の症状 A を招く）。

> **（注意）`main.py` の `default_options={"store": False}` では直りません。** これは **エージェント→モデル クライアント（`FoundryChatClient`）呼び出しの既定値**であり、**Foundry ホスト側 Responses エンドポイントの store フラグ（Playground 既定 = True）とは別レイヤー**です。コードを `store=False` にしても、ログには `store=True` が出続けます。

**対処（推測を含む）:** これはランタイム（agentserver beta 版）側の挙動なので、ハンズオンの範囲では次のいずれかで切り分けます。
- `agent-framework-foundry-hosting` を**明示的に別バージョンへ固定**して、依存する agentserver の組み合わせを変える（版ごとの依存は `pip index versions agent-framework-foundry-hosting` 等で確認）。
- 事象が再現する場合は、正確なエラー文言（`azd ai agent monitor --follow` で取得）を添えて Issue（<https://github.com/microsoft/agent-framework/issues>）に報告する。

### 切り分けのコツ
- **正確なエラー文言を先に確定する:** `azd ai agent monitor --follow`（または `--tail <N>`）でコンテナ ログを取得し、`ModuleNotFoundError`（症状 A・起動失敗）か `Resilient task subsystem ...`（症状 B・実行時失敗）かを最初にロックする。
- `ModuleNotFoundError: No module named 'agents'`（末尾が `agents`）は **無害**（任意の A365 計装が無いだけ）。`agent_framework_foundry_hosting`（症状 A）とは別物なので混同しない。
- ログの `model=`（空）は **応答生成時の 1 フィールドで表示上の問題**。モデルは `FoundryChatClient` に埋め込まれているので、起動できている時点でモデルは設定済み。

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
