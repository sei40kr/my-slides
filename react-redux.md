# React + Redux で開発がしたかった

---

## 自己紹介

* よんじゅ。
* Twitter [@sei40kr](https://twitter.com/sei40kr) にはしょうもないことしか書きません。

---

## なぜ React なのか

* フレームワークではなく、ライブラリ。
* 独自の構文をもたない。
* ライブラリなので、その他のライブラリと親和性が高い。

---

## 内部ステート vs 外部ステート

---

### 内部ステート

#### コンポーネントの生死がステートの生死

コンポーネントに紐付けるステート。コンポーネントのライフサイクルやイベントの発火に応じて変更する。

```jsx
this.setState({
  hoge: 'fuga',
});
```

---

* コンポーネントの外側から参照できない。親から参照したければ、そのステートは子にもたせずに、親がもって子に伝搬する。

```jsx
render() {
  <form onSubmit={this.onSubmit()}>
    <CustomInput id="..." value={this.state.value} onChange={this.props.onInputChange()} />
  </form>
}

onInputChange(event) {
  this.setState({
    value: event.target.value,
  });
}

onSubmit() {
  sendInputData(this.state.value)
}
```

---

### 外部ステート

---

#### ステートによってコンポーネントの生死を決める

---

* コンポーネントが消えてもステートは残る。内部ステートに慣れてしまうと事故が起こりやすい。
* ダイアログを開くと前回の入力内容がクリアされていない、など。

---

#### Redux

* Redux は簡単に言うと、この外部ステートを結合して単一のグローバルステートにしてしまうもの。

---

## React の罠

---

### Prop について

* Prop はタグの属性とは少し違う。DOM オブジェクトのプロパティ。
* でも基本的にタグの属性と認識していいレベル。
* 例えば、 `class` と `for` は言語の予約語なのでプロパティとして存在しない。`className` と `htmlFor` を使う。

```jsx
<div className="">
  <label htmlFor="" />
</div>
```

---

### "key" prop

```jsx
<ListView items={[1, 3, 4, 5]} />
```

```jsx
this.props.items.forEach(item => <ChildView {...item} />)
```

```html
<div className="child-view">This is 1st item</div>
<div className="child-view">This is 3rd item</div>
<div className="child-view">This is 4th item</div>
<div className="child-view">This is 5th item</div>
```

---

#### 要素が増えたらどうする?

---

#### 末尾に追加したケース

```jsx
// 6 を追加
<ListView items={[1, 3, 4, 5, 6]} />
```

```html
<div className="child-view">This is 1st item</div>
<div className="child-view">This is 3rd item</div>
<div className="child-view">This is 4th item</div>
<div className="child-view">This is 5th item</div>
<!-- この DOM を追加すれば良い -->
<div className="child-view">This is 6th item</div>
```

---

#### 末尾以外に追加したケース

```jsx
// 2 を追加
<ListView items={[1, 2, 3, 4, 5]} />
```

```html
<div className="child-view">This is 1st item</div>
<!-- ここから DOM の書き換え開始/表示するアイテムを1つずつズラす -->
<div className="child-view">This is 2st item</div>
<div className="child-view">This is 3rd item</div>
<div className="child-view">This is 4th item</div>
<!-- この DOM は追加されたもの -->
<div className="child-view">This is 5th item</div>
```

---

#### 無駄な DOM の変更が発生

---

#### 繰り返し表示するコンポーネントには `key` Prop を付与

```jsx
this.props.items.forEach(item => <ChildView key={item.id} {...item} />)
```

```html
<div className="child-view">This is 1st item</div>
<!-- この DOM は追加されたもの -->
<div className="child-view">This is 2st item</div>
<div className="child-view">This is 3rd item</div>
<div className="child-view">This is 4th item</div>
<div className="child-view">This is 5th item</div>
```

---

### "className" prop

* Use COMPONENT_ID as a prefix to avoid confliction.
* Use "classNames()" to accept classnames as a component prop.

---

### How to do something with form values?

* Dispatch an action from input event and update Redux state.

---

### How to bind event?

---

### Whose action is it?

* Which component fires?
* Which component to know?

---

## Redux の罠

---

### "getState" or "mergeProps"?

* "getState" seems simpler, but how to test it?
* The winner is "mergeProps".

---

### Asynchronous processing

* Use action-dispatcher.

---

### 複数のアクションを Dispatch する



* Create middleware or use "redux-thunk"?
* Is it neccessary to be dispatchable?

---

### Action Dispatcher の単体テスト

#### Action Creator をテストするのは簡単

```javascript
const hogeAction = (p) => ({
  type: 'HOGE_ACTION',
  payload: 'fuga',
});
```

```javascript
expect(hogeAction('fuga')).toBe({
  type: 'HOGE_ACTION',
  payload: 'fuga',
});
```

---

#### Action Dispatcher はどうテストする?

```javascript
const hogeActionDispatcher = (p1, p2) => (dispatch) => {
  dispatch(hogeAction(p1));
  dispatch(hogehogeAction(p2));
};
```

---

#### スパイ Dispatcher を作りましょう。

```javascript
const spyDispatchLog = [];

const spyDispatch = (o) => {
  if (typeof o === 'function') {
    o(spyDispatch);
  } else {
    spyDispatchLog.push(o);
  }
};

expect(hogeActionDispatcher(spyDispatch)('fuga', 'fugafuga')).toBe([{
  type: 'HOGE_ACTION',
  payload: 'fuga',
}, {
  type: 'HOGEHOGE_ACTION',
  payload: 'fugafuga',
}]);
```

---

### State structure and file/directory structure

---

### Which is it related, domain or page?

---

### Normalize the state

---

* Normalize a list by record IDs.
* Memoize using "reselect".

---

## 型チェック

---

### Typescript

* 公式の Language Server が充実。
* NPM レポジトリにモジュールの型定義ファイルも充実。

---

### Flow

* オプトイン設計。プロジェクトに簡単に導入でき、簡単に捨てられる。
* ファイル単位での導入が可能。
* ESLint も使える。

---

### Why type checking is needed for React development?

---

### How to add type annotations for React + Redux?

* How to define types of a component event?
* How to cast nullable to non-null type?
* Use an union type to merge parent props and child props.
* Prohibit reassignments of the state using type definition.
* Cast the type of an action in reducer to payload.
