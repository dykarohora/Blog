---
title: "20190823 Nuxt Development"
date: 2019-08-15T07:05:10+09:00
draft: true
---
# Docker
## Dockerfileをつくる
FROM node:12.8.0-alpine
ENV APP_ROOT /app/

WORKDIR $APP_ROOT
RUN apk update

COPY . $APP_ROOT

RUN yarn

ENV HOST 0.0.0.0
EXPOSE 3000

CMD ["yarn", "start"]

## docker-compose
version: '3'
services:
  web:
    build:
      context: .
      dockerfile: ./Dockerfile
    environment:
      NODE_ENV: development
      PORT: 3000
      HOST: 0.0.0.0
    ports:
      - "3000:3000"
    volumes:
      - .:/app

イメージをビルドしてコンテナにログイン
nuxt-app d_yama$ docker-compose run --rm web ash

# Nuxtつくる
yarn create nuxt-app front

✨ Generating Nuxt.js project in front
? Project name front
? Project description Nuxt.js application
? Author name d_yama
? Choose the package manager Yarn
? Choose UI framework None
? Choose custom server framework Express
? Choose Nuxt.js modules Axios
? Choose linting tools (Press <space> to select, <a> to toggle all, <i> to invert selection)

? Choose test framework None
? Choose rendering mode Single Page App

## ディレクトリ構成を変える
変更前
.
├── Dockerfile
├── docker-compose.yml
└── front
    ├── README.md
    ├── assets
    ├── components
    ├── layouts
    ├── middleware
    ├── node_modules
    ├── nuxt.config.js
    ├── package.json
    ├── pages
    ├── plugins
    ├── server
    ├── static
    ├── store
    └── yarn.lock

変更後
.
├── Dockerfile
├── docker-compose.yml
├── front
│   ├── README.md
│   ├── assets
│   ├── components
│   ├── layouts
│   ├── middleware
│   ├── nuxt.config.js
│   ├── pages
│   ├── plugins
│   ├── static
│   └── store
├── node_modules
├── nuxt.config.js
├── package.json
├── server
│   └── index.js
└── yarn.lock


## ファイルを修正する
nuxt.config.js
以下を追加
srcDir: './front',

## script設定
    "dev": "cross-env NODE_ENV=development nodemon server/index.js --watch server",
    "build": "nuxt build",
    "start": "nuxt build && cross-env NODE_ENV=production node server/index.js",

## 動作確認
yarn dev
yarn start

イメージをビルド
docker-compose build

イメージの動作確認
docker run -p 3000:3000 nuxt-app_web

# Azureへのデプロイを自動化する
ステージング
## ACRをつくる
PowerShellでいくか
New-AzContainerRegistry -ResourceGroupName mxpFunctions -Location 'japan east' -Name 'mxpContainerRegistry' -Sku 'Basic'

## AppServicePlanをつくる

## WebApp for Containersをつくる

## パイプラインをつくる

### Buildパイプライン

### Relaseパイプライン 

# TypeScript対応
Nuxt 2.8.1→2.9.1にアップデートする（僕はよくncu使う）
## Nuxt
yarn add --dev @nuxt/typescript-build
yarn add vue-property-decorator

nuxt.config.jsに追加
buildModules: ['@nuxt/typescript-build']

./front/tsconfig.jsonに追加
昔、nuxtコマンド実行時に自動生成されたが、されなくなった。

ts-loaderはpagesからtsconfig.jsonの存在を探していく
上にのぼっていく

// tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "node",
    "lib": [
      "esnext",
      "esnext.asynciterable",
      "dom"
    ],
    "esModuleInterop": true,
    "experimentalDecorators": true,   // クラスタイプ使いたいとき
    "allowJs": true,
    "sourceMap": true,
    "strict": true,
    "noEmit": true,
    "baseUrl": "..",
    "paths": {
      "~/*": [
        "./*"
      ],
      "@/*": [
        "./*"
      ]
    },
    "types": [
      "@types/node",
      "@nuxt/types"
    ]
  },
  "files": [
    "shims-vue.d.ts"
  ],
  "exclude": [
    "node_modules"
  ]
}

shim-vue.d.tsを追加する

TypeScripでvueを編集してみる
<script lang="ts">
import { Vue, Component, Prop } from "vue-property-decorator";
import Logo from "~/components/Logo"

@Component({
  components: {
    Logo
  }
})
export default class Index extends Vue {
  @Prop({
    required: true
  })
  private readonly name: string = "mxp";
}
</script>

## express
index.js → index.ts
import express from "express";
import consola from "consola"
// @ts-ignore
import { Nuxt, Builder } from "nuxt";
import config from "../nuxt.config";

config.dev = process.env.NODE_ENV !== "production";

const app = express();

async function start() {
  // Init Nuxt.js
  const nuxt = new Nuxt(config);
  const { host, port } = nuxt.options.server;

  // Build only in dev mode
  if (config.dev) {
    const builder = new Builder(nuxt);
    await builder.build();
  } else {
    await nuxt.ready();
  }

  // Give nuxt middleware to express
  app.use(nuxt.render);

  // Listen the server
  app.listen(port, host);
  consola.ready({
    message: `Server listening on http://${host}:${port}`,
    badge: true
  });
}

start();

プロジェクトルートにもtsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "commonjs",
    "moduleResolution": "node",
    "lib": ["esnext", "esnext.asynciterable", "dom"],
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "allowJs": false,
    "sourceMap": true,
    "strict": true,
    "noEmit": false,
    "baseUrl": "./",
    "paths": {
      "~/*": ["./*"],
      "@/*": ["./*"]
    },
    "types": ["@types/node", "@nuxt/types"]
  },
  "exclude": ["./node_modules", "./front/**/*", "./nuxt.config.js"]
}

## オートリロード
フロントはyarn devで起動すればオートリロードされる
### ts-nodeを入れる
typescriptで動かしたい！
yarn add ts-node

スクリプト変更
"dev": "cross-env NODE_ENV=development ts-node server/index.ts"

### nodemon
yarn add -D nodemon
{
  "watch": ["./server"],
  "ext": "ts",
  "exec": "ts-node ./server/index.ts"
}
"dev": "cross-env NODE_ENV=development nodemon",

# テスト書く
jest入れる
yarn add -D jest ts-jest @types/jest

設定
module.exports = {
  verbose: true,
  preset: 'ts-jest'
}

テスト通す
"test":"jest"

## Azure DevOpsでテスト回す
http://takasdev.hatenablog.com/entry/2018/11/11/212534
yarn add -D jest-junit

module.exports = {
  verbose: true,
  preset: 'ts-jest',
  reporters: [
    "default",
    "jest-junit"
  ]
}

Azure DevOpsでのテスト実行時には、JUnit形式のレポートを出力するようにする
"test:ado": "jest --reporters=jest-junit"

テストタスクとレポート出力のタスクを追加する
          - task: Npm@1
            displayName: 'npm run test'
            inputs:
              command: 'custom'
              workingDir: './'
              customCommand: 'run test:ado'
          - task: PublishTestResults@2
            displayName: 'Publish test result'
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/junit.xml'
            condition: always()

## カバレッジレポートを出力する
module.exports = {
  verbose: true,
  preset: 'ts-jest',
  reporters: [
    "default",
    "jest-junit"
  ],
  coverageReporters: [
    "text",
    "html",
    "cobertura"
  ]
}

## Vueコンポーネントのテスト
yarn add -D jest ts-jest vue-jest babel-jest @vue/test-utils @types/jest babel-core
module.exports = {
  verbose: true,
  moduleFileExtensions: [
    "js",
    "ts",
    "vue"
  ],
  transform: {
    "^.+\\.tsx?$": "ts-jest",
    ".*\\.(vue)$": "vue-jest"
  },
  reporters: [
    "default",
    "jest-junit"
  ],
  coverageReporters: [
    "text",
    "html",
    "cobertura"
  ]
}

# Vuetifyを使う
yarn add @nuxtjs/vuetify
  modules: [
    // Doc: https://axios.nuxtjs.org/usage
    "@nuxtjs/axios",
    "@nuxtjs/vuetify"
  ],

  

アップしてAppService作ってデプロイする








yarn add -D typescript
yarn add -D @nuxt/typescript
yarn add ts-node

tsconfig.jsonの生成
touch tsconfig.json

nuxt.config.js→nuxt.config.tsへリネーム

index.js
let config = require('../nuxt.config.js')
let config = require('../nuxt.config.ts')

nuxt buildでtsconfig.jsonを生成
yarn devで動作確認

## tsconfig
{
  "compilerOptions": {
    "target": "esnext",
    "moduleResolution": "node",
    "lib": [
      "esnext",
      "esnext.asynciterable",
      "dom"
    ],
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "allowJs": true,
    "sourceMap": true,
    "strict": true,
    "noImplicitAny": false,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "~/*": [
        "./*"
      ],
      "@/*": [
        "./*"
      ]
    },
    "types": [
      "@types/node",
      "@nuxt/vue-app"
    ]
  }
}

modulesをけす

### 設定ファイル
import NuxtConfiguration from '@nuxt/config'

### expressのTS化
import express from "express";
import consola from "consola";
import { Nuxt, Builder } from "nuxt";

import config from "../nuxt.config";

const app: express.Application = express();

config.dev = !(process.env.NODE_ENV === "production");

async function start() {
  const nuxt = new Nuxt(config);

  console.log("hogehoge");
  const {
    host = process.env.HOST || "127.0.0.1",
    port = process.env.PORT || 3000
  } = nuxt.options.server;

  if (config.dev) {
    const builder = new Builder(nuxt);
    await builder.build();
  } else {
    await nuxt.ready();
  }

  // Give nuxt middleware to express
  app.use(nuxt.render);

  // Listen the server
  app.listen(port, host);
  consola.ready({
    message: `Server listening on http://${host}:${port}`,
    badge: true
  });
}
start();


### コンポーネントをTSで
vue-property-decorator
vue-property-decoratorのインストール

<script lang="ts">
  import {Component, Vue} from 'vue-property-decorator'

  @Component({
    components: {
      Logo: () => import('~/components/Logo.vue')
    }
  })
  export default class Index extends Vue {}
</script>

## expressのTS化

import express from "express";
import consola from "consola";
import { Nuxt, Builder } from "nuxt";

const app: express.Application = express();

// let config = require('../nuxt.config.ts')
import config from "../nuxt.config";

config.dev = !(process.env.NODE_ENV === "production");

async function start() {
  // Init Nuxt.js
  const nuxt = new Nuxt(config);

  const {
    host = process.env.HOST || "127.0.0.1",
    port = process.env.PORT || 3000
  } = nuxt.options.server;

  // Build only in dev mode
  if (config.dev) {
    const builder = new Builder(nuxt);
    await builder.build();
  }

  // Give nuxt middleware to express
  app.use(nuxt.render);

  // Listen the server
  app.listen(port, host);
  consola.ready({
    message: `Server listening on http://${host}:${port}`,
    badge: true
  });
}

start();

# tsc-watchを入れる

# server tsconfig 
{
  "compilerOptions": {
    "target": "esnext",
    "module": "commonjs",
    "moduleResolution": "node",
    "lib": [
      "es6",
      "dom"
    ],
    "esModuleInterop": true,
    "sourceMap": true,
    "experimentalDecorators": true,
    "noImplicitAny": false,
    "noImplicitThis": false,
    "strictNullChecks": true,
    "removeComments": true,
    "suppressImplicitAnyIndexErrors": true,
    "strict": true,
    "skipLibCheck": true,
    "allowJs": true,
    "noEmit": false,
    "baseUrl": ".",
    "paths": {
      "~/*": [
        "./*"
      ],
      "@/*": [
        "./*"
      ]
    },
    "types": [
      "@types/node"
    ]
  }
}
