# migration-pipeline

**モノリスからの API 移行を、AI エージェントの分業で最後まで回す。**

<p>
  <img alt="Claude Code" src="https://img.shields.io/badge/Claude_Code-skills_+_subagents-D97757">
  <img alt="対象" src="https://img.shields.io/badge/対象-API移行の実行-2ea44f">
  <img alt="framework-agnostic" src="https://img.shields.io/badge/framework-Laravel_/_Django_/_Rails-6E56CF">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green">
</p>

移行は「エンドポイント数 × 検証項目数」の掛け算になり、1会話では終わりません。
かといって人間が毎回指示を出すと、今度は人間がボトルネックになります。

このリポジトリは、**ドメイン単位で自律的に進み、判断が要る時だけ止まる**
移行ループと、その分業を担う専門エージェント一式です。

- [migration-loop](skills/migration-loop/SKILL.md) — 全ドメインを順に処理するループ
- [migration-pipeline](skills/migration-pipeline/SKILL.md) — 1ドメイン分の手順
- [agents/](agents/) — 専門エージェント9個

## コンテキストを枯渇させない設計

長時間動く AI ワークフローの最大の敵は、**会話が膨らんで動かなくなること**です。
このループは3つの仕掛けでそれを避けます。

- **ドメインごとにエージェントを使い捨て起動する** — 前ドメインの文脈を持ち越さない
- **成果物はファイルで受け渡す** — 会話に全文を載せない
- **メイン会話が保持するのはタグ集計サマリーだけ** — 数十ドメイン流しても溢れない

状態管理はスケジュールファイル1枚に集約し、**会話の記憶に頼りません**。
中断しても、ファイルを読めば続きから再開できます。

### 実装上の制約

サブエージェントは、さらに子エージェントを起動できません。
そのため「まとめ役エージェントの中から専門エージェントを呼ぶ」二重ネストは成立しません。
専門エージェントの起動は**必ずメイン会話から**行い、メイン会話自身がまとめ役を兼ねます。

## 原因タグ — この手順の中心

フロントが呼ぶパスが新システムに無いとき、**「無い」だけでは対処が決まりません。**
フロントを直すのか、新システムに実装するのか、原因によって逆になります。

| タグ | 原因 | 対処 |
|------|------|------|
| `[FE変更漏れ]` | フロントが旧パスを呼んでいる | フロント側を修正 |
| `[BE欠損]` | 新システムに未実装 | 新システムに実装 |
| `[パス変更]` | パスが変わりフロントが未追従 | **ユーザー判断** |
| `[ロジック欠落]` | EPはあるが検証・権限・副作用が欠落 | **ユーザー承認後**に補完 |
| `[正常]` | 一致 | 対処不要 |

原因を切り分けるために、**フロント・旧システム・新システムの3点を突き合わせます**（三角照合）。
2点比較では「フロントが古いのか、新システムが未実装なのか」が判別できません。

## 止まる設計

`[パス変更]` と `[ロジック欠落]` は**自動で解決しません**。どちらも仕様に関わるためです。

ここを自動化すると、**静かに仕様が変わります**。
移行で最も避けたいのは「動いているが、前と違う」状態なので、
判断が要る場面では必ず人間に戻します。

止まるときは理由とエンドポイント一覧を提示し、判断後は同じコマンドで再開できます。

## エージェント構成

| フェーズ | エージェント | 役割 |
|---|---|---|
| 抽出 | `fe-analyst` | フロントが呼んでいるものと**実際に使っているフィールド** |
| 抽出 | `legacy-analyst` | 旧システムの実装（正常系・非正常系） |
| 抽出 | `service-router` | 新システムの既存EPと配置判定 |
| 設計 | `schema-architect` | 三角照合・原因タグ付与・確定契約 |
| 計画 | `planner` | 変更ファイル・倣う既存実装・影響範囲・検証手順を書いた対処計画（実装しない） |
| 実装 | `be-engineer` / `fe-engineer` | 承認済み計画に沿って実装 |
| 検証 | `contract-verifier` | 契約 vs 実装 |
| 検証 | `integration-verifier` | フロントの期待値 vs 実装 |

抽出エージェントは**片側しか読みません**。
新旧を同時に見ると「新側がこうだから旧側もこうだろう」と補完してしまい、
差分検出の意味が失われるためです。

実装と検証も別エージェントに分けています。
実装者が自分で検証すると「実装したとおり」を確認するだけになります。

## 使い方

### 1. スキルとエージェントを入れる

移行作業をするプロジェクトの `.claude/` に置きます（全プロジェクトで使うなら `~/.claude/`）。

```bash
git clone https://github.com/Akinori901/migration-pipeline
cd <移行作業をするプロジェクト>
mkdir -p .claude/skills .claude/agents
cp -r ../migration-pipeline/skills/* .claude/skills/   # migration-loop / migration-pipeline
cp ../migration-pipeline/agents/*.md .claude/agents/    # 専門エージェント 9 個
```

### 2. 設定とスケジュールを書いて起動する

```bash
cp ../migration-pipeline/examples/migration.config.yml ./migration.config.yml
mkdir -p docs/schedules
cp ../migration-pipeline/examples/schedule.md ./docs/schedules/migration.md
# 自社の構成とドメイン一覧に書き換える

# Claude Code から
/migration-loop                 # 未着手のドメインを順に処理する
/migration-pipeline orders      # 1 ドメインだけ流す
```

サービスの分割と責務は `migration.config.yml` に書きます。
`responsibility` が配置判定の根拠になるため、**扱う業務領域を具体的に書いてください**。

## 関連

- [api-migration-diff](https://github.com/Akinori901/api-migration-diff) — 移植済みEPの差分検証
- [clean-arch-laravel](https://github.com/Akinori901/clean-arch-laravel) — 移植先の層構成を deptrac で強制

## ライセンス

MIT
