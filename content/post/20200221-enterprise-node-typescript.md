---
title: "20200209 Node Design Pattern"
date: 2020-02-07T17:55:37+09:00
draft: true
---


# Enterprise Node Application with TypeScript

## Clean & Consistent Express.js Controllers

綺麗で一貫性があるExpress.jsのコントローラをTypeScriptで抽象化とカプセル化を駆使して作ってみる。
Express.jsは最小限のコードで動くので取っつきやすくはあるが、その勘弁さゆえ、コントローラとルーターの記述は汚くなりがちである。
バグの混入を防ぐため、コードをDRYに保ち、APIに一貫性を持たせることがとても大切。

応答の安定性と構造をよくするために良い方法が2つある。

1. Data Transfer Object (DTO)を使おう。レスポンスの構造をこれで決めてしまう。
2. BaseControllerという抽象オブジェクトを使って、具象オブジェクトを作る。

まずは2について考えてみる。

### Creating a BaseController

コントローラの役割を考えてみる。

* requestオブジェクトとresponseオブジェクトを受け取る
* レスポンスのDTOを200で返す
* レスポンスがないときは200/201を返す
* 400エラーを返す
* 500エラーを返す

これらの処理をカプセル化するため抽象クラスを作る。

```typescript
import * as express from 'express'

export abstract class BaseController {
    // コントローラの処理の実態
    // サブクラスに実装は移譲する
    protected abstract executeImpl(req: express.Request, res: express.Response): Promise<void | any>

    // ルーターからコールされるメソッド
    // 内部で実装されていないエラーを必ずキャッチする
    public async execute(req: express.Request, res: express.Response) {
        try {
            await this.executeImpl(req, res)
        } catch (err) {
            console.log(`[BaseController]: Uncaught controller error`)
            console.log(err)
            this.fail(res, 'An unexpected error occurred')
        }
    }

    public static jsonResponse(res: express.Response, code: number, message: string) {
        return res.status(code).json({message})
    }

    public ok<T>(res: express.Response, dto?: T) {
        if (!!dto) {
            res.type('application/json')
        } else {
            return res.sendStatus(200)
        }
    }

    public created(res: express.Response) {
        return res.sendStatus(201)
    }

    public clientError(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 400, message ? message : 'Unauthorized')
    }

    public unauthorized(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 401, message ? message : 'Unauthorized')
    }

    public paymentRequired(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 402, message ? message : 'Payment required')
    }

    public forbidden(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 403, message ? message : 'Forbidden')
    }

    public notFound(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 404, message ? message : 'Not found')
    }

    public conflict(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 409, message ? message : 'Conflict')
    }

    public tooMany(res: express.Response, message?: string) {
        return BaseController.jsonResponse(res, 429, message ? message : 'Too many requests')
    }

    public todo(res: express.Response) {
        return BaseController.jsonResponse(res, 400, 'TODO')
    }

    public fail(res: express.Response, error: Error | string) {
        console.log(error)
        return res.status(500).json({
            message: error.toString()
        })
    }
}
```

どうして`execute`メソッドに加えて`executeImpl`メソッドを持つのか疑問に思うかもしれない。
これは`execute`をpublicにしてExpressのルータハンドラから実行させ、内部で`executeImpl`を呼び出しこのメソッドにコントローラのロジックの責務をもたそうという考えである。これはRequestとResponseをカプセル化し、手動で渡す必要をなくす？？

例として簡単なコントローラを作ってみる。

BaseControllerは抽象メソッドを持つので、まずはサブクラスにてこれを実装する必要がある。
まずは入力値のバリデーションについてだが、これはVO生成時のResultを見て、失敗した場合はレスポンスを返す。

```typescript
import * as express from 'express'

export class CreateUserController extends BaseController {
  protected executeImpl (req: express.Request, res: express.Response): void {
    try {
      const { username, password, email } = req.body;
      const usernameOrError: Result<Username> = Username.create(username);
      const passwordOrError: Result<Password> = Password.create(password);
      const emailOrError: Result<Email> = Email.create(email);

      const result = Result.combine([ 
        usernameOrError, passwordOrError, emailOrError 
      ]);

      if (result.isFailure) {
        // Send back a 400 client error
        return this.clientError(res, result.error);
      }

      // ... continue

    } catch (err) {
      return this.fail(res, err.toString())
    }
  }
}
```

もしバリデーションがパスできたら、ユーザを作ってDoneする。

コントローラは次のようにexportする。

```typescript
import { CreateUserController } from './CreateUserController'
import { UserRepo } from '../repos/UserRepo';
import { models } from '../infra/sequelize';

const userRepo = new UserRepo(models);
const createUserController = new CreateUserController(userRepo);

// Export the feature 
export { createUserController };
```

ルータのセッテイング

```typescript
import { createUserController } from '../useCases/createUser'

import * as express from 'express'
import { Router } from 'express'
const userRouter: Router = Router();

userRouter.post('/new', 
  middleware.useCORS,
  middleware.rateLimit,
  // + any other middleware 
  ...
  (req, res) => createUserController.execute(req, res)
);

export { userRouter }
```

expressインスタンス

```typescript
import { userRouter } from '../users/http/routers'

const app = express();
app.use('/user', userRouter)
```

## Flexible Error Handling w/ the Result Class

## Knowing When CRUD & MVC Isn't Enough

## Clean Node.js Architecture

## Better Software Design with Application Layer Use Cases

ユースケースを見出すための重要なポイント

* アクターが誰なのか
* そのユースケースはコマンドなのかクエリなのか
* そのユースケースは境界づけられたコンテキスト上のサブドメインに属していないか

### Actor

アクターは大体システムのユーザと同義ではあるが、「ロール」を見極めることが重要。

### Command or Queries

オブジェクト指向の原則論にコマンドクエリ分離原則がある。これにしたがっているとアプリケーションの副作用の見極めが用意になるので可読性、バグ発生率を下げることができる。

### Use cases belong to a particular subdomain

ドメインの分割、mixpaceなら

* 認証
* フォルダ管理
* ファイル管理（アップロード含む）
* アクセスコントロール管理
* 利用量

課題となるのは

* サブドメインをどのように分割するか
* ユースケースをどのサブドメインに所属させるか
* ユースケースの変更容易性をどうやって向上させるか

### アプリケーションレイヤーの役割

ビジネスの'ルール'はドメインオブジェクトに書いていくのが原則。
インフラストラクチャは外界サービスとの接続方法を書いていくのが原則。

ではアプリケーションは？アプリケーションは’機能'を記述していく？
ドメインロジックの実行とエンティティの取得？？？

リポジトリからエンティティを取り出して、またリポジトリに保存するのが基本。

プロジェクト構造として

* Subdomain
  * UseCases
  * entities

みたいにしてみる。ドメインオブジェクトとユースケースを可視化しやすくする。

1ユースケース1クラスとして実装を考えると、こんなインタフェースが考えられる。

```typescript
export interface UseCase<IRequest, IResponse> {
  execute (request?: IRequest) : Promise<IResponse> | IResponse;
}
```

IRequest、IResponseを型パラメータとして、ユースケースの入力と出力を型定義する。

### ユースケースはインフラに関知しない


## Functional ErrorHanding with Express.js and DDD

## Why Event-Based System?