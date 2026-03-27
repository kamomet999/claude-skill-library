---
name: skill-unload
description: 使い終わったスキルをローカルから削除してコンテキストを節約する
category: ユーティリティ
command: /skill-unload
version: 1.0.0
tags:
  - meta
  - cleanup
  - library
  - unload
---

# Skill Unload - スキル削除

使い終わったスキルをローカル (`~/.claude/skills/`) から削除し、コンテキストを節約します。スキル本体はGitHubライブラリに残っているので、再度必要になったら `/skill-pull` で取得できます。

## 使い方

```
/skill-unload <skill-name>
/skill-unload postgres-patterns
/skill-unload golang-patterns golang-testing   # 複数同時OK
```

## 実行フロー

### Step 1: 引数の確認

引数がない場合はエラー:
```
使い方: /skill-unload <skill-name> [<skill-name2> ...]
```

### Step 2: 存在確認

```bash
ls ~/.claude/skills/<skill-name>/SKILL.md 2>/dev/null
```

存在しない場合:
```
⚠ <skill-name> はローカルにありません。
```

### Step 3: 保護チェック

以下のスキルは**削除禁止**（システムに必要）:
- `skill-finder` — スキル検索の起点
- `skill-library` — ライブラリ辞書検索
- `skill-pull` — スキル取得
- `skill-unload` — このスキル自身

保護スキルが指定された場合:
```
⛔ <skill-name> はシステムスキルのため削除できません。
```

### Step 4: 削除実行

```bash
rm -rf ~/.claude/skills/<skill-name>
```

### Step 5: 結果報告

```markdown
## Skill Unload Complete

| スキル | ステータス |
|--------|-----------|
| {name} | 削除完了 |

再度必要になったら `/skill-pull <skill-name>` で取得できます。
```

## 注意事項

- ライブラリにバックアップがあるスキルのみ安全に削除できる
- ライブラリに**ない**ローカル専用スキルを削除しようとした場合は警告する:
  ```
  ⚠ <skill-name> はライブラリに存在しません。削除すると復元できません。本当に削除しますか？
  ```
- claude-code-skills（既存の自動同期リポ）にあるスキルは同期で復元されるため問題なし
- 削除後、次のセッションでsystem-reminderのスキル一覧から消える
