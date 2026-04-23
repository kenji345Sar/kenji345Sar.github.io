# service-site

GitHub Pages で公開するためのサービス紹介サイトの最小構成。

## フォルダ構成

```
service-site/
├── index.html         # トップページ
├── assets/
│   └── style.css      # スタイル
└── README.md          # このファイル
```

## ローカル確認

Finder から `index.html` をダブルクリックすればブラウザで開けます。

もしくは、ターミナルで以下を実行してローカルサーバで確認できます(Python が入っていれば動きます)。

```bash
cd /Users/apple/Desktop/Site/service-site
python3 -m http.server 8000
```

ブラウザで http://localhost:8000 にアクセス。

## 次の段階

1. 文章を自分用に書き換える(`index.html` の各セクション)
2. GitHub にリポジトリを作り、push する
3. リポジトリ設定の Pages でデプロイする
4. 独自ドメインを設定する

段階2以降は、準備ができたタイミングで手順を追加します。
