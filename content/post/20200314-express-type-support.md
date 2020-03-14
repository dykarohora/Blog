---
title: "ExpressのRequestとResponseがGenericsになった"
date: 2020-03-14T20:42:11+09:00
slug: "20200314_express-reqres-generics"
author: "d_yama"
draft: false
categories: ["typescript"]
---

# Expressをもっとタイプセーフに

v4.17.2にて`Request`型がジェネリクス型に、v4.17.3にて`Reponse`型がジェネリクス型に型定義が変更されたので、どのようなものになったのか調べてみました。結論から先に言うと、`Request`型では名前付きルートパラメータの型を、`Response`型はレスポンスボディの型を型パラメータで指定できるようになりました。

## Request

`Request`型の型定義は以下の通りです。

```typescript
interface Request<P extends core.Params = core.ParamsDictionary> extends core.Request<P> { }
interface RequestHandler<P extends core.Params = core.ParamsDictionary> extends core.RequestHandler<P> { }
interface RequestParamHandler extends core.RequestParamHandler { }
```

型定義の実態は`@types/express-serve-static-core`に書かれています。

```typescript
export interface Request<P extends Params = ParamsDictionary, ResBody = any, ReqBody = any> extends http.IncomingMessage, Express.Request {
    body: ReqBody;
    params: P;
    query: any;
    res?: Response<ResBody>;
}
```

どうやら`body`や`params`(ルートパラメータ)の型をジェネリクスで指定できるみたいです。ただ、今のところアクセスできる型パラメータは`params`の型だけのようです。その型パラメータ`P`には`Params`という型制約が設けらており、デフォルトは`ParamsDictionary`となっています。

```typescript
export interface ParamsDictionary { [key: string]: string; }
export type ParamsArray = string[];
export type Params = ParamsDictionary | ParamsArray;
```

`string`の配列、もしくは`string`型のインデックスシグネチャを型パラメータに指定できるようです。実際に使ってみると。

```typescript
type RootParams = {
    id: string,
    class: string
}

app.get('/api/students/:id/:class', (req: Express.Request<RootParams>, res:Express.Response)=> {
    // paramsの型推論が可能となる
    console.log(req.params.id)
    console.log(req.params.class)
    res.sendStatus(200)
})
```

## Response

```typescript
export interface Response<ResBody = any> extends core.Response<ResBody> { }
```

型定義の実態は`@types/express-serve-static-core`に書かれています。

```typescript
export interface Response<ResBody = any> extends http.ServerResponse, Express.Response {
    /** 中略 **/
    send: Send<ResBody, this>;
    json: Send<ResBody, this>;
    jsonp: Send<ResBody, this>;
}

export type Send<ResBody = any, T = Response<ResBody>> = (body?: ResBody) => T;
```

`Response`型のジェネリクスはシンプルです。型パラメータに指定した型が`send`メソッドや`json`メソッドの引数の型になります。すなわちレスポンスボディの型をジェネリクスで指定できるということですね。

```typescript
interface Student {
    name: string,
    age: number
}

app.get('/api/students/:id', (req: Express.Request, res:Express.Response<Student>)=> {
    const student: Student = {
        name: 'nanoha',
        age: 9
    }

    res.status(200).send(student)
    // res.status(200).send('fate') ←Student interfaceに合致していない場合はエラー
})
```

## さいごに

`Request`型は`ResBody`や`ReqBody`という型パラメータが使えるようになると一気に使いやすくなりそうです。一方`Response`型は今すぐにでも効果が得られそうなので、どんどんレスポンスオブジェクトの型を縛っていけばいいんじゃないかなと思います。
