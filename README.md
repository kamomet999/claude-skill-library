# Claude Skill Library

Claude Codeスキルの大規模ライブラリ。必要なスキルだけ `skill-pull` でローカルに取得し、使い終わったら `skill-unload` で削除する。

## 使い方

```
skill-finder (ローカル検索)
    ↓ 見つからない
skill-library (catalog.md で辞書検索)
    ↓ 有用なスキル発見
skill-pull <skill-name> (ライブラリからローカルへ)
    ↓ 使用
skill-unload <skill-name> (ローカルから削除)
```

## 構成

```
claude-skill-library/
├── catalog.md          # 辞書（スキル名+説明+タグ）
├── README.md
└── skills/             # スキル本体
    ├── backend-patterns/
    ├── postgres-patterns/
    ├── tdd-workflow/
    └── ... (数百スキル)
```

## catalog.md

`skill-library` スキルが参照する軽量インデックス。スキル追加時は catalog.md にもエントリを追加すること。

## スキルの追加

1. `skills/<skill-name>/SKILL.md` にスキル本体を配置
2. `catalog.md` に説明とタグを追記
3. push
