# playground-gemini

## Setup

前提: Dockerインストール済み、 VSCodeに拡張機能としてDevcontainer, Remote Developmentがインストール済み

postCreateCommandではインストールエラーとなったので以下を手動実行:

```bash
npm install -g @google/gemini-cli
```

インストール後のセットアップ: [参考記事](https://blog.g-gen.co.jp/entry/gemini-cli-explained)

## Action Item

- ソースコード解析のプロンプト作成
  - templates/*.md に解析観点を追加
