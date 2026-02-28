# fix-issue スキル：AIエージェントによるIssue修正ワークフロー

> このドキュメントは、AIエージェント（GenSpark AI Developer / GitHub Copilot / Claude Code 等）が
> GitHub Issueに基づいてコードを修正する際のステップバイステップフローを定義する。
> GenSparkの `agent.md` やClaude Codeの `SKILL.md`、GitHub Copilotの `AGENTS.md` にそのまま転用可能。

---

## 全体フロー概要

```
Issue作成（人間）
  ↓
Phase 1: Issue読み取り・理解
  ↓
Phase 2: コードベース探索
  ↓
Phase 3: 修正計画の策定
  ↓
Phase 4: 実装
  ↓
Phase 5: テスト・検証
  ↓
Phase 6: セルフレビュー
  ↓
Phase 7: コミット・PR作成
  ↓
Phase 8: レビュー対応・イテレーション
  ↓
マージ（人間の承認）
```

---

## Phase 1: Issueの読み取りと理解

### やること
1. GitHub IssueのタイトルとDescriptionを読み取る
2. ラベル（bug / enhancement / etc.）を確認し、修正の種類を把握する
3. コメント欄に追加情報やスクリーンショットがないか確認する
4. 受け入れ基準（Acceptance Criteria）が明記されている場合はそれを優先する
5. 関連するIssueやPRがリンクされていれば、それも確認する

### 判断ポイント
- **Issueの内容が曖昧な場合** → 作業を中断し、Issueコメントで質問する（推測で実装しない）
- **スコープが大きすぎる場合** → 分割を提案する

### 使用コマンド例
```bash
gh issue view <issue-number>
gh issue view <issue-number> --comments
```

---

## Phase 2: コードベースの探索と理解

### やること
1. プロジェクト全体の構造を把握する（ディレクトリ構成、主要ファイル）
2. Issueに関連するファイルを特定する
   - キーワード検索（Grep）でIssueに記載された機能名・エラーメッセージ等を検索
   - ファイル名パターン検索（Glob）で関連コンポーネントを特定
3. 関連ファイルの内容を読み、現在のコードの動作を理解する
4. 既存テストを確認し、テストの書き方のパターンを把握する
5. プロジェクトの規約ファイルを確認する
   - `CLAUDE.md` / `AGENTS.md` / `copilot-instructions.md`
   - `README.md` / `CONTRIBUTING.md`
   - `.eslintrc` / `tsconfig.json` 等の設定ファイル

### 判断ポイント
- **関連ファイルが見つからない場合** → 検索範囲を広げる。それでも見つからなければ質問する
- **依存関係が複雑な場合** → 影響範囲をリストアップしてから次へ進む

---

## Phase 3: 修正計画の策定

### やること
1. 変更が必要なファイルをリストアップする
2. 各ファイルへの具体的な変更内容を記述する
3. 影響範囲（blast radius）を評価する
   - この変更で壊れる可能性がある箇所はどこか？
   - 他の機能への副作用はないか？
4. エッジケースを洗い出す
5. 計画を明示的にテキストとして出力する（後でレビューできるように）

### 計画テンプレート
```
## 修正計画: Issue #<number>

### 概要
<1-2文で変更の概要>

### 変更ファイル
1. `src/components/Header.tsx` — ヘッダーの背景色を変更
2. `src/styles/theme.ts` — カラー定数を追加

### 影響範囲
- Header コンポーネントを使用している全ページ

### エッジケース
- ダークモード時の表示
- モバイル表示時のレスポンシブ対応

### テスト方針
- 既存のHeader関連テストを更新
- 新しいカラーのスナップショットテストを追加
```

### 判断ポイント
- **変更ファイルが5つ以上になる場合** → Issueの分割を検討
- **アーキテクチャ変更を伴う場合** → 人間の承認を得てから実装

---

## Phase 4: 実装

### やること
1. 新しいブランチを作成する
   - 命名規則: `fix/issue-<number>` または `feat/issue-<number>`
2. 計画に従い、**小さな単位で**変更を実施する
3. プロジェクトのコーディング規約に従う
   - インデント、命名規則、import順序など
4. 1つのIssueに対して1つのPR（スコープを限定する）

### 禁止事項
- mainブランチに直接コミットしない
- Issueのスコープ外の「ついで修正」をしない
- シークレット・クレデンシャルをコードに含めない
- `node_modules/` `dist/` `vendor/` 等の自動生成ディレクトリを変更しない
- 関係のないファイルのフォーマット変更をしない

### 使用コマンド例
```bash
git checkout -b fix/issue-<number>
```

---

## Phase 5: テストと検証

### やること
1. **既存テストスイートを実行** — 回帰テスト（変更で既存機能が壊れていないか確認）
2. **新しいテストを追加** — 修正内容に対応するテストを書く
3. **リンター / フォーマッターを実行** — コードスタイルの自動チェック
4. **型チェックを実行** — TypeScriptの場合は `tsc --noEmit`
5. **テスト失敗時** → 原因を調査し修正してから次へ（テストをスキップしない）

### 使用コマンド例
```bash
npm test                    # テスト実行
npm run lint                # リンター実行
npm run type-check          # 型チェック
npx prettier --check .      # フォーマットチェック
```

### 判断ポイント
- **テストが存在しないプロジェクトの場合** → 最低限の動作確認（ビルド成功）を確認
- **テストの書き方が不明な場合** → 既存テストのパターンに倣う

---

## Phase 6: セルフレビュー

### やること
1. `git diff` でコード差分を確認する
2. 以下のチェックリストを確認する：

```
□ 変更がIssueの要件を満たしているか
□ 不要なファイル変更が含まれていないか
□ デバッグ用のconsole.logやprint文が残っていないか
□ シークレットやAPIキーが含まれていないか
□ ハードコードされた値がないか（設定ファイルや環境変数にすべきでないか）
□ エッジケースが考慮されているか
□ 変更のサイズが妥当か（大きすぎないか）
```

3. 問題があればPhase 4に戻って修正

---

## Phase 7: コミットとPR作成

### コミット
1. 変更をステージングする（`git add`）
2. 明確なコミットメッセージを作成する
   - フォーマット: `fix: <修正内容の説明> (#<issue-number>)`
   - 例: `fix: ヘッダー背景色の変更 (#42)`

### PR作成
1. ブランチをリモートにプッシュする
2. PRを作成する。PRの本文には以下を含める：

```markdown
## 概要
<変更内容の要約（1-3文）>

## 変更点
- <具体的な変更内容1>
- <具体的な変更内容2>

## テスト
- [ ] 既存テスト通過
- [ ] 新規テスト追加
- [ ] 目視確認

## 関連Issue
Closes #<issue-number>
```

3. `Closes #<issue-number>` を記載することで、PRマージ時にIssueが自動クローズされる

### 使用コマンド例
```bash
git add <files>
git commit -m "fix: 修正内容 (#<issue-number>)"
git push origin fix/issue-<number>
gh pr create --title "fix: 修正内容" --body "Closes #<issue-number>"
```

---

## Phase 8: レビュー対応とイテレーション

### やること
1. PRにレビューコメントが付いたら内容を確認する
2. フィードバックに基づいて修正を実施する
3. 修正後、テストを再実行する
4. 追加コミットをプッシュする
5. レビューアーに再レビューを依頼する
6. 承認が得られたらマージ（人間が実行）

### 判断ポイント
- **レビューで根本的な設計変更を求められた場合** → Phase 3に戻って再計画
- **スコープの追加を求められた場合** → 別Issueとして切り出すことを提案

---

## 安全のための絶対ルール

| カテゴリ | ルール |
|---------|--------|
| **ブランチ** | mainブランチに直接コミット・プッシュしない |
| **スコープ** | 1 Issue = 1 PR。スコープ外の変更は別Issueにする |
| **テスト** | テストをスキップしない。失敗したまま先に進まない |
| **シークレット** | APIキー・パスワード・トークンをコミットしない |
| **不明点** | 推測で実装しない。不明な点はIssueコメントで質問する |
| **破壊的変更** | 既存APIの変更・データベーススキーマの変更は人間の承認を得る |
| **ファイル** | 自動生成ファイル（node_modules, dist, .env等）をコミットしない |

---

## GenSpark agent.md 向け簡易版

GenSparkの `agent.md` に貼り付けて使用する場合は、以下の簡易版を使用してください：

```markdown
# Agent Instructions

あなたはGitHub Issueに基づいてコードを修正するAIエージェントです。
以下の手順に必ず従ってください。

## 修正フロー

1. **Issue確認**: Issueの内容を完全に理解する。不明点があれば質問する
2. **コード探索**: 関連ファイルを特定し、現在の動作を理解する
3. **計画作成**: 変更ファイルと変更内容を事前にリストアップする
4. **実装**: 新しいブランチで作業する。スコープ外の変更はしない
5. **テスト**: 既存テスト通過 + 新規テスト追加。テスト失敗のまま先に進まない
6. **PR作成**: `Closes #<issue-number>` を含むPRを作成する

## 禁止事項
- mainブランチへの直接コミット
- 推測による実装（不明点は必ず質問）
- シークレットのコミット
- Issueスコープ外の変更
- テストのスキップ
```

---

## Sources

- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Skill authoring best practices - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [About GitHub Copilot coding agent - GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)
- [From idea to PR - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/from-idea-to-pr-a-guide-to-github-copilots-agentic-workflows/)
- [How to write a great agents.md - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
- [AGENTS.md specification](https://agents.md/)
- [Guardrails for Agentic Coding](https://jvaneyck.wordpress.com/2026/02/22/guardrails-for-agentic-coding-how-to-move-up-the-ladder-without-lowering-your-bar/)
- [Tests Are Everything in Agentic AI](https://dev.to/htekdev/tests-are-everything-in-agentic-ai-building-devops-guardrails-for-ai-powered-development-2onl)
- [awesome-claude-code - GitHub](https://github.com/hesreallyhim/awesome-claude-code)
