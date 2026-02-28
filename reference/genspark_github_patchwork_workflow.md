# GitHub Issue駆動「パッチワーク開発」ワークフロー ベストプラクティス調査レポート

> 調査日: 2026-02-28
> 対象: REGAINプログラム受講者（非エンジニア）がAIでアプリを修正・改善するためのワークフロー

---

## 1. 背景と課題

REGAINコーチングサービスの受講者（非エンジニア）が、AIでアプリを開発する際に以下の問題を抱えている：

- エラーが出ると**ゼロから作り直すしかない**
- 既存コードの**特定箇所だけを安全に修正**する方法がわからない
- 何が変わったか把握できず、**元に戻せない**

これらを解決する「パッチワーク開発」手法について、GenSparkに限定せず幅広く調査を行った。

---

## 2. 推奨ワークフロー全体像

```
非エンジニア（受講者）
  │
  ├─ Step 1: GitHub Issueテンプレートで変更リクエスト作成
  │     └─ 構造化フォーム：現状・期待動作・スクリーンショット・優先度
  │
  ├─ Step 2: AIエージェントが自動的にPR作成
  │     └─ Issue → コード変更 → Draft PR
  │
  ├─ Step 3: 自動チェックが実行される
  │     ├─ CI/CDテスト（GitHub Actions）
  │     ├─ プレビューデプロイ（Vercel/Netlify）
  │     └─ AIコードレビュー
  │
  ├─ Step 4: 受講者がプレビューURLで目視確認
  │     └─ OK → マージ / NG → Issueにコメントで追加指示
  │
  └─ Step 5: 問題発生時のロールバック
        ├─ 即時：Vercelダッシュボードで前バージョンに復帰
        └─ 恒久：GitHubのRevertボタンでコードを戻す
```

---

## 3. AIツール比較：Issue → PR 自動生成

### 3-1. ツール一覧と特徴

| ツール | Issue→PR自動生成 | レビュー自動化 | 非エンジニア向け | 操作方法 | 料金目安 |
|--------|:---:|:---:|:---:|------|------|
| **GitHub Copilot Coding Agent** | ✅ | ✅ | ◎ 最も簡単 | Issueの担当者に「Copilot」を選択 | Copilot Pro以上（$10〜/月） |
| **Claude Code Action** | ✅ | ✅ | ◎ | Issue/PRで `@claude` メンション | APIキー費用（従量課金） |
| **OpenAI Codex** | ✅ | ✅ | ◎ | Issue/PRで `@codex` メンション | ChatGPT Plus以上（$20/月） |
| **Sweep AI** | ✅ | △ | ○ | Issueタイトルに「Sweep:」を付与 | 無料枠あり |
| **Devin AI** | ✅ | ✅ | △ | Webダッシュボード経由 | $20〜$500/月 |
| **aider (GitHub Action)** | ✅ | ✗ | △ | Issueに `aider` ラベル付与 | 無料（LLM API費用別） |
| **CodeRabbit** | ✗（レビュー専門） | ✅ | ○ | PR自動レビュー | 無料枠あり |
| **GenSpark AI Developer** | ✅ | △ | ○ | GenSpark UIからGitHub連携 | GenSparkプラン依存 |

### 3-2. 各ツール詳細

#### GitHub Copilot Coding Agent（最も推奨）

- **最も非エンジニアに適している**: Issueを作成し、担当者に「Copilot」を選択するだけ
- Copilotがセキュアな GitHub Actions 環境でリポジトリをクローン→セマンティックコード検索→修正計画→ドラフトPR作成→テスト実行→レビュー依頼まで自律的に実行
- 自身の変更を Copilot Code Review でセルフレビューしてから PR を作成
- フィードバックをPRコメントで残すと、Copilotが修正を反復
- **Mission Control**で複数タスクを一つのダッシュボードから管理可能
- 2025年9月に一般提供開始（GA）

Sources:
- [About GitHub Copilot coding agent - GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)
- [Asking Copilot to create a PR - GitHub Docs](https://docs.github.com/copilot/using-github-copilot/coding-agent/asking-copilot-to-create-a-pull-request)
- [5 ways to integrate coding agent - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/5-ways-to-integrate-github-copilot-coding-agent-into-your-workflow/)
- [Copilot coding agent 101 - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/github-copilot-coding-agent-101-getting-started-with-agentic-workflows-on-github/)

#### Claude Code Action（柔軟性が高い）

- `@claude` でIssueやPRにメンションすると、コード分析→変更実装→PR作成まで実行
- GitHub Actions v1.0で**モード自動検出**（Issue/PR/レビュー）
- セットアップ: Claude Codeターミナルで `/install-github-app` を実行するだけ
- カスタムスキル（`.claude/skills/`）でワークフローを拡張可能
- AWS Bedrock、Google Vertex AI、Microsoft Foundryにも対応

基本的なGitHub Actionsワークフロー:
```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Sources:
- [Claude Code GitHub Actions - 公式ドキュメント](https://code.claude.com/docs/en/github-actions)
- [anthropics/claude-code-action - GitHub](https://github.com/anthropics/claude-code-action)
- [Claude Code Action - GitHub Marketplace](https://github.com/marketplace/actions/claude-code-action-official)

#### OpenAI Codex

- `@codex` でIssueやPRにメンション可能
- Codex GitHub Action（`openai/codex-action@v1`）でCI/CDジョブに統合可能
- JiraとGitHubの間のワークフロー自動化もサポート
- 2025年6月にChatGPT Plusユーザーに提供開始

Sources:
- [Use Codex in GitHub - OpenAI](https://developers.openai.com/codex/integrations/github/)
- [Codex GitHub Action - OpenAI](https://developers.openai.com/codex/github-action/)

#### GenSpark AI Developer（参考）

- GitHub App（[genspark-ai-developer](https://github.com/apps/genspark-ai-developer)）を通じてリポジトリ連携
- GenSpark UIから自然言語でコード修正→コミット or PR作成を選択
- Diff表示で変更箇所を確認可能
- GenSpark 2.0ではiOS/Androidネイティブアプリにも対応

Sources:
- [GenSpark公式サイト](https://www.genspark.ai/)
- [GenSpark AI Developer GitHub App](https://github.com/apps/genspark-ai-developer)

---

## 4. Issue テンプレート設計（非エンジニア向け）

GitHub Issue Formsを使うことで、非エンジニアが迷わず必要情報を提供できるフォームを作成できる。

### 4-1. バグ報告テンプレート例

ファイル: `.github/ISSUE_TEMPLATE/bug_report.yml`

```yaml
name: バグ報告
description: 不具合を報告してください
title: "[バグ]: "
labels: ["bug", "triage"]
body:
  - type: markdown
    attributes:
      value: |
        バグ報告にご協力いただきありがとうございます。
        以下の項目をできるだけ詳しく記入してください。

  - type: textarea
    id: current-behavior
    attributes:
      label: 現在の動作
      description: 今どのような問題が起きていますか？
      placeholder: 例）ボタンをクリックしてもページが遷移しない
    validations:
      required: true

  - type: textarea
    id: expected-behavior
    attributes:
      label: 期待する動作
      description: 本来どのように動くべきですか？
      placeholder: 例）ボタンをクリックすると設定ページに遷移する
    validations:
      required: true

  - type: textarea
    id: steps-to-reproduce
    attributes:
      label: 再現手順
      description: 問題を再現するための手順を記入してください
      placeholder: |
        1. ○○ページにアクセス
        2. △△ボタンをクリック
        3. エラーが表示される
    validations:
      required: true

  - type: textarea
    id: screenshots
    attributes:
      label: スクリーンショット
      description: 問題がわかるスクリーンショットを貼り付けてください（ドラッグ&ドロップで添付可能）
    validations:
      required: false

  - type: dropdown
    id: urgency
    attributes:
      label: 緊急度
      options:
        - 低（いつでも対応可能）
        - 中（近いうちに対応が必要）
        - 高（業務に支障がある）
        - 緊急（サービスが停止している）
      default: 0
    validations:
      required: true
```

### 4-2. 機能変更リクエストテンプレート例

ファイル: `.github/ISSUE_TEMPLATE/feature_request.yml`

```yaml
name: 変更・機能リクエスト
description: アプリの変更や新しい機能をリクエストしてください
title: "[リクエスト]: "
labels: ["enhancement", "triage"]
body:
  - type: textarea
    id: what-to-change
    attributes:
      label: 変更したい内容
      description: 何をどう変えたいですか？具体的に記述してください
    validations:
      required: true

  - type: textarea
    id: why
    attributes:
      label: 変更の理由・背景
      description: なぜこの変更が必要ですか？
    validations:
      required: true

  - type: textarea
    id: screenshots-mockup
    attributes:
      label: 参考画像・モックアップ
      description: イメージがわかる画像があれば添付してください
    validations:
      required: false

  - type: dropdown
    id: priority
    attributes:
      label: 優先度
      options:
        - 低い
        - 普通
        - 高い
    validations:
      required: true
```

Sources:
- [Syntax for issue forms - GitHub Docs](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms)
- [Configuring issue templates - GitHub Docs](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository)

---

## 5. ブランチ保護と安全性

### 5-1. GitHub Rulesets（推奨）

2025-2026年のベストプラクティスでは、従来のBranch Protection Rulesよりも**GitHub Rulesets**が推奨されている。

| 項目 | Branch Protection Rules | Rulesets |
|------|------|------|
| 適用範囲 | リポジトリ単位、ブランチ単位 | 組織全体で複数リポジトリに適用可能 |
| 対象 | ブランチのみ | ブランチ、タグ、リポジトリイベント |
| 複数ルール | 1つのルールのみ | 複数ルールが同時適用（最も厳格なものが優先） |
| 管理 | 各リポジトリで個別設定 | 中央集権的なガバナンス |

### 5-2. mainブランチの推奨最小設定

| 設定項目 | 推奨値 | 理由 |
|---------|--------|------|
| Restrict deletions | 有効 | mainブランチの削除を防止 |
| Require a pull request before merging | 有効 | 直接pushを防止、必ずPR経由に |
| Required approving reviews | 0〜1人 | ソロ開発でもPR作成→確認フローを強制 |
| Require status checks to pass | **有効（最重要）** | CIテスト・ビルドの通過を必須化 |
| Block force pushes | 有効 | 履歴の改変を防止 |

### 5-3. AI駆動PRに対するセキュリティ対策

- コメントにシークレットを含めることを禁止
- 保護ブランチへの直接編集を禁止
- 最小権限の原則: 読み取り専用から開始し、単一ブランチへの書き込みのみ追加

Sources:
- [About rulesets - GitHub Docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Branch Protection Rules vs Rulesets - DEV Community](https://dev.to/piyushgaikwaad/branch-protection-rules-vs-rulesets-the-right-way-to-protect-your-git-repos-305m)

---

## 6. 自動テスト・プレビューデプロイ

### 6-1. Vercelプレビューデプロイ（最も簡単）

VercelのGitHub連携を有効にするだけで、すべてのPRに自動的にプレビューURLが生成される。Vercel botがPRコメントにプレビューURLを投稿するため、非エンジニアはURLをクリックして変更結果を目視確認できる。

### 6-2. AIコードレビュー自動実行

```yaml
name: Claude Auto Review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Please review this pull request with a focus on:
            - Code quality and best practices
            - Potential bugs or issues
            - Security implications
```

**非エンジニアにとってのメリット**: PRを作成するだけで、プレビューURL（目視確認）+ AIレビュー（品質確認）が自動実行される。

Sources:
- [Deploying GitHub Projects with Vercel](https://vercel.com/docs/git/vercel-for-github)
- [Claude Code Action - GitHub](https://github.com/anthropics/claude-code-action)

---

## 7. ロールバック戦略

### 7-1. GitHubの「Revert」ボタン（最も簡単）

マージ済みPRのページに表示される「Revert」ボタンが最も簡単。

1. 問題を起こしたマージ済みPRのページを開く
2. 「Revert」ボタンをクリック
3. 自動的にリバートPRが作成される
4. リバートPRをマージする

### 7-2. Vercel/Netlifyのインスタントロールバック

Vercelの場合、ダッシュボードから過去のデプロイメントを選んで「Promote to Production」をクリックするだけで、即座に前バージョンに復帰。Git操作不要で、非エンジニアに最も優しい方法。

### 7-3. 推奨フロー

```
何か壊れた！
  ↓
まず：Vercel/Netlifyダッシュボードで前のデプロイに戻す（即時復旧）
  ↓
次に：GitHubで該当PRの「Revert」ボタンを押す（コードも戻す）
  ↓
リバートPRのテスト通過を確認 → マージ
```

Sources:
- [Reverting a pull request - GitHub Docs](https://docs.github.com/articles/reverting-a-pull-request)
- [Undoing a Pull Request Safely - Reliable Penguin](https://blogs.reliablepenguin.com/2025/10/08/undoing-a-pull-request-safely-revert-vs-reset-with-github-cli)

---

## 8. Claude Code Skillsによるカスタムワークフロー

Claude Codeでは `.claude/skills/` にカスタムスキルを配置することで、ワークフローを拡張できる。

### 8-1. Issue修正スキル例

`.claude/skills/fix-issue/SKILL.md`:

```yaml
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue description
2. Understand the requirements
3. Implement the fix
4. Write tests
5. Create a commit
```

`/fix-issue 123` と実行すると、Claude が Issue #123 を読み、修正を実装する。

### 8-2. PRレビュースキル例

`.claude/skills/review-pr/SKILL.md`:

```yaml
---
name: review-pr
description: Review the current pull request
disable-model-invocation: true
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Review instructions
1. Analyze all changed files
2. Check for bugs, security issues, and style violations
3. Provide specific, actionable feedback
```

### 8-3. その他の人気カスタムスキル

| コマンド | 機能 |
|----------|------|
| `/commit` | diffを分析し、Conventional Commitメッセージを生成してコミット |
| `/pr` | ブランチ上の全コミットを分析し、PRタイトルと説明を生成 |
| `/push` | ステージング→コミット→プッシュを一括実行 |
| `/fix-issue [番号]` | GitHub Issueを読み、修正を実装 |

Sources:
- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Slash commands - Claude Code Docs](https://code.claude.com/docs/en/slash-commands)
- [awesome-claude-code - GitHub](https://github.com/hesreallyhim/awesome-claude-code)

---

## 9. Spec駆動開発（上級者向け参考情報）

GitHubが2025年に推進している新しいアプローチ。`.spec.md`ファイルで機能仕様を定義し、AIエージェントへの入力として使用する。

- 仕様を明確にすることで、AIが生成するコードが既存プロジェクトに馴染むものになる
- [Spec Kit](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) というオープンソースツールキットが提供されている
- Addy Osmani氏（Google Chromeチーム）も「15分でウォーターフォール」として推奨

Sources:
- [Spec-driven development with AI - GitHub Blog](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [How to write a good spec for AI agents - Addy Osmani](https://addyosmani.com/blog/good-spec/)

---

## 10. 「ゼロから作り直す」問題の解決まとめ

| 現状の問題 | Issue駆動ワークフローでの解決 |
|---|---|
| エラーが出ると全部作り直し | mainブランチに安定版があり、いつでも戻せる |
| 修正したい箇所だけ変更できない | Issueに自然言語で指示→AIが該当箇所のみ修正 |
| 何が変わったか分からない | PRのDiff表示で変更箇所を視覚的に確認 |
| 壊れたら元に戻せない | Revertボタン1クリック / Vercelで即時ロールバック |
| AIに全部お任せで品質が不安 | 自動テスト + AIレビュー + プレビューデプロイの3重チェック |

---

## 11. REGAINプログラムへの推奨導入方針

### Week 3-4で導入する場合の段階

1. **GitHub基礎**（30分）: アカウント作成、リポジトリの概念、mainブランチ = 安全な場所
2. **Issue作成体験**（20分）: テンプレートを使って実際にIssueを作成
3. **AI連携設定**（15分）: Copilot / Claude Code Actionの初期設定（エンジニアがサポート）
4. **パッチワーク修正実践**（30分）: 実際にIssueを作成→AIがPR作成→プレビュー確認→マージ
5. **ロールバック体験**（15分）: わざと壊す→Revertで戻す体験

### 推奨ツール選定

| 優先度 | ツール | 理由 |
|--------|--------|------|
| 第1候補 | **GitHub Copilot Coding Agent** | 最もシンプル。Issueに担当者を選ぶだけ |
| 第2候補 | **Claude Code Action** | カスタマイズ性が高く、APIキー管理で柔軟 |
| 補助 | **Vercel** | プレビューデプロイで目視確認 |
| 補助 | **CodeRabbit** | AIレビューの追加レイヤー |

---

## Sources（全体）

### GitHub公式
- [About GitHub Copilot coding agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)
- [Asking Copilot to create a PR](https://docs.github.com/copilot/using-github-copilot/coding-agent/asking-copilot-to-create-a-pull-request)
- [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Syntax for issue forms](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms)
- [Reverting a pull request](https://docs.github.com/articles/reverting-a-pull-request)
- [Spec-driven development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)

### Claude Code
- [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Slash Commands](https://code.claude.com/docs/en/slash-commands)

### OpenAI
- [Use Codex in GitHub](https://developers.openai.com/codex/integrations/github/)
- [Codex GitHub Action](https://developers.openai.com/codex/github-action/)

### その他
- [GenSpark公式](https://www.genspark.ai/)
- [My LLM coding workflow going into 2026 - Addy Osmani](https://addyosmani.com/blog/ai-coding-workflow/)
- [How to write a good spec for AI agents - Addy Osmani](https://addyosmani.com/blog/good-spec/)
- [awesome-claude-code - GitHub](https://github.com/hesreallyhim/awesome-claude-code)
- [Branch Protection Rules vs Rulesets - DEV Community](https://dev.to/piyushgaikwaad/branch-protection-rules-vs-rulesets-the-right-way-to-protect-your-git-repos-305m)
- [Deploying GitHub Projects with Vercel](https://vercel.com/docs/git/vercel-for-github)
- [From GitHub Issue to PR - Coder Tasks](https://coder.com/blog/launch-dec-2025-coder-tasks)
- [Pick your agent: Use Claude and Codex on Agent HQ - GitHub Blog](https://github.blog/news-insights/company-news/pick-your-agent-use-claude-and-codex-on-agent-hq/)
