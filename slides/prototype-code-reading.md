# Prototype.js をコードリーディングして知る JavaScript の黒魔術

---

## 自己紹介

よんじゅ (Twitter: [@sei40kr_sub](https://twitter.com/sei40kr_sub))

- Arch Linux と Emacs と Z-shell が好き
- 最近 Spacemacs から Doom Emacs に乗り換えた
- 9 歳のとき (2002年) に JavaScript を初めて触る

---

## Prototype.js

- **Prototype JavaScript framework: a foundation for ambitious web user interfaces**
  http://prototypejs.org
- JavaScript のフルスタックライブラリ (Yahoo UI や Closure Library に網羅範囲は劣るが...)
- 最終更新日は 2015/09/22

---

## jQuery との違い

|                          | Prototype.js                           | jQuery                                                |
|:-------------------------|:---------------------------------------|:------------------------------------------------------|
| 実装スタイル             | OOP & Prototype 拡張                   | `jQuery` 以下に Function を生やす                     |
| class 実装               | Alex Arnell's Class Inheritance ベース | なし                                                  |
| Element 拡張             | Prototype 拡張                         | 要素をもつ配列をラップしたものに `jQuery.fn` をマージ |
| ID から要素を取得        | `$`                                    | なし                                                  |
| セレクターから要素を取得 | `$$`                                   | `$`                                                   |
| アニメーション           | なし                                   | 内蔵                                                  |

---

## Prototype.js の他の特徴

* メソッド名などを見る限り、かなり Ruby に影響を受けている
* 今回紹介する `Class.create` も、コンストラクタを定義するプロパティが `initialize`

---

## Let's do Code Reading!

* といきたいところですが、まず基本をおさらい
* `Class.create` をリーディングしたいので JavaScript の Prototype チェーンについておさらいしておきます

---

## prototype

* **コンストラクタ**のプロパティ
* そのコンストラクタによって生成されたインスタンスに対して存在しないプロパティを参照したとき、代わりに `prototype` に定義されているプロパティが参照される

```js
function Class() {}
Class.prototype = { foo: 'foo' };

var instance = new Class();
instance.foo;    // 'foo'

instance.foo = 'bar';
instance.foo;           // 'bar'
Class.prototype.foo;    // 'foo'
```

---

## Object.getInstanceOf, \_\_proto\_\_

* 途中からインスタンス側からも `Object.getPrototypeOf` で参照できるようになった
* ちなみに `__proto__` は非推奨だが、このスライドでは説明の便宜上使う。

```js
function Class() {}

var instance = new Class();
Object.getPrototypeOf(instance) === Class.prototype;    // true
instance.__proto__ === Class.prototype;                 // true
```

---

## すべての道は Object に通ず

* `prototype` 自体は `Object` で定義されることが多い
* `Class` インスタンス `instance` のプロパティ `prop` を参照すると次の順に参照される
  * `instance.prop`
  * `instance.__proto__.prop`
  * **`instance.__proto__.__proto__.prop`**

```js
function Class() {}
Object.prototype = { prop: 'foo' };

var instance = new Class();
instance.prop;    // 'foo'
```

* Prototype.js ではこの性質を利用しクラス継承の仕組みを実装する

---

## prototype.constructor について

- 何も指定しなければコンストラクタ自身が代入されている
- 上書きすることもできる
```js
Class.prototype.constructor = Object;
```
- ただし `instanceof` は欺けない
```js
instance instanceof Class;    // true
```

---

## Vanilla JS でのクラス継承

```js
function ParentClass() {}

function ChildClass() {}
// prototype の prototype を変更していることに注意!
Object.setPrototypeOf(ChildClass.prototype, ParentClass.prototype);
```

---

## prototype を使った小技

* Function 内部の `arguments` は Array-like なオブジェクトだが Array ではないた
  め `Array.prototype` が使えない
* `__proto__` を無理やり `Array.prototype` にしてしまうことで使えるメソッド**もある**
  ```js
  arguments.__proto__ = Array.prototype;
  arguments.shift();
  ```
* その他に `slice` を無理やり使って `Array` に変換するテクニックもある
  ```js
  var args = Array.prototype.slice.call(arguments);
  args.shift();
  ```
  
---

## prototype 定義の弊害

```js
var array = [0, 1, 2];

Array.prototype.foo = 'foo';

for (var key in array) {
  key;    // <- 0, 1, 2, 'foo'
}
```
  
---

## prototype 定義の弊害

`.hasOwnProperty` を使おう! (というか Array を for-in で回すのをやめよう)

```js
for (var key in array) {
  if (array.hasOwnProperty(key)) {
    key;    // <- 0, 1, 2
  }
}
```

---

## Don't Enum 属性

- よくよく考えると built-in のメソッドは for-in しても列挙されない
- この定義されているのに列挙されない属性を **Don't Enum 属性** と呼ぶ

---

## コードリーディング

---

## Class.create  の仕様

* クラスメソッドを定義したオブジェクトを引数として渡す
* `initialize` はコンストラクタとして扱われる

```js
var Animal = Class.create({
  initialize: function(name, sound) {
    this.name  = name;
    this.sound = sound;
  },

  speak: function() {
    alert(this.name + " says: " + this.sound + "!");
  }
});
```

---

## Class.create の仕様

* 第一引数に別のクラスを渡すことによってそのクラスを親とする子クラスを生成できる
* メソッドの第一引数の名前を `$super` にすると、それをスーパーメソッドとして扱える。

```js
var Snake = Class.create(Animal, {
  initialize: function($super, name) {
    $super(name, 'hissssssssss');
  }
});
```

```js
var ringneck = new Snake("Ringneck");
ringneck.speak();
//-> alerts "Ringneck says: hissssssssss!"
```

---

## Class.create

1. 第一引数が Function であればそれを親クラスとみなす。他は定義するメソッドとみなす。
   ```js
   var parent = null, properties = $A(arguments);

     をもったオブジェクト
   if (Object.isFunction(properties[0]))
     parent = properties.shift();
   ```

1. コンストラクタでは単純に initialize を呼び出す
   ```js
   function klass() {
     this.initialize.apply(this, arguments);
   }
   ```
   
---

## Class.create

1. コンストラクタに対してメソッドを追加する。これにより addMethods() が追加される。
   ```js
   Object.extend(klass, Class.Methods);
   ```
1. コンストラクタにクラスのメタデータを格納
   ```js
   klass.superclass = parent;
   klass.subclasses = [];
   ```

---

## Class.create

prototype の prototype を親クラスの prototype にする

```js
if (parent) {
  subclass.prototype = parent.prototype;
  klass.prototype = new subclass;
}
```

つまり `subclass` は `parent.prototype` を prototype にもつオブジェクトを作るための使い回しコンストラクタ。
ブラウザでサポートされていれば、次のいずれかのように書ける。

```js
klass.prototype = Object.create(parent.prototype);
Object.setPrototypeOf(klass.prototype, parent.prototype)
klass.prototype.__proto__ = parent.prototype;
```

---

## Class.create

クラスメソッドを追加する

```js
for (var i = 0, length = properties.length; i < length; i++)
  klass.addMethods(properties[i]);
```

---

## Class.Methods.addMethods

既に Don't Enum 属性が付与されている built-in プロパティと同名のプロパティを定義したときに for-in で列挙されないバグに対する workaround

```js
var IS_DONTENUM_BUGGY = (function(){
  for (var p in { toString: 1 }) {
    if (p === 'toString') return false;
  }
  return true;
})();
```

```js
if (IS_DONTENUM_BUGGY) {
  if (source.toString != Object.prototype.toString)
    properties.push("toString");
  if (source.valueOf != Object.prototype.valueOf)
    properties.push("valueOf");
}
```

---

## Class.Methods.addMethods

```js
ancestor && Object.isFunction(value) && value.argumentNames()[0] == "$super"
```

---

## Function.prototype.toString

Function を `toString()` するとソースコードが文字列で返る

```js
function f(x) { x; }
f.toString()    // 'function f(x) { x; }'
```

ただし built-in な関数や Command Line API などの関数の中身は見れない

```js
Object.toString();    // 'function() { [native code] }'
$$.toString();        // 'function() { [Command Line API] }'
```

---

## Function.prototype.argumentNames

これを利用して引数を格納する変数名を正規表現で取得する (マジか)

```js
function argumentNames() {
  var names = this.toString().match(/^[\s\(]*function[^(]*\(([^)]*)\)/)[1]
    .replace(/\/\/.*?[\r\n]|\/\*(?:.|[\r\n])*?\*\//g, '')
    .replace(/\s+/g, '').split(',');
  return names.length == 1 && !names[0] ? [] : names;
}
```

---

## Class.Methods.addMethods

* `Function.prototype.wrap` は引数の先頭から分割代入を行うための独自メソッド (ここでは第一引数に `$super` を分割代入)
* 即時関数を使ってメソッドを再定義しているのはなぜだろう

```js
var method = value;
value = (function(m) {
  return function() { return ancestor[m].apply(this, arguments); };
})(property).wrap(method);
```

---

## 変数のスコープ

```js
var f = new Array(3);

for (var i = 0; i < 3; i++) {
  f[i] = function() { return i; };
}

f[0]();    // 0 ではなく 2 が返る
```

* 上記のスクリプトでは、変数 `i` はループを実行したスコープを escape した後も生存している
* ただし値は**ループ終了時の値**になっている

---

## 変数のスコープ

* 各ループがスコープを共有しているのが問題
* ならば**各ループの中で新しいスコープを作ってその時点での値をコピー**すればいい

```js
var f = new Array(3);

for (var i = 0; i < 3; i++) {
  f[i] = (function(j) {
    return function() { return j; };
  })(j);
}

f[0]();    // 0
```

---

## Class.Methods.addMethods

最後に `toString()` したときにラップする前の関数のソースコードを返すようにする (細かい...)

```js
value.valueOf = (function(method) {
  return function() { return method.valueOf.call(method); };
})(method);

value.toString = (function(method) {
  return function() { return method.toString.call(method); };
})(method);
```

---

## まとめ

* JavaScript なんもわからん...

---

# おしまい
