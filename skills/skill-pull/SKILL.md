---
name: skill-pull
description: GitHubのスキルライブラリから指定スキルをローカルにインストールする
category: ユーティリティ
command: /skill-pull
version: 1.0.0
tags:
  - meta
  - install
  - library
  - pull
---

# Skill Pull - スキル取得

GitHubのスキルライブラリから指定されたスキルをローカル (`~/.claude/skills/`) にインストールします。

## 使い方

```
/skill-pull <skill-name>
/skill-pull postgres-patterns
/skill-pull golang-patterns golang-testing   # 複数同時OK
```

## 実行フロー

### Step 1: 引数の確認

引数がない場合はエラー:
```
使い方: /skill-pull <skill-name> [<skill-name2> ...]
利用可能なスキルは /skill-library で確認できます。
```

### Step 2: ローカル重複チェック

```bash
# 既にインストール済みか確認
ls ~/.claude/skills/<skill-name>/SKILL.md 2>/dev/null
```

既にある場合:
```
⚠ <skill-name> は既にローカルにあります。上書きしますか？
```
ユーザーに確認してから進む。

### Step 3: ライブラリからコピー

ローカルにライブラリのクローンがある場合（推奨）:
```bash
cp -r ~/Dev/claude-skill-library/skills/<skill-name> ~/.claude/skills/<skill-name>
```

ローカルにクローンがない場合、GitHubから直接取得:
```bash
# スキルのSKILL.mdをfetch
mkdir -p ~/.claude/skills/<skill-name>
curl -sL "https://raw.githubusercontent.com/kamomet999/claude-skill-library/master/skills/<skill-name>/SKILL.md" \
  -o ~/.claude/skills/<skill-name>/SKILL.md
```

スキルに追加ファイル（config.json, scripts/等）がある場合も全てコピーする。

### Step 4: 確認

```bash
ls -la ~/.claude/skills/<skill-name>/
```

### Step 5: 結果報告

```markdown
## Skill Pull Complete

| スキル | ステータス |
|--------|-----------|
| {name} | インストール完了 |

次のセッションまたはリロード後に `/skill-name` コマンドが使えます。
使い終わったら `/skill-unload <skill-name>` で削除できます。
```

## 注意事項

- ライブラリに存在しないスキル名が指定された場合はエラーを返す
- ローカルクローン (`~/Dev/claude-skill-library/`) がある場合はそちらを優先（高速）
- クローンがない場合はGitHub raw URLからfetch
- スキルディレクトリ内の全ファイルをコピーする（SKILL.md以外にもconfig.json等がある場合）
