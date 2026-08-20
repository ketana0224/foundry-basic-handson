# Step 10 — Hosted Agent デプロイ（GA 構成・発展・任意）（15分 / 2:40〜2:55）

⬅️ 前へ: [Step 9 — Microsoft 365 / Teams への公開](step09-m365-publish.md) ｜ 🏠 [目次](README.md) ｜ ➡️ 次へ: [Step 11 — クリーンアップ](step11-cleanup.md)

---

> **この Step は「発展・任意」です。** ここまでのハンズオンは **ポータル UI のみ** で完結しますが、本 Step だけは **コマンドライン（azd CLI）とコード** を使う発展課題です。時間が押している場合や、まずはノーコードの流れを一通り体験したい場合は、この Step を飛ばして [Step 11 — クリーンアップ](step11-cleanup.md) に進んで構いません。**必須パス（約180分）はこの Step を含みません。**

> **（この版について）** 本 Step は **一般提供（GA）の `azure-ai-agentserver-responses`（2.0.0 以上）** だけで Hosted Agent を構成します。プレビュー専用パッケージ（`agent-framework-foundry-hosting` など）や `pip install --pre` は **使いません**。GA パッケージのみで Playground / SDK / Teams から呼び出せる Agent を作ります。

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

## この版で使うパッケージ（GA）
| パッケージ | 役割 | 備考 |
|---|---|---|
| `azure-ai-agentserver-responses` (>=2.0.0) | Responses プロトコルの **Hosted Agent ランタイム本体** | GA（Production/Stable）。`ResponsesAgentServerHost` / `@app.response_handler` / `TextResponse` / `app.run()` を提供。 |
| `azure-ai-agentserver-core` (>=2.0.0) | 上記の共通基盤 | `-responses` の依存として **自動解決**（明示不要）。 |
| `openai` (>=1.100.0) | モデル呼び出し（`AsyncAzureOpenAI`） | Foundry の Azure OpenAI 互換エンドポイントを叩く。 |
| `azure-identity` (>=1.19.0) | `DefaultAzureCredential` で無資格キー認証 | Hosted 実行時はマネージド ID、ローカルは `az login`。 |

> **（重要）** これらはすべて **PyPI の安定版**です。`--pre` も `agent-framework-foundry-hosting` も不要です。

## 事前準備
- **Azure Developer CLI（azd）1.27.1 以上**。`azd ai agent` コマンド群が使えること（`azd version` で確認）。**VS Code の Foundry Toolkit 拡張機能は今回は使いません**（本手順は CLI だけで完結します）。

  未導入の場合は次でインストールします（導入済みならスキップ）:

  ```powershell
  winget install microsoft.azd
  ```

  > **（注）** インストール後は VS Code / ターミナルを開き直して `azd version` が表示されることを確認してください。
- **Python 3.11 以上**（GA ランタイムのビルドに使用）。
- **既存の Foundry プロジェクトに対して `Foundry Project Manager` ロール**を持っていること（Hosted Agent の登録＝データプレーン操作に必要）。`azure.yaml` の `ai-project` に既存プロジェクトの `endpoint` を設定するため、`azd deploy` は**新規プロジェクトを作らず既存プロジェクトへ接続するだけ**になります（詳細は C 節参照）。
- **Azure へのサインイン**（未サインインの場合）。`azd`・`az` はいずれも Azure 認証が必要です。まず次でサインインしておきます（ブラウザーが開きます）:

  ```powershell
  az login
  ```

  > **（注）** 複数のテナントやサブスクに所属する場合は `az account set --subscription "<サブスク名 or ID>"` で対象を揃えておくと確実です。

- リソース プロバイダーの登録（初回のみ。`az login` 済みであること）:

  ```powershell
  az provider register --namespace Microsoft.CognitiveServices
  ```

  > **（注）** これはサブスクリプション レベルの操作です。**共有ハンズオン環境では管理者が登録済み**のため、参加者は実行不要（権限が無い場合はスキップして構いません）。

> **（注）** `azd ai agent` はプレビュー段階のため、対話プロンプトの文言や選択肢名は更新されることがあります。表示が異なる場合は近い名称を選んでください。

## 手順（azd CLI を使うフロー）

### A. Agent プロジェクトのひな形を用意する（`azd ai agent init`）
1. ターミナル（PowerShell）で**自分用の作業フォルダーを新規作成**して移動する。

   ```powershell
   # 空の作業フォルダーを作って移動（フォルダー名は任意）
   mkdir $env:USERPROFILE\AIFoundryProjects -Force
   cd $env:USERPROFILE\AIFoundryProjects
   ```

   > **（注意）** このハンズオン文書のリポジトリ（`…\Foundry_basic_Handson` など）の**中では `init` しない**でください。`azd ai agent init` はカレント フォルダーに `azure.yaml` や `src\` を生成するため、**空の専用フォルダー**で行うのが安全です。

2. azd で Azure にサインインする（未サインインの場合。ブラウザーが開きます）。

   ```powershell
   azd auth login
   ```

3. ひな形を生成する。

   ```powershell
   azd ai agent init
   ```

   対話プロンプトで次のとおり選ぶ（文言は版により多少変わります。画面の `azd` バージョンは `v1.0.0-beta.10` など）。

   | 順 | プロンプト | 選ぶ値（このハンズオン） | 補足 |
   |---|---|---|---|
   | 1 | **Select a language** | **Python**（↑↓ で選び Enter） | もう一方は C#。本手順は Python 前提。 |
   | 2 | **Select a starter template** | **Basic agent (Responses, Agent Framework, Python)** | `responses` プロトコル 2.0.0 の最小構成。一覧には `Agent with Local Tools` / `Agent with MCP Tools` / `Basic agent (Invocations, …)` / `Note-taking agent (…without a framework…)` などもあるが、**`Basic agent (Responses, Agent Framework, Python)`** を選ぶ。 |
   | 3 | **Select a Foundry project to host your agent** | **Use an existing Foundry project** | 本ハンズオンは**既存の共有 Foundry プロジェクト**を使うため、`Create a new Foundry project` ではなく **`Use an existing Foundry project`** を選ぶ。続けて表示される一覧から**共有プロジェクト（例: `proj-foundryobs-jyenh`）を選択**する。 |
   | 4 | **How would you like to proceed?**（モデルの取り扱い） | **Skip this model entirely (remove from azure.yaml)** | ひな形の `azure.yaml` は既定で `gpt-5.4-mini` の**新規モデルデプロイ**を作ろうとしますが、共有プロジェクトには**既に `gpt-5.4-mini` がデプロイ済み**です。受講者11人が各自新規モデルを作ると**クォータ枯渇・重複**になるため、**`Skip this model entirely (remove from azure.yaml)`** を選んで `azure.yaml` からモデル定義を外します（モデル名は次の順5で指定）。`Deploy as specified` / `Choose a different model` は新規デプロイになるので**選ばない**。 |
   | 5 | **Enter a value for AZURE_AI_MODEL_DEPLOYMENT_NAME:** | **`gpt-5.4-mini`**（手入力して Enter） | 順4でモデルを Skip すると、init の最後に `azure.yaml` が参照する環境変数の値を聞かれます。**共有プロジェクトに実在するデプロイ名**（本ハンズオンでは `gpt-5.4-mini`）を手入力します。デプロイ名は Foundry ポータルの **「モデル + エンドポイント」→「デプロイ済みモデル」** タブの「名前」列で確認できます。 |

   > **（フォルダー名について）** テンプレート選択後、`azd` は**テンプレート名のフォルダー**を自動作成します（フォルダー名は聞かれません）。例: `Basic agent (Responses, Agent Framework, Python)` を選ぶと `…\AIFoundryProjects\agent-framework-agent-basic-responses\` が作られ、そこへ一式がコピーされます。
   >
   > **（なぜこれか）** 最小構成で `azure.yaml`（`responses` プロトコル 2.0.0）とプロジェクト構造が生成されるため。**このひな形から `main.py` と `requirements.txt` を GA 版に差し替えて使います。** 手順 3 で**既存プロジェクトを選ぶ**とエンドポイント接続が構成されますが、`azure.yaml` は念のため C 節でも接続用に確認・編集します（`ai-project` に `endpoint` を追加）。
   >
   > **（重要・複数受講者の一意名）** 本ハンズオンは **user01〜user11 が同一の共有 Foundry プロジェクト**を使います。Hosted Agent は **`azure.yaml` のサービス名がそのまま共有プロジェクト上のエージェント名**になるため、**既定名のままだと受講者どうしで衝突・上書き**します。Step 3 / Step 4 と同様に、**自分の番号（`userNN`）を含めて一意化**してください（次の C 節でサービス名を変更します）。
   >
   > **（実機で出る WARNING は想定内）** 手順 3〜4 の途中で次のような警告が出ることがあります。
   >
   > ```text
   > WARNING: unable to check whether agent "..." already exists: HTTP request failed:
   >   Get https://aif-foundryobs-jyenh.services.ai.azure.com/...: dial tcp: lookup
   >   aif-foundryobs-jyenh.services.ai.azure.com: no such host
   > ```
   >
   > これは `azd` が**リソース名ホスト**（例: `aif-foundryobs-jyenh.services.ai.azure.com`）でエンドポイントを組み立てているためで、このホストは **DNS 解決できません**（正しいのは**カスタム サブドメイン** `foundryobsjyenh.services.ai.azure.com`）。ここでは「エージェント存在チェックができなかった」だけの**非致命的警告**なので init は続行して構いませんが、**このままでは後続の `azd provision`/`deploy` がエンドポイント不到達で失敗**します。**必ず C 節の C3 で `FOUNDRY_PROJECT_ENDPOINT` をカスタム サブドメインに設定**してください。

4. 生成物を確認する。**ルートに `azure.yaml`**、コードは **`src\<テンプレート名>\`** の下にあります（例: `src\agent-framework-agent-basic-responses\` の `main.py` / `requirements.txt`）。以降このフォルダーで作業します。

   > **（参考・VS Code 派の人へ）** VS Code の Foundry Toolkit 拡張機能の **`Create new Hosted Agent`** も内部で `azd ai agent init` を実行するだけなので、生成物は同じです。拡張機能を入れている場合はそちらの GUI から作ってもかまいません（本手順は CLI で統一します）。

### B. `main.py` と `requirements.txt` を GA 版に差し替える
ひな形の中身をこのハンズオンの GA 構成に置き換えます。**この B 節では `azure.yaml` は触りません**（エントリ ポイント `python main.py` と `responses` プロトコル 2.0.0 はそのままで動きます。`azure.yaml` の編集は C 節で行います）。

0. **VS Code で生成フォルダーを開く。** 以降の編集をしやすくするため、init で作られたフォルダーを VS Code で開きます。

   ```powershell
   code $env:USERPROFILE\AIFoundryProjects\agent-framework-agent-basic-responses
   ```

   > **（メモ）** テンプレートが異なる場合はフォルダー名も変わります（例: `agent-framework-agent-basic-responses`）。`azure.yaml` がルートに見えるフォルダーを開いてください。既に開いている VS Code に追加したい場合は **［ファイル］→［フォルダーをワークスペースに追加］** でも構いません。

1. **`requirements.txt`** を次の内容で**丸ごと上書き**する。

   ```text
   azure-ai-agentserver-responses>=2.0.0
   openai>=1.100.0
   azure-identity>=1.19.0
   python-dotenv>=1.0.0
   ```

   > **（注）** `azure-ai-agentserver-core` は `-responses` の依存として自動で入るため、明示不要です。`--pre` は付けません（すべて安定版）。

2. **`main.py`** を次の内容で**丸ごと上書き**する。ユーザー入力を Foundry のモデルに渡し、その回答を返す最小の Agent です。

   ```python
   import asyncio
   import os

   from dotenv import load_dotenv

   # .env をローカル起動時に読み込む（Hosted 実行時は Foundry が環境変数を注入するため無害）
   load_dotenv()

   from azure.ai.agentserver.responses import (
       CreateResponse,
       ResponseContext,
       ResponsesAgentServerHost,
       TextResponse,
   )
   from azure.identity import DefaultAzureCredential, get_bearer_token_provider
   from openai import AsyncAzureOpenAI

   # GA のホスティング ランタイム（azure-ai-agentserver-responses 2.0.0）
   app = ResponsesAgentServerHost()

   # Foundry がデプロイ時に自動注入する環境変数
   PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]           # https://<...>.services.ai.azure.com/api/projects/<proj>
   MODEL_DEPLOYMENT = os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"]     # 例: gpt-5.4-mini

   # プロジェクト エンドポイントからアカウント ルート（/api/projects/... を除いた部分）を取り出す
   ACCOUNT_ENDPOINT = PROJECT_ENDPOINT.split("/api/projects/")[0]

   # Hosted 実行時はマネージド ID、ローカルは az login のユーザーで認証する
   token_provider = get_bearer_token_provider(
       DefaultAzureCredential(),
       "https://cognitiveservices.azure.com/.default",
   )
   client = AsyncAzureOpenAI(
       azure_endpoint=ACCOUNT_ENDPOINT,
       azure_ad_token_provider=token_provider,
       api_version="2024-10-21",
   )


   @app.response_handler
   async def handler(
       request: CreateResponse,
       context: ResponseContext,
       cancellation_signal: asyncio.Event,
   ):
       """ユーザー入力をモデルに渡し、その回答を返す最小エージェント。"""
       user_text = await context.get_input_text()

       completion = await client.chat.completions.create(
           model=MODEL_DEPLOYMENT,
           messages=[
               {"role": "system", "content": "あなたは日本語で簡潔に答える親切なアシスタントです。"},
               {"role": "user", "content": user_text},
           ],
       )
       answer = completion.choices[0].message.content or ""
       return TextResponse(context, request, text=answer)


   def main() -> None:
       app.run()  # ローカルは http://localhost:8088 で待ち受ける（Foundry 上では PORT を自動設定）


   if __name__ == "__main__":
       main()
   ```

   > **（ハンドラーの約束事）** `@app.response_handler` を付ける関数は **`async` かつ引数は必ず 3 つ**（`request`, `context`, `cancellation_signal`）です。この形以外にすると起動時にハンドラーが登録されません。

   > **（疎通確認だけしたいとき）** モデルを呼ばず入力をそのまま返す最小版に差し替えると、RBAC やモデル設定に依存せず起動確認できます（トラブル切り分けに便利）。
   >
   > ```python
   > @app.response_handler
   > async def handler(request, context, cancellation_signal):
   >     text = await context.get_input_text()
   >     return TextResponse(context, request, text=f"Echo: {text}")
   > ```

3. 仮想環境（`.venv`）を作って依存をインストールする。

   > **（注意）** グローバルの Python に古いパッケージが入っていると衝突するため、**必ず `.venv` を作って隔離**します。

   ```powershell
   # 1) main.py / requirements.txt のあるフォルダーへ移動（フォルダー名は生成時のものに置き換え）
   cd src\agent-framework-agent-basic-responses

   # 2) 仮想環境を作成して有効化
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   # ↑ 実行ポリシーで弾かれる場合は先に次を実行:
   #   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

   # 3) pip を更新し、依存をインストール（安定版のみ。--pre は不要）
   python -m pip install --upgrade pip
   python -m pip install -r requirements.txt
   ```

   > **（注）** `pip install` はダウンロードとビルドに **数分程度**かかることがあります。途中で止まったように見えても、そのまま完了を待ってください。

   > プロンプト先頭に `(.venv)` が付いていれば有効化成功です。以降このターミナルで `python` を実行すると `.venv` 側が使われます。

4. `.env` に接続情報を設定する（ローカル起動時のみ必要。Hosted デプロイ時は Foundry が自動注入するため不要）。

   ```ini
   FOUNDRY_PROJECT_ENDPOINT=<Foundry プロジェクトのエンドポイント>
   AZURE_AI_MODEL_DEPLOYMENT_NAME=<Step 2 で使ったモデル デプロイ名>
   ```

   > **（重要）** `main.py` の先頭で `load_dotenv()` を呼んでいるため、この `.env` は **`main.py` と同じフォルダー**に置きます。置き忘れると起動時に `KeyError: 'FOUNDRY_PROJECT_ENDPOINT'` で落ちます。`.env` を使わず、その場のシェルに直接設定してもかまいません（例: `$env:FOUNDRY_PROJECT_ENDPOINT = "..."`）。

   **各値を Foundry ポータルで確認する手順:**

   - **`FOUNDRY_PROJECT_ENDPOINT`**（プロジェクト エンドポイント）
     1. Foundry ポータル（<https://ai.azure.com>）で対象プロジェクトを開く。
     2. **ホーム** 画面の下部にある **「プロジェクト エンドポイント」** の欄を確認する（`https://<リソース名>.services.ai.azure.com/...` の形式）。
     3. 右側のコピー アイコンで値をコピーし、`.env` に貼り付ける。
     > **（注）** 隣にある **「Azure OpenAI エンドポイント」**（`...openai.azure.com`）ではありません。必ず **プロジェクト エンドポイント**（`services.ai.azure.com`）を使ってください。

   - **`AZURE_AI_MODEL_DEPLOYMENT_NAME`**（モデル デプロイ名）
     1. 左メニューの **「デプロイ」** を開く。
     2. **「デプロイ済みモデル」** タブの一覧から、使いたいモデルの **「名前」** 列の値をそのまま使う（例: `gpt-5.4-mini`）。
     3. Step 2 のプレイグラウンドで使ったデプロイ名と合わせると確実です。
     > **（注）** 「モデル」列ではなく **「名前」列（デプロイ名）** を使います。デプロイの状態が **Succeeded** のものを選んでください。

   `DefaultAzureCredential` を使うため、事前に `az login` でサインインしておく。ローカルでモデル呼び出しまで試す場合、**サインインしているユーザーに Foundry アカウントに対する `Cognitive Services OpenAI User` ロール**が必要です（無い場合は上記 Echo 版で起動確認だけ行い、モデル呼び出しはデプロイ後に確認してください）。

5. ローカルで起動する。

   ```powershell
   python main.py
   ```

   起動すると `http://localhost:8088` で待ち受けます。別のターミナルから疎通確認します（Responses プロトコルの `POST /responses`）。

   ```powershell
   curl -X POST http://localhost:8088/responses `
     -H "Content-Type: application/json" `
     -d '{"input": "こんにちは。あなたは誰ですか？", "stream": false}'
   ```

   > **（確認）** ヘルスチェックは `GET http://localhost:8088/readiness` で 200 が返れば OK です。確認できたら `Ctrl+C` で停止します。

### C. Hosted Agent としてデプロイする

デプロイは `azd` を使います。`azure.yaml` の `ai-project` に既存プロジェクトの `endpoint` を設定すると、**新規プロジェクトを作らず既存プロジェクトへ接続**します。`azd deploy` で既存プロジェクトへの接続と Hosted Agent 登録を実行します。

1. **プロジェクト ルートへ移動**して azd 環境を作る。

   ```powershell
   # ひな形のルート（azure.yaml がある階層）へ移動
   cd $env:USERPROFILE\AIFoundryProjects\agent-framework-agent-basic-responses   # 生成されたフォルダー名（テンプレート名）に置き換え
   azd env new handson-hosted-userNN                        # userNN は自分の番号（例: handson-hosted-user01）
   ```

2. **サブスクリプション・リージョン・プロジェクト ID・エンドポイント・モデル デプロイ名を設定**する。

   ```powershell
   # サブスクリプション ID（空の環境には入っていないので必須）
   azd env set AZURE_SUBSCRIPTION_ID "d1bf4d07-2dac-43a8-9060-4d5274fc7e33"        # az account show --query id -o tsv で確認可

   # リージョン（既存プロジェクトのリージョンに合わせる。例: eastus2）
   azd env set AZURE_LOCATION "eastus2"

   # プロジェクトのリソース ID（Hosted Agent の登録先。endpoint だけでは解決できないので必須）
   azd env set AZURE_AI_PROJECT_ID "/subscriptions/d1bf4d07-2dac-43a8-9060-4d5274fc7e33/resourceGroups/rg-foundryobs-eastus2/providers/Microsoft.CognitiveServices/accounts/aif-foundryobs-jyenh/projects/proj-foundryobs-jyenh"

   # プロジェクト エンドポイント（カスタム サブドメイン ホストであること）
   azd env set FOUNDRY_PROJECT_ENDPOINT "https://foundryobsjyenh.services.ai.azure.com/api/projects/proj-foundryobs-jyenh"

   # モデル デプロイ名（既存プロジェクトにデプロイ済みのもの）
   azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME "gpt-5.4-mini"
   ```

   > **（重要・空環境の落とし穴）** 手順 1 の `azd env new` は**空の新規環境**を作ります。`AZURE_SUBSCRIPTION_ID` がないと `ERROR: AZURE_SUBSCRIPTION_ID is required ...`、`AZURE_AI_PROJECT_ID` がないと `ERROR: Microsoft Foundry project ID is required: AZURE_AI_PROJECT_ID is not set` で `azd deploy` が失敗します。**必ず上記 5 つをすべて設定**してください（`azd env get-values` で現在値を確認できます）。
   >
   > **（`AZURE_AI_PROJECT_ID` の形式）** `/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<アカウント>/projects/<プロジェクト>` です。値が不明なときは Foundry ポータルのプロジェクト概要、または `az cognitiveservices account show` で確認できます。

   > **（重要・エンドポイントの落とし穴／`no such host` の原因）** `FOUNDRY_PROJECT_ENDPOINT` のホストは **カスタム サブドメイン**（例: `foundryobsjyenh.services.ai.azure.com`）である必要があります。**リソース名そのまま**（例: `aif-foundryobs-jyenh.services.ai.azure.com`）だと `azd deploy` の Agent 登録が `dial tcp: lookup aif-foundryobs-jyenh.services.ai.azure.com: no such host` で失敗します。`azd deploy` はこの `FOUNDRY_PROJECT_ENDPOINT` のホストへ `/api/projects/<プロジェクト>/agents/...` を叩くため、**ここを間違えると DNS 解決できません**。ポータルの「プロジェクト エンドポイント」に表示される値（カスタム サブドメイン）をそのまま使ってください。

3. **`azure.yaml` を既存プロジェクト接続用に編集**する。`ai-project` サービスに `endpoint` を追加し、**エージェント サービス名を一意にリネーム**します（既存プロジェクトには使用するモデルが既にデプロイ済みのため、`deployments` は空の `[]` のままで構いません）。

   変更後:

   ```yaml
   services:
     agent-basic-responses-userNN:   # ← userNN は自分の番号（例: agent-basic-responses-user01）。共有プロジェクトでの衝突を避ける
       project: src/agent-framework-agent-basic-responses   # ← ソース ディレクトリは変えない
       host: azure.ai.agent
       # ...（以降はそのまま）
     ai-project:
       host: azure.ai.project
       deployments: []                          # ← 空のまま（既存プロジェクトのモデルを使う）
       endpoint: ${FOUNDRY_PROJECT_ENDPOINT}    # ← この 1 行を変更または追加（既にある場合はカスタム サブドメインに直す）
   ```

4. **`azd deploy` でデプロイ**する。

   ```powershell
   # azure.yaml があるプロジェクト ルート（src の 1 つ上）に戻ってから実行する
   cd $env:USERPROFILE\AIFoundryProjects\agent-framework-agent-basic-responses

   # 既存プロジェクトへ接続し Hosted Agent を登録
   azd deploy
   ```

   > **（重要・実行フォルダ）** `azd deploy` は**カレント ディレクトリの `azure.yaml` を読み込む**ため、必ず **`azure.yaml` があるプロジェクト ルート**（`src` フォルダーの **1 つ上**）で実行してください。`src` の中に入ったまま実行すると `azure.yaml` が見つからずエラーになります。`ls azure.yaml` でファイルの存在を確認してから `azd deploy` を実行すると確実です。

   > **（注）** `endpoint` を設定しているため `azd deploy` は**新規 Foundry プロジェクト/モデルを作成せず既存プロジェクトへ接続**します。リモート ビルドで `requirements.txt` から GA パッケージが解決されるため、ローカルに Docker が無くてもデプロイできます。`ai-project` が `Done`、Agent サービスが `Done` になれば成功です。

5. **動作確認**する。

   ```powershell
   # 呼び出し
   azd ai agent invoke -p "こんにちは。自己紹介してください。"

   # ログをリアルタイムで見る（別ターミナル推奨）
   azd ai agent monitor --follow
   ```

   デプロイ後は **Foundry ポータルの Playground** からも同じ Hosted Agent を選んで会話できます。

> **（補足）** ここまでが CLI だけで完結する Hosted Agent デプロイの全体像です（`azd ai agent init` → `main.py`/`requirements.txt` を GA 版へ → `azure.yaml` の `ai-project` に `endpoint` を設定 → 環境変数（`AZURE_SUBSCRIPTION_ID`/`AZURE_LOCATION`/`AZURE_AI_PROJECT_ID`/`FOUNDRY_PROJECT_ENDPOINT`/`AZURE_AI_MODEL_DEPLOYMENT_NAME`）を設定 → `azd deploy`）。

## トラブルシュート

### 症状 A: デプロイ／起動が失敗する（`ModuleNotFoundError: No module named 'azure.ai.agentserver'`）
- **原因**: `requirements.txt` に `azure-ai-agentserver-responses` が入っていない（差し替え漏れ）、または `.venv` を有効化せずに古い Python で起動している。
- **対処**:
  1. `requirements.txt` が **B 節の GA 内容**になっているか確認（`azure-ai-agentserver-responses>=2.0.0` が先頭にあること）。
  2. `(.venv)` が付いたターミナルで `python -m pip install -r requirements.txt` を実行。
  3. `python -c "import azure.ai.agentserver.responses; print('ok')"` が `ok` を返すことを確認してから再デプロイ。

> **（無害なメッセージ）** ログに末尾が `...No module named 'agents'`（複数形 `agents`）と出ることがありますが、これは無害です。**問題なのは `azure.ai.agentserver` を解決できないケース**です。

### 症状 B: 実行時にモデル呼び出しが 401 / 403（`PermissionDenied` / `access denied`）
- **原因**: Hosted Agent の **Instance Identity（マネージド ID）に `Cognitive Services OpenAI User` ロールが無い**、または付与直後でトークン キャッシュが古い。
- **対処**:
  1. Foundry アカウント スコープで、Hosted Agent の Instance Identity に **`Cognitive Services OpenAI User`** を付与する。
     ```powershell
     az role assignment create `
       --assignee-object-id <Instance Identity のプリンシパル ID> `
       --assignee-principal-type ServicePrincipal `
       --role "Cognitive Services OpenAI User" `
       --scope <Foundry アカウントのリソース ID>
     ```
  2. 付与後もエラーが続く場合は、コンテナがロール付与前のトークンをキャッシュしているだけなので、**`main.py` に軽微な変更を加えて再ビルド（`azd deploy`）** し、新しいコンテナを起動する。
  3. ローカルで同じエラーが出る場合は、`az login` 中のユーザーに同ロールが付与されているか確認。

### 症状 C: `KeyError: 'AZURE_AI_MODEL_DEPLOYMENT_NAME'` / モデルが見つからない
- **原因**: 環境変数が注入されていない（`azd env set` 漏れ）、またはモデル デプロイ名が実在しない。
- **対処**: `azd env get-values` で `AZURE_AI_MODEL_DEPLOYMENT_NAME` を確認。ポータルの **デプロイ → デプロイ済みモデル** で **状態が Succeeded** のデプロイ名（「名前」列）と一致させる。

### 症状 D: デプロイが `provisioning failed` / ファイルが見つからない
- **原因**: `main.py` と `requirements.txt` が zip ルート（`azure.yaml` が参照するサービス ディレクトリ）に無い、または対象モデルが未デプロイ。
- **対処**: `main.py` / `requirements.txt` がサービス ディレクトリ直下にあること、モデル デプロイが存在することを確認して再実行。

### 症状 E: `ERROR: AZURE_SUBSCRIPTION_ID is required ...` / `ERROR: Microsoft Foundry project ID is required: AZURE_AI_PROJECT_ID is not set`
- **原因**: `azd env new` で作った**空の新規環境**にサブスクリプション/リージョン/プロジェクト ID が入っていない。
- **対処**: C 節の手順 2 で `AZURE_SUBSCRIPTION_ID` / `AZURE_LOCATION` / `AZURE_AI_PROJECT_ID` を `azd env set` する。`AZURE_AI_PROJECT_ID` は `/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<アカウント>/projects/<プロジェクト>` 形式。`azd env get-values` で 5 つ（上記＋`FOUNDRY_PROJECT_ENDPOINT`/`AZURE_AI_MODEL_DEPLOYMENT_NAME`）が入っていることを確認してから `azd deploy`。

### 症状 F: `dial tcp: lookup aif-... : no such host`（Agent 登録が `create_agent: HTTP request failed`）
- **原因**: `FOUNDRY_PROJECT_ENDPOINT` のホストが**リソース名そのまま**（例: `aif-foundryobs-jyenh.services.ai.azure.com`）になっている。このホストは DNS 解決できず、`azd deploy` の Agent 登録先 URL が引けない。
- **対処**: ホストを**カスタム サブドメイン**（例: `foundryobsjyenh.services.ai.azure.com`）に直す。ポータルの「プロジェクト エンドポイント」に表示される値をそのまま使う。

  ```powershell
  azd env set FOUNDRY_PROJECT_ENDPOINT "https://<カスタムサブドメイン>.services.ai.azure.com/api/projects/<プロジェクト名>"
  azd env get-values   # FOUNDRY_PROJECT_ENDPOINT のホストが aif-... でないことを確認
  azd deploy
  ```

### 切り分けのコツ
- **起動できない**（Playground/`invoke` が返らない）→ 症状 A（パッケージ）を疑う。まず Echo 版に差し替えて「モデル無しで起動するか」を確認する。
- **起動はするが応答がエラー**→ 症状 B（RBAC）／症状 C（モデル名）を疑う。`azd ai agent monitor --follow` でコンテナ ログを見る。
- ローカルの `curl POST /responses` が通れば、コード自体は正しい。あとはデプロイ環境の環境変数と RBAC の問題に絞れる。

## 時間が足りないときは
デプロイまで到達できなくても問題ありません。**「コードで書いた Agent を GA パッケージだけで Hosted に載せられる」** という流れを理解できれば十分です。ローカルの `python main.py` + `curl POST /responses` で疎通確認できたら、この Step のねらいは達成です。

## メニュー名・プロンプトが違うときは
`azd`（特に `azd ai agent`）はプレビュー〜更新が続いており、対話プロンプトの文言や選択肢名が変わることがあります。表示が異なる場合は近い名称（`azd ai agent init` / `azd deploy`）を選んでください。コア概念（**GA ランタイム `azure-ai-agentserver-responses` を、`ai-project` の `endpoint` で既存プロジェクトへ接続し `azd deploy` で登録**）は同じです。

## 完了チェック
- [ ] `requirements.txt` を GA 版（`azure-ai-agentserver-responses>=2.0.0`）に差し替えた。
- [ ] `main.py` を GA 版（`ResponsesAgentServerHost` + `@app.response_handler`）に差し替えた。
- [ ] `.venv` で `pip install -r requirements.txt`（`--pre` なし）が成功した。
- [ ] ローカルで `python main.py` を起動し、`POST /responses` が応答した。
- [ ]（任意）`azure.yaml` の `ai-project` に `endpoint` を設定し、`azd deploy` で既存プロジェクトに登録して、`azd ai agent invoke` / Playground で応答を確認した。

---

⬅️ 前へ: [Step 9 — Microsoft 365 / Teams への公開](step09-m365-publish.md) ｜ 🏠 [目次](README.md) ｜ ➡️ 次へ: [Step 11 — クリーンアップ](step11-cleanup.md)
