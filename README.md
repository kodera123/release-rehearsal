# verifying-release

リリース前に「このまま本番に出して大丈夫か」を、エビデンス付きで判定する [Claude Code](https://claude.com/claude-code) 用の Agent Skill です。

結論は **Go / 条件付き Go / No-Go** のどれかで、必ず根拠を添えて返します。

> A Claude Code skill for pre-release verification. It builds a risk-driven test list from the diff, has an independent subagent check it for gaps, runs it while saving screenshots and logs as files, reproduces production-like "dirty data" before migrating, and returns a Go / Conditional Go / No-Go verdict with evidence. The skill text is written in Japanese.

## 何をするか

普通の動作確認は「新機能が動くか」を見ます。このスキルはそれに加えて、**本番にしかない条件で壊れないか**を重点的に見ます。

- 本番に以前から溜まっている汚いデータ（重複・NULL・大文字小文字違い・古い区分値・論理削除済み）
- デプロイ中に新旧のコードが混在する状態、ロールバック時
- デプロイ前から溜まっているジョブ・retry 中のジョブ
- 日時・タイムゾーン・期間の境界（毎回確認）

### 流れ

| 段階 | やること |
|---|---|
| 0. 前提確認 | 対象の差分（本番 → リリース）を確定。デプロイ方式・DB バージョン・外部利用者・タイムゾーンなど、推測で埋めてはいけない前提はまとめて質問する。所要時間も見積もる |
| 1. 差分の棚卸し | 変更ファイルを 1 つ残らず種別（migration / API / UI / ジョブ / 設定 …）に分類する |
| 2. 確認リスト作成 | 「変更 → 想定リスク → 確認項目」の順で逆算する。種別ごとのリスクカタログを使う |
| 3. 十分性レビュー | リストを見せていない別のサブエージェントに、差分だけから「本番で壊れうるシナリオ」を挙げさせて突き合わせる |
| 4. 実行 | 1 項目ずつ確認し、スクショとログを**ファイルとして**保存する。結果は OK / NG / 未実施（理由） |
| 5. 既存データの影響判定 | 本番と同じコードで DB を作って汚いデータを入れ、リリース版で migrate して確認し直す。本番で確かめる必要がある点は read-only の確認クエリにまとめる |
| 6. レポート | 判定・リリース前に満たす条件・不具合が出る条件・未確認事項を `report.md` にまとめる |

### 特徴

- **判定が機械的に決まる。** 未修正の NG が残っていれば No-Go。未確認の点があっても、「何を確認して、どういう結果なら Go か」と回避策が書けていれば条件付き Go。何も残っていなければ Go。急いでいても、未確認を Go に丸めない
- **作った本人が十分性を自己採点しない。** 独立したサブエージェントのレビューと突き合わせる
- **「seed で migrate が通ったから OK」を認めない**
- **本番には触らない。** 本番で確認が要る点は、人が実行する read-only のクエリとして渡す
- **検証中に見つけた不具合を黙って直さない。** NG として記録し、直すかは人に諮る
- **できなかった項目を消さない。** 未実施のまま残し、判定に反映する

## 前提・想定

- **Claude Code**（Agent Skills が使える環境）。独立レビューにサブエージェントを使う
- **git** で管理された repo と、本番にデプロイ済みの ref（タグ / ブランチ / SHA）
- **スクショをファイルに保存できる手段**: 例 [chrome-devtools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp) の `take_screenshot`（`filePath` 指定）、Playwright。画像がエージェントに返るだけのスクショはエビデンスになりません
- リスクカタログの SQL やジョブの例は **PostgreSQL / Rails / Sidekiq** を想定しています。他のスタックでは読み替えてください
- 性能確認（N+1・実行計画・インデックス）はオプションです。前提確認のときに含めるかを聞きます

## インストール

```bash
git clone https://github.com/kodera123/verifying-release.git
```

```bash
cp -R verifying-release/skills/verifying-release ~/.claude/skills/
```

プロジェクト単位で使う場合は、`~/.claude/skills/` の代わりに `<project>/.claude/skills/` に置きます。

## 使い方

Claude Code で次のように頼むと起動します。

```
/verifying-release v1.4.0..release/1.5.0 をリリース前検証して
```

「リリース前チェック」「このままリリースして大丈夫？」「既存データに影響ある？」「デグレしない？」などでも起動します。

エビデンスとレポートは `<repo>/tmp/release-verification/<YYYYMMDD>-<ref>/` に保存されます（gitignore されていない場合は `~/release-verification/<repo名>/...`）。

```
<保存先>/
├── report.md        # 判定・影響表・確認リスト・十分性の根拠・未確認事項
├── screenshots/     # <ID>-<説明>.png
└── logs/            # 画面に出ないもの（migrate・API・ジョブ・grep 結果）
```

## ファイル構成

```
skills/verifying-release/
├── SKILL.md                    # 手順・鉄則・判定基準
└── reference/
    ├── risk-catalog.md         # 変更種別ごとのリスク・確認方法・汚いデータ・本番確認クエリ
    └── report-template.md      # report.md の雛形
```

## カスタマイズ

- **リスクカタログ**（`reference/risk-catalog.md`）に、自分のスタックやプロダクト固有の壊れ方を足すのが一番効果があります
- 日時の洗い出しに使う grep パターンは Ruby / JS / SQL の一般的な名前を想定しています。独自の日時ユーティリティがあれば足してください

## ライセンス

[MIT](LICENSE)
