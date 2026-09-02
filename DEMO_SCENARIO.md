# Azure SRE Agent × GitHub Copilot デモシナリオ

このドキュメントでは、Azure SRE Agent と GitHub Copilot の機能を実際に体験するためのデモシナリオを説明します。

## 前提条件

- Terraform による環境デプロイが完了していること
- Azure Portal へのアクセス権限があること
- GitHub リポジトリへのアクセス権限があること

---

## 1. デプロイした環境の確認

### 1.1 App Service と GitHub の連携確認

1. **Azure Portal** にアクセス
2. リソースグループ `rg-sre-agent-demo` を開く
3. **App Service** を選択
4. 左メニューから **デプロイセンター** を選択
5. 以下を確認：
   - ソース: **GitHub**
   - リポジトリ: 設定した GitHub リポジトリ名
   - ブランチ: **main**
   - ビルドプロバイダー: **GitHub Actions**

6. **GitHub リポジトリ** で `.github/workflows` フォルダを確認
   - Azure が自動生成したワークフローファイルが存在すること

### 1.2 サンプルアプリへのアクセス確認

1. App Service の **概要** ページを開く
2. **既定のドメイン** の URL をクリック（例: `https://app-sreagent-dev-xxxxxx.azurewebsites.net`）
3. サンプルアプリが表示されることを確認する

---

## 2. SRE Agent の設定

### 2.1 SRE Agent へのアクセス

1. ブラウザで [sre.azure.com](https://sre.azure.com) にアクセス
   - または **Azure Portal** のリソースグループ `rg-sre-agent` から **SRE Agent** リソースを選択
2. SRE Agent の管理画面が表示されることを確認する

### 2.2 Application Insights の接続

SRE Agent がデモアプリのテレメトリ（ログ、例外、リクエスト）を読み取れるようにします。

1. SRE Agent の左メニューから **Builder → Knowledge Sources** を開く
2. **Logs** セクションで **Connect** をクリック
3. デモアプリ用の Application Insights リソース（`app-insights-xxxxxx`）を選択して保存する

> **注意**: SRE Agent 自身の RG（`rg-sre-agent`）にある Application Insights ではなく、デモアプリの RG（`rg-sre-agent-demoapp`）にある Application Insights を登録してください。

### 2.3 GitHub リポジトリの接続

SRE Agent がソースコードを読み取り、Issue の起票や PR の作成を行えるようにします。

1. SRE Agent の左メニューから **Builder → Knowledge Sources** を開く
2. **Code** セクションで **Connect** をクリック
3. 認証方式を選択する:
   - **OAuth**（推奨）: GitHub アカウントでサインイン
   - **PAT**: Personal Access Token を入力（`repo` スコープが必要）
4. デモアプリのリポジトリを選択して接続する

### 2.4 インシデントプラットフォームの接続（Azure Monitor）

SRE Agent が Azure Monitor のアラートを自動検出・調査できるようにします。

1. SRE Agent の左メニューから **Builder → Incident platform** を開く
2. ドロップダウンから **Azure Monitor** を選択
3. **Quickstart response plan** トグルを **オフ** にする（次のステップで独自のプランを作成するため）
4. **Save** をクリック
5. ステータスが「Azure Monitor connected」と表示されることを確認する

> **注意**: Quickstart response plan をオンのままにすると、デフォルトで Sev3 アラートのみが Autonomous モードで処理されます。独自のプランで制御したい場合はオフにしてください。

### 2.5 カスタムエージェントの作成

インシデント対応時に使用する専門エージェントを作成します。カスタムエージェントに指示を設定することで、日本語での対応や GitHub Issue の起票など、チーム固有の要件を反映できます。

1. SRE Agent の左メニューから **Builder → Agent Canvas** を開く
2. カスタムエージェントを新規作成する
3. 以下の内容で設定する:

   - **名前**: `demo-incident-handler`（任意）
   - **指示（System Prompt）**:

     ```
     アラート対象リソースがApplication InsightsやLog Analyticsワークスペースであった場合、アラートを上げる要因となったリソースは別に存在するため、ログから特定すること。
     アラート対象リソースにGitHubがリンクされている場合はソースコードも確認すること。
     思考や応答、GitHubへのIssue起票などはすべて日本語で行うこと。
     GitHubにIssueを起票したらGitHub Copilotにアサインすること。
     ```

4. **ツール**（Tools）に以下が含まれていることを確認する:
   - **CreateGithubIssue**
   - 含まれていない場合はツールの管理から追加する
5. 保存する

### 2.6 インシデント対応計画の作成

アラートを検出した際にカスタムエージェントへ自動的にルーティングする計画を作成します。

1. SRE Agent の左メニューから **Builder → Incident response plans** を開く
2. **New incident response plan** をクリック
3. **Step 1 — Set up incident filters**:
   - 名前を入力（例: `demo-all-incidents`）
   - Severity: **All severity** を選択（デモ用にすべてのアラートを対象とする）
4. **Next** をクリック
5. **Step 2 — Preview filter results**: 過去のアラートのプレビューを確認し、**Next** をクリック
6. **Step 3 — Save response plan**:
   - **Response custom agent**: 前のステップで作成したカスタムエージェント（`demo-incident-handler`）を選択
   - **Agent autonomy level**: **Autonomous** を選択（エージェントが自律的に調査・対応を実行）
   - **Save** をクリック
7. 対応計画がリストに表示され、ステータスが **On** になっていることを確認する

> **注意**: Quickstart response plan が残っている場合は、ルーティングの重複を避けるため削除してください。

---

## 3. 疑似インシデント発生

### 3.1 Web アプリで Exception を発生させる

1. サンプルアプリにアクセス
2. **Click me** ボタンをクリックし続ける
3. カウントが 9 より増えないことを確認する（サーバー側では例外が発生している）

### 3.2 Alert の発火確認

1. Azure Portal で Application Insights を開く
2. アラート セクションを確認する
3. **Exception Alert** アラートが発火していることを確認する

### 3.3 SRE Agent のインシデント対応起動

1. SRE Agent の画面を開く（[sre.azure.com](https://sre.azure.com)）
2. **インシデント** 一覧に新しいインシデントが表示されていることを確認する
3. インシデントをクリックして調査スレッドの詳細を表示する
4. SRE Agent が自律的に対応している状況を確認する:
   - ログ・メトリクスの分析
   - ソースコードの確認
   - GitHub への Issue 起票

### 3.4 GitHub リポジトリへの Issue 起票確認

1. GitHub リポジトリ を開く
2. Issues タブを確認し、SRE Agent が自動作成した Issue を確認する
3. Issue に Copilot がアサインされていることを確認する

4. Copilot の対応状況を眺め、Pull Request(PR) が作成されるのを待つ

### 3.5 PR のレビュー・マージ

1. PR の変更内容をレビューし、問題がなければ承認・マージする
2. マージ後、GitHub Actions が自動的にデプロイを開始
3. デプロイ完了後、App Service で修正が反映されていることを確認

## 4. リクエスト監視と Teams 通知の設定

このシナリオでは、SRE Agent にリクエスト状況の確認を依頼し、その後スケジュールタスクを使って定期的（10分間隔）にリクエスト状況を Teams チャネルに通知する仕組みを構築します。

### 4.1 事前準備: Teams コネクタの追加

SRE Agent から Teams にメッセージを投稿できるようにするため、Teams コネクタを追加します。

1. SRE Agent の左メニューから **Capabilities → Tools** を開く
2. **Add tool** をクリック
3. コネクタ一覧から **Microsoft Teams** を選択
4. Teams アカウントで OAuth 認証を行う
5. 接続名を設定する（例: `teams`）
6. **ツールの設定** 画面で、以下のツールを検索して選択する:

   | 検索キーワード | 選択するツール |
   |---|---|
   | `Post message` | **Post message in a chat or channel** |
   | `Post card` | **Post card in a chat or channel** |
   | `List joined` | **List joined teams** |
   | `List channels` | **List channels** |
   | `Get a team` | **Get a team** |
   | `Get details for` | **Get details for a specific channel in a team** |
   | `unified` | **Get unified action input metadata** |
   | `Get message details` | **Get message details** |
   | `mention` | **Get an @mention token for a user** |

7. 各ツールのアクセス許可を **「許可」** に設定する
8. **次へ** をクリックし、確認画面で **作成** をクリックする

### 4.2 事前準備: Teams チームとチャネルの準備

監視結果の投稿先となる Teams チームとチャネルを用意します。

1. **Microsoft Teams** で新しいチームを作成する（例: `PoC-SREAgent`）
   - チームの種類は「プライベート」でも「パブリック」でもよい
2. チャネルを確認する（デフォルトの **General** チャネルを使用可能）

> **注意**: 新しく作成したチームが SRE Agent から認識されるまで数分かかる場合があります。

### 4.3 SRE Agent にリクエスト状況の確認を依頼

1. SRE Agent のスレッドで以下のように指示する:

   ```
   app-sreagent-dev-xxxxxx の過去24時間のリクエスト状況を教えて
   ```

2. SRE Agent がリクエスト数、ステータスコード別の内訳（2xx / 4xx / 5xx）、平均応答時間をチャート付きで返すことを確認する

### 4.4 スケジュールタスクの作成（リクエスト監視 + Teams 通知）

10分ごとにリクエスト状況を取得し、Teams チャネルに通知するスケジュールタスクを作成します。

1. SRE Agent の左メニューから **Automation** を開く
2. ツールバーの **Create** をクリックし、**Scheduled task** を選択する
3. 以下の内容でタスクを設定する:

   | フィールド | 設定値 |
   |---|---|
   | **Task name** | `app-sreagent-dev リクエスト監視 (10分)` |
   | **Task details** | 下記のタスク指示を入力 |
   | **Frequency** | **Custom cron** を選択し、`*/10 * * * *` を入力 |
   | **Agent autonomy level** | **Autonomous** |

4. **Task details** に以下の指示を入力する:

   ```
   app-sreagent-dev-xxxxxx の直近10分間のリクエスト状況を Azure Monitor メトリクスから取得し、
   Teams「PoC-SREAgent」チームの General チャネルに投稿してください。

   取得するメトリクス:
   - Requests（リクエスト数）
   - Http2xx, Http4xx, Http5xx（ステータスコード別）
   - AverageResponseTime（平均応答時間）

   投稿フォーマット:
   - 期間（開始〜終了 UTC）
   - 総リクエスト数
   - ステータスコード別内訳（2xx / 4xx / 5xx）
   - 平均応答時間

   異常検知:
   - 5xx エラー率が 5% を超えた場合は⚠️マーク付きで警告
   - 平均応答時間が 10 秒を超えた場合は⚠️マーク付きで警告

   すべて日本語で記述すること。
   ```

   > **注意**: `app-sreagent-dev-xxxxxx` と `PoC-SREAgent` は実際の App Service 名と Teams チーム名に置き換えてください。

5. **Create task** をクリックする
6. **Automation** 一覧にタスクが表示され、ステータスが **On** になっていることを確認する
7. 10分後に Teams チャネルにリクエスト監視レポートが自動投稿されることを確認する

### 4.5 スケジュールタスクの管理

作成したスケジュールタスクは **Automation** 画面から管理できます。

- **一時停止 / 再開**: タスク行の **⋯** をクリック → **Turn off** / **Turn on** を選択
- **編集**: タスク行の **⋯** をクリック → **Edit** を選択（実行履歴は保持される）
- **削除**: タスク行の **⋯** をクリック → **Delete** を選択
- **実行履歴**: タスク名をクリックすると、各実行の詳細スレッドを確認できる

> **注意**: デモ終了後はスケジュールタスクを削除し、不要な通知の発生を防いでください。

---

## 参考リンク

- [Azure SRE Agent ドキュメント](https://learn.microsoft.com/azure/sre-agent)
- [SRE Agent ポータル](https://sre.azure.com)
- [ソース管理を接続](https://learn.microsoft.com/ja-jp/azure/sre-agent/code-repository-connect?pivots=github)
- [インシデント対応計画の作成](https://learn.microsoft.com/ja-jp/azure/sre-agent/incident-response-plan)
- [カスタムエージェント](https://learn.microsoft.com/ja-jp/azure/sre-agent/custom-agents)
- [スケジュールタスク](https://learn.microsoft.com/ja-jp/azure/sre-agent/scheduled-tasks)
- [Teams コネクタ](https://learn.microsoft.com/ja-jp/azure/sre-agent/send-notifications)