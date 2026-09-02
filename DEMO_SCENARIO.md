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

## 参考リンク

- [Azure SRE Agent ドキュメント](https://learn.microsoft.com/azure/sre-agent)
- [SRE Agent ポータル](https://sre.azure.com)
- [ソース管理を接続](https://learn.microsoft.com/ja-jp/azure/sre-agent/code-repository-connect?pivots=github)
- [インシデント対応計画の作成](https://learn.microsoft.com/ja-jp/azure/sre-agent/incident-response-plan)
- [カスタムエージェント](https://learn.microsoft.com/ja-jp/azure/sre-agent/custom-agents)