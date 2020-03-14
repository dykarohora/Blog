---
title: "20200208 Node Design Pattern"
date: 2020-02-07T17:55:37+09:00
draft: true
---

# Resultでエラーをハンドリングしよう！

エラーのthrowを使うとコードの可読性とトレーサビリティに悪影響を及ぼす可能性がある。他の方法を考えてみよう！

try-catchで処理するような例外はどこで使うべきなんだろうか。

ファクトリメソッドに引数をわたしてUserオブジェクトを作る例で考えよう。

```typescript
class User {
  public email: string;
  public firstName: string;
  public lastName: string;

  private constructor (email: string, firstName: string, lastName: string): User {
    this.email = email;
    this.firstName = firstName;
    this.lastName = lastName;
  }

  public static createUser (email: string, firstName: string, lastName: string): User {
    if (!isValidEmail(email)) {
      throw new Error("Email is invalid")
    }

    // .. validate firstName

    // .. validate lastName

    // return new user
  }
}
```

ここで、クラス内からそのエラーをキャッチするのは理にかなっているかを考える。そんなことはぶっちゃけありえない。nullを返すのは？あかんでしょ。メソッドの呼び出し側はUserオブジェクトが返ってくることを期待しているもの。ということは呼び出し側をtry-catchで包んでやるしかないんだけど、それっていけてなくない？

## なんでエラーをthrowするのはよくないのだろう？

ケースバイケース、すべてにおいて最良の選択肢など存在しないのだけど、今回のようにオブジェクトの作成時にエラーをthrowするのは危険で、呼び出し側に制約を課すことになってしまう。

例外スローはtype safeではない。

また、大域脱出でコードがジャンプしてしまうので追跡性が落ちる、GOTOのようだ。

## Resultクラス

```typescript
export class Result<T> {
    public isSuccess: boolean
    public isFailure: boolean
    public error?: string | null
    private _value?: T

    private constructor(isSuccess: boolean, error?: string | null, value?: T) {
        // 成功なのにエラーがある
        if (isSuccess && error) {
            throw new Error(`InvalidOperation: A result cannot be 
        successful and contain an error`)
        }
        // 失敗なのにエラーがない
        if (!isSuccess && !error) {
            throw new Error(`InvalidOperation: A failing result 
        needs to contain an error message`)
        }

        this.isSuccess = isSuccess
        this.isFailure = !isSuccess
        this.error = error
        this._value = value

        Object.freeze(this)
    }

    public getValue(): T {
        if (!this.isSuccess) {
            throw new Error(`Cant retrieve the value from a failed result.`)
        }

        return this._value!
    }

    public static ok<U>(value?: U): Result<U> {
        return new Result<U>(true, null, value)
    }

    public static fail<U>(error: string): Result<U> {
        return new Result<U>(false, error)
    }

    public static combine(results: Result<any>[]): Result<any> {
        for (let result of results) {
            if (result.isFailure) return result
        }
        return Result.ok<any>()
    }
}
```

* エラーを安全に返す
* 結果を取得できる
* Resultを組み合わせて、全体で成功したか、一部が失敗したかを判定できる

### Resultの使い方

```typescript
class User {
  public email: string;
  public firstName: string;
  public lastName: string;

  private constructor (email: string, firstName: string, lastName: string): User {
    this.email = email;
    this.firstName = firstName;
    this.lastName = lastName;
  }

  public static createUser (email: string, firstName: string, lastName: string): Result<User> {
    if (!isValidEmail(email)) {
      return Result.fail<User>('Email is invalid')
    }

    if (!!firstName === false && firstName.length > 1 && firstName.length < 50) {
      return Result.fail<User>('First name is invalid')
    }

    if (!!lastName === false && lastName.length > 1 && lastName.length < 50) {
      return Result.fail<User>('Last name is invalid')
    }

    return Result.ok<User>(new User(email, firstName, lastName));
  }
}
```

```typescript
class CreateUserController {
  public executeImpl (): void {
    const { req } = this;
    const { email, firstName, lastName } = req.body;
    const userOrError: Result<User> = User.create(email, firstName, lastName);

    if (userOrError.isFailure) {
      return this.fail(userOrError.error)
    }

    const user: User = userOrError.getValue();

    // persist to database ...
    
  }
}
```

値オブジェクトも同じ思想で作ったとすると、`combine`を活用してこんなこともできる。

```typescript
class CreateUserController {
  public executeImpl (): void {
    const { req } = this;
    const { email, firstName, lastName } = req.body;
    const emailOrError: Result<Email> = Email.create(email);
    const firstNameOrError: Result<FirstName> = FirstName.create(firstName);
    const lastNameOrError: Result<LastName> = LastName.create(lastName);

    const userPropsResult: Result<any> = Result.combine([ 
      emailOrError, firstNameOrError, lastNameOrError
    ])

    // If this failed, it will return the first error that occurred.
    if (userPropsResult.isFailure) {
      return this.fail(userPropsResult.error)
    }

    const userOrError: Result<User> = User.create(
      emailOrError.getValue(), 
      firstNameOrError.getValue(), 
      lastNameOrError.getValue()
    );

    if (userOrError.isFailure) {
      return this.fail(userOrError.error)
    }

    const user: User = userOrError.getValue();

    // persist to database ...
  }
}
```

## throwはいつ使えばいいのか

ライブラリを使う時、または外界との接続が必要な処理を書くとき


# Expressでのエラーハンドリング

エラーをどうやって、どこで処理をするのかというのは難しい問題である。可用性を担保するのにとても大切であるのに対し、後回しにされがちな事柄である。

TypeScript + Node.jsだと

* エラーをthrowする
* nullを返す

が考えられる。

エラーをthrowするのはプログラムの流れをわかりにくくするという、GOTOと同じような問題がある。

一方nullを返すのはどうかというと、メソッドは単一の型を返すべしという原則が守られない。これはクライアントがメソッドを誤用してしまう原因となる。

一つの解として、Resultクラスを使うことにより、エラーをthrowしてフローをわかりにくくしてしまうという問題を解決した。また、Result.ok、Result.failで成否を区別できるようにしたが、まだ欠けていることがある。

エラーもドメインの要素ではなかろうか。

エラーはプログラムの動作の多くを占めるし無視することができない。また、モデリングにおいてエラーをドメインの概念として表現すると、ドメインモデルがより豊かになる。

これは可読性の向上により、バグの作り込みを減らすことに繋がる。

理解すべきこと

* ドメインモデリングにとってエラーを明示的に表現することが重要な理由  
* 型を使ってエラーの表現力を高める方法
* ユースケースごとにすべてのエラーを整理する方法と理由
* エレガントにExpressと統合する方法

## 一般的なエラーハンドリングの方法は、ドメインの表現力を著しく損なう

### nullを返す

```typescript
function CreateUser (email, password) {
  const isEmailValid = validateEmail(email);
  const isPasswordValid = validatePassword(password);

  if (!isEmailValid) {
    console.log('Email invalid');
    return null;
  }

  if (!isPasswordValid) {
    console.log('Password invalid');
    return null;
  }

  return new User(email, password);
}
```

これは明らかにダメ！圧倒的にダメ！！
これは呼び出しが側が「Userの生成に失敗すると、nullが返ってくる」ということを把握しなければいけない、要はnullチェックが必要になる。いかなるところでもnullチェックをしないといけない？コードは汚くなるし、ミスも誘発する、あかんでしょ！

しかも、nullだと失敗した理由がわからない。例えば上のケースだとメールアドレスに不備があるのか、パスワードに不備があるのか、全くもってわからんのだよ！

これを解決するために出てくるであろうものが、ロギングとエラーthrow

```typescript
function CreateUser (email, password) {
  const isEmailValid = validateEmail(email);
  const isPasswordValid = validatePassword(password);

  if (!isEmailValid) {
    console.log('Email invalid');
    return new Error('The email was invalid');
  }

  if (!isPasswordValid) {
    console.log('Password invalid');
    return new Error('The password was invalid');
  }

  return new User(email, password);
}
```

エラークラスのメッセージに失敗理由を書くようにした。これを呼び出し側でtry-catchで囲えばエラーの理由を知ることができる。

```typescript
try {
  const user = CreateUser(email, password);
} catch (err) {
  switch (err.message) {
    // Fragile code
    case 'The email was invalid':
      // handle
    case 'The password was invalid':
      // handle
  }
}
```

通常、エラーをthrowするのはDBやAPIへのアクセスなど、外界と通信するときに限定すべきである。

## よりよいエラーハンドリング

Result<T>を使うことによって、例外スローを使わずに表現できるようにはなったが、最大の問題はエラーの理由が複数考えられる場合である。

ユーザ名とメールアドレス、パスワードを受け取ってユーザを作ることを考える。成功を表すのは「ユーザの作成が完了した」というただ一つに限定される。一方、エラーの場合は

* メールアドレスの形式に不備がある
* ユーザ名がすでに使用されている
* 登録済みのメールアドレスが入力された
* パスワードのセキュリティ強度が弱い

ExpressのAPIだとすると、成功は201、エラーは40x、内部エラーは50xで返すよね。

ユースケースで考える。

```typescript
interface User {}

// UserRepository contract
interface IUserRepo {
  getUserByUsername(username: string): Promise<User>;
  getUserByEmail(email: string): Promise<User>;
}

// Input DTO
interface Request {
  username: string;
  email: string;
  password: string;
}

class CreateUserUseCase implements UseCase<Request, Promise<Result<any>>> {
  private userRepo: IUserRepo;

  constructor (userRepo: IUserRepo) {
    this.userRepo = userRepo;
  }

  public async execute (request: Request): Promise<Result<any>> {
    ... 
  }
}
```

ユースケースでは考慮すべきエラー状態があることがわかる

```typescript
class CreateUserUseCase implements UseCase<Request, Promise<Result<any>>> {
  private userRepo: IUserRepo;

  constructor (userRepo: IUserRepo) {
    this.userRepo = userRepo;
  }

  public async execute (request: Request): Promise<Result<any>> {

    // Determine if the email is valid
      // If not, return invalid email error

    // Determine if the password is valid
      // if not, return invalid password error

    // Determine if the username is taken
      // If it is, return an error expressing that the username was taken

    // Determine if the user already registered with this email
      // If they did, return an account created error

    // Otherwise, return success

    return Result.ok<any>()
  }
}
```

値オブジェクトの中にバリデーションロジックがカプセルかしているならば、Resultを使って次のようにできる

```typescript
class CreateUserUseCase implements UseCase<Request, Promise<Result<any>>> {
  private userRepo: IUserRepo;

  constructor (userRepo: IUserRepo) {
    this.userRepo = userRepo;
  }

  public async execute (request: Request): Promise<Result<any>> {

    const { username, email, password } = request;

    const emailOrError: Result<Email> = Email.create(email);

    if (emailOrError.isFailure) {
      return Result.fail<any>(emailOrError.message)
    }

    const passwordOrError: Result<Password> = Password.create(password);

    if (passwordOrError.isFailure) {
      return Result.fail<void>(passwordOrError.message)
    }

    ...
  }
}
```

ユースケースでは、エラーが発生した時にResultから失敗を検知できるが、失敗の理由はメッセージの中身をみないと判定することができない。

## ユースケースでエラーの種別まで表現するには

