---
title: "複数のAPIドキュメントを管理するRedocサーバの構築"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenApi, Redoc, Docker, Nginx]
published: true
---

# 記事の概要
この記事では、複数のOpenAPIドキュメントを一つのインターフェースで管理するRedocサーバの構築方法について説明します。
これにより、フロントエンドとバックエンド開発者が、複数のAPIドキュメントを効率的に参照できるようになります。

# 背景
私のチームでは、開発を始める前にAPIドキュメントを作成し、フロントエンドとバックエンドのインターフェースを定義しています。このAPIドキュメントは開発中に頻繁に参照されるのですが、これまではいちいち手元でコマンドをたたいてhtmlファイルを生成していました。これでは、コマンドを叩くたびにhtmlファイルを生成するので、非常に手間がかかるのと、無意識にgit addしてしまうことがあるので不便でした。また、私のチームでは以下のように複数のAPIドキュメントを管理しており、それぞれのhtmlファイルを開くのも面倒でした。
```
├── orders
│   ├── index.yaml
│   ├── paths.yaml
│   └── schema.yaml
├── points
│   ├── index.yaml
│   ├── paths.yaml
│   └── schema.yaml
├── products
│   ├── index.yaml
│   ├── paths.yaml
│   └── schema.yaml
```

RedocサーバーをDockerで構築して、APIドキュメントを閲覧できるようにすることは一般的です。しかし、それは1つのドキュメントを管理することが前提であり、複数のAPIドキュメントを管理することはできません。そこで、複数のAPIドキュメントを管理するRedocサーバを構築しました。

# 開発方針
開発方針は以下の通りです
- redocを埋め込んだhtmlを作成
- 保守性の観点からドキュメントのルートは別Jsonファイルで管理する
- Dockerでnginxを用いて、そこにhtmlファイルを置く
- Dockerでホスティングして手元で複数のAPIドキュメントを一つのUIインターフェースで閲覧できるようにする

開発後のファイル構成は以下の通りです。apiディレクトリ配下は既存のAPIドキュメント群です。
```
├── redoc-server
│   ├── src
│   │   ├── index.html
│   │   ├── urls.json
│   │   └── style.css
│   └── default.conf
├── docker-compose.yaml
├── api
│   ├── orders
│   │   ├── index.yaml
│   │   ├── schema.yaml
│   │   └── paths.yaml
│   ├── customers
│   │   ├── index.yaml
│   │   ├── schema.yaml
│   │   └── paths.yaml
```

# nginxの設定
まずnginxの設定をします。一般的な設定です。
後で作成するhtmlファイル、apiドキュメントのyamlファイル、nginxの設定ファイルをマウントしています。
``` yaml:docker-compose.yaml
version: '3'
services:
  redoc:
    image: nginx:latest
    volumes:
      - ./redoc-server/default.conf:/etc/nginx/conf.d/default.conf
      - ./redoc-server/src:/usr/share/nginx/html
      - ./api:/usr/share/nginx/html/api
    ports:
      - "8080:80"
```

```conf:default.conf
server {
        listen 80;
        listen 443;
        server_name localhost;

        root /usr/share/nginx/html;
        index index.html;

        location / {
                try_files $uri $uri/ /index.html$query_string;
        }

        error_page   500 502 503 504  /50x.html;
                 location = /50x.html {
                   root   /usr/share/nginx/html;
        }
}
```

# htmlファイルの作成
次にhtmlファイルを作成します。以下のように、urls.jsonにAPIドキュメントのパスを記述し、index.htmlでそれを読み込むようにします。
初期表示時にセレクトボックスにAPIドキュメントのリストを格納し、選択したAPIドキュメントを表示するようにしています。
`Redoc.init`の引数にAPIドキュメントの相対パスを指定することでAPIドキュメントが表示できます。
(jsのところはあんまりきれいなコードじゃなくて読みにくいかもです...)

参考: https://redocly.com/docs/redoc/deployment/html/

```json:urls.json
[
    {
        "name": "orders",
        "url": "/api/orders/index.yaml"
    },
    {
        "name": "customers",
        "url": "/api/customers/index.yaml"
    }
]
```

```html:index.html
<!DOCTYPE html>
<html>

<head>
    <title>Redoc</title>
    <!-- needed for adaptive design -->
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700|Roboto:300,400,700" rel="stylesheet">
    <link rel="stylesheet" href="style.css" type="text/css">

    <!--
    Redoc doesn't change outer page styles
    -->
    <style>
        body {
            margin: 0;
            padding: 0;
        }
    </style>
</head>

<body>
    <select class="redoc-select" name="doc">
        <option value="">Select a document</option>
    </select>
    <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"> </script>
    <!-- <redoc spec-url=./yamls/order.yaml></redoc> -->
    <div id="redoc-container"></div>
</body>
<script>
    fetch('./urls.json')
        .then(response => response.json())
        .then(data => {
            let select = document.querySelector('select[name="doc"]');
            data.forEach((item, index) => {
                let option = document.createElement('option');
                option.value = item.url;
                option.text = item.name;
                select.appendChild(option);
            });
        })
        .catch(error => console.error(error));

    document.querySelector('select[name="doc"]').addEventListener('change', function (e) {
        const url = e.target.value;
        if (url === "") {
            return;
        }
        Redoc.init(url, {}, document.getElementById('redoc-container'))
    })
</script>

</html>
```

# 運用
以上で構築は終了です。あとは手元で
```bash
docker compose up redoc -d
```
をたたいて、ブラウザで`http://localhost:8080`にアクセスすれば、複数のAPIドキュメントをセレクトボックスで切り替えて閲覧できます。

