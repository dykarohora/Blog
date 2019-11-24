---
title: "20191023 Azure Functions Dev Environment"
date: 2019-10-23T17:38:12+09:00
draft: true
---

# List\<T>を実装してみよう！

コレクションとLINQとRxへの理解を深めたいと思い、何となくまずは一番シンプル（な気がする）`List<T>`クラスを実装してみるとよいのではないのだろうかと考えてやってみました。
参考とするのは.NET Core 3.0の実装です。

## 解

[公式のListの実装](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/Collections/Generic/List.cs)

# 写経してみた

細部は違うんだけど

リスト＝そとからみた
配列＝うちからみた

## えるしっているか、Listは配列

何はともあれ要素を詰め込んでおく入れ物が必要です。

```csharp
public class List<T>
{
    // Listの正体は配列だった！！！
    internal T[] _items;
    // たぶん配列に格納されている要素の数（＝Listの要素数）
    internal int _size;

    // T型の空配列
    private static readonly T[] s_emptyArray = new T[0];

    public List()
    {
        // デフォルトコンストラクタはstaticな空配列への参照をコピー
        _items = s_emptyArray;
    }
}
```

## とりあえず要素を追加できるようにしてみよう

`List<T>`で要素の追加といえば`Add`メソッド、実装してみよう。

```csharp
    public void Add(T item)
    {
        // メソッド呼び出し時点の配列への参照とListの要素数をコピー
        T[] array = _items;
        int size = _size;

        // 要素数が配列の長さより小さい、すなわち配列にまだ要素を入れる余裕があるなら配列に引数として渡された要素を入れる
        if ((uint)size < (uint) array.Length)
        {
            _size = size + 1;
            array[size] = item;
        }
        // 配列に要素を入れる余裕がないとき
        else
        {
            AddWithResize(item);
        }
    }
```

配列に要素を追加しているだけ、計算量も明らかにO(1)。なんだけど、配列の長さは固定長なのでどんどん詰め込むと溢れてしまう。例外がスローされてあぼんです。
というかデフォルトコンストラクタで空配列を使っているので最初から要素入れられない…
ということでif-else文にある`AddWithResize`メソッドがなんか大事っぽいです。

```csharp
    private void AddWithResize(T item)
    {
        int size = _size;
        // メソッド名から推測するに、ここで配列のキャパシティを増やしている…？
        EnsureCapacity(size + 1);
        // キャパシティを増やしたあとに要素を格納
        _size = size + 1;
        _items[size] = item;
    }

    private void EnsureCapacity(int min)
    {
        // 現在の配列の長さが、要求されている長さより小さいことを確認
        if (_items.Length < min)
        {
            // 更新後のキャパシティを計算しているけど、配列そのものはまだ操作してない
            int newCapacity = _items.Length == 0 ? DefaultCapacity : _items.Length * 2;
            if (newCapacity < min) newCapacity = min;
            Capacity = newCapacity;
        }
    }

    private const int DefaultCapacity = 4;
```

どうやら現在の配列の長さの2倍の値を更新後のサイズとしているようです。ただ、`AddWithResize`メソッドからの`EnsureCapacity`メソッドでは配列そのものは弄っておらず、`Capacity`というプロパティに計算結果をセットしているだけです。
この`Capacity`プロパティですが、[公式ドキュメント](https://docs.microsoft.com/ja-jp/dotnet/api/system.collections.generic.list-1.capacity?view=netframework-4.8#System_Collections_Generic_List_1_Capacity)では以下のように書かれています。

> 内部データ構造体がサイズ変更せずに格納できる要素の合計数を取得または設定します。

なるほど、`List`のキャパシティはある程度操作できるみたいです。

```csharp
    public int Capacity
    {
        get => _items.Length;
        set
        {
            // 現在のリストの要素数よりキャパシティを小さくすることは許されない
            if (value < _size)
            {
                throw new ArgumentOutOfRangeException();
            }

            if (value != _items.Length)
            {
                if (value > 0)
                {
                    // 指定された長さの配列を新しく確保し、古い配列から中身をまるっとコピーする
                    T[] newItems = new T[value];
                    if (_size > 0)
                    {
                        Array.Copy(_items, 0, newItems, 0, _size);
                    }
                    // 参照先の変更
                    _items = newItems;
                }
                else
                {
                    _items = s_emptyArray;
                }
            }
        }
    }
```

配列の長さの変更は`Capacity`プロパティのセッターを使ったときに行われます。しかし`new`とコピーを駆使しているのでメモリと時間のリソースをじゃぶじゃぶ使ってる感がありますね…。しかもデフォルトコンストラクタを使っている場合だと、`List`の要素数が0 → 4 → 8 → 16 → 32 →...を超えるタイミングで上の操作が発生することとなりそうです。

微々たるものだと言えばそうなんですが、`List<T>`のコンストラクタにはインスタンス生成時のキャパシティを指定できるものがあるので、要素数がある程度見積もれる最初からキャパシティを指定しておくのもいいかもしれません。

```csharp
    public List(int capacity)
    {
        if (capacity < 0)
        {
            throw new ArgumentOutOfRangeException();
        }

        if (capacity == 0)
        {
            _items = s_emptyArray;
        }
        else
        {
            _items = new T[capacity];
        }
    }
```

## 要素を取得したい

要素を追加したあとはリストから要素を取得するだろ？誰だってそーする、俺もそーする。

```csharp
var magicalGirls = new List<string>() {"nanoha","fate","hayate"};
Console.WriteLine(magicalGirls[0]);     // nanoha
Console.WriteLine(magicalGirls[1]);     // fate
Console.WriteLine(magicalGirls[2]);     // hayate
```

C#のList\<T>ではこんな感じで配列アクセスのようにインデックスを指定して要素にアクセスすることができます。
しかしこのような配列アクセスっぽい振る舞いは自分で実装できるものなんでしょうか？なんとC#ではできます。

```csharp
    public T this[int index]
    {
        get
        {
            // リストの要素数を超えたインデックスアクセスはもちろん許されない
            if ((uint) index >= (uint) _size)
            {
                throw new ArgumentOutOfRangeException();
            }
            return _items[index];
        }
    }
```

なんだかプロパティ定義に似ていますが、これは`インデクサー`と呼ばれる仕組みです。この仕組みを使うことによって、自前のクラスでも配列アクセスのような機能を提供することができます。

`List<T>`での実装では配列からインデックスを指定して要素を取得しているので計算量はO(1)ですね。追加もO(1)だし要素がどんどん増えていくようなコレクションに向いてそう。

なお、上ではgetだけの定義ですが、インデクサーもプロパティと同様、`set`を定義することができますし、実際`List<T>`にも定義されています。

```csharp
    public T this[int index]
    {
        get { ... }
        set
        {
            if ((uint) index >= (uint) _size)
            {
                throw new ArgumentOutOfRangeException();
            }
            _items[index] = value;
        }
    }
```

こんな感じで`List<T>`では配列のようにインデックスを指定して要素の読み書きできるようになっています。

## 要素の削除だってしたい

`List<T>`には要素を削除するメソッドとして`Remove(T)`、`RemoveAt(int)`、`RemoveAll(Predicate<T>)`、`RemoveRange(int,int)`があります。なんとなく字面から`Remove`が一番正道に見えますね。

```csharp
    public bool Remove(T item)
    {
        // 引数に指定された値がリスト内に存在する場合、その値が最初に見つかった位置のインデックスを返す
        int index = IndexOf(item);
        if (index >= 0)
        {
            // インデックスを指定してリストから要素を削除する
            RemoveAt(index);
            return true;
        }

        return false;
    }

    // 指定した要素のインデックスを見つけるメソッドはArray(配列型)のstaticメソッドを使っているようだ
    public int IndexOf(T item) => Array.IndexOf(_items, item, 0, _size);
```

どうやら削除する前にまずはリスト内に削除したい要素が本当に存在しているかどうか確認しているみたい。それが`IndexOf`メソッドなんですが、この配列型のメソッドはいったいなんなんだ…。

### IndexOf

```csharp
    // List<T>.Remove(T)からのコールの場合だと次の通り引数が渡されてくる
    // array - リストの実態である配列(_items)
    // value - 削除したい要素
    // startIndex - 0、要はリストの先頭から検索する
    // count - リスト内の要素の数
    public static int IndexOf<T>(T[] array, T value, int startIndex, int count)
    {
      if (array == null)
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.array);
      if (startIndex < 0 || startIndex > array.Length)
        ThrowHelper.ThrowStartIndexArgumentOutOfRange_ArgumentOutOfRange_Index();
      if (count < 0 || count > array.Length - startIndex)
        ThrowHelper.ThrowCountArgumentOutOfRange_ArgumentOutOfRange_Count();
      return EqualityComparer<T>.Default.IndexOf(array, value, startIndex, count);
    }
```

いろいろ引数の事前条件チェックを行って、それがパスしたら`EqualityComparer`クラスのstaticメソッドを呼んでいる、いつになったらサーチが始まるんだ…。`EqualityComparer`はオブジェクトの等価判定に関わる抽象クラスっぽいんですが、こちらの詳細は追々…。

```csharp
public abstract class EqualityComparer<T> : IEqualityComparer, IEqualityComparer<T>
{
    internal virtual int IndexOf(T[] array, T value, int startIndex, int count)
    {
        int num = startIndex + count;
        // ここでサーチ！！！
        for (int index = startIndex; index < num; ++index)
        {
            if (this.Equals(array[index], value))
                return index;
        }

        return -1;
    }
}
```

等価性のもろもろはすっ飛ばして！とりあえず気にしておきたいのは、要素をサーチするために配列の先頭からループまわして等価性チェックしてるってところですね。なので(最悪の)計算量はO(n)になるってところですかね。

### RemoveAt

削除対象が見つかった、あとは削除するだけ。メソッドシグネチャ見ればなんとなく分かる気がしますが、`RemoveAt`は対象の要素のインデックスを指定して削除するかんじですね。

```csharp
    public void RemoveAt(int index)
    {
        if ((uint) index >= (uint) _size)
        {
            throw new ArgumentOutOfRangeException();
        }

        _size--;
        if (index < _size)
        {
            Array.Copy(_items, index + 1, _items, index, _size - index);
        }
    }
```

Listにおける削除の実体はなんと配列のコピーであった！
このコピー操作のイメージは以下みたいなかんじです。


## 検索だってしたい！

## ソートもしたい！

## foreachで列挙もしたい！

## LINQだって使いたい！