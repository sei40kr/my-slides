# React + Redux で開発がしたかった

---

## 自己紹介

* よんじゅ。
* Twitter [@sei40kr](https://twitter.com/sei40kr) にはしょうもないことしか書きません。

---

### 今回のスライド

https://github.com/sei40kr/my-slides

---

## なぜ React なのか

* フレームワークではなく、ライブラリ。
* 独自の構文をもたない。
* ライブラリなので、その他のライブラリと親和性が高い。

---

## 内部ステート vs 外部ステート

---

### 内部ステート

---

#### 内部ステート

コンポーネントに紐付けるステート。コンポーネントのライフサイクルやイベントの発火に応じて変更する。

```jsx
this.setState({
  hoge: 'fuga',
});
```

---

#### ステートの参照

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

##### Props を伝搬するだけの簡単なお仕事です

---

#### コンポーネントの生死がステートの生死

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

### フォームの値をアクションに埋め込む

---

#### redux-thunk という選択肢

```javascript
const loginAction = () => (dispatch, getState) => {
  const { userName, password } = getState().application.loginForm;
};
```

---

#### `mergeProps` という選択肢

シンプルでテストしやすい。

```javascript
const mapStateToProps = (state) => ({
  userName: state.application.loginForm.userName,
  password: state.application.loginForm.password,
});

const dispatchProps = (dispatch) => bindActionCreators({
  onClickLoginButton: loginAction,
})

const mergeProps = (stateProps, dispatchProps, ownProps) => ({
  ...stateProps,
  ...dispatchProps,
  ...ownProps,
  onClickLoginButton: dispatchProps.onClickLoginButton(stateProps.userName, stateProps.password),
});
```

---

### ステートを扱うイベントを定義

---

```jsx
// Don't
<button onClick={(e) => this.props.onClickButton(this.state.inputValue)}>
  Click me!
</button>
```

---

```jsx
constructor() {
  this.onClickButton = this.onClickButton.bind(this);
}

onClickButton() {
  this.props.onClickButton(this.state.inputValue)
}

render() {
  return (
    <button onClick={this.onClickButton}>
      Click me!
    </button>
  );
}
```

---

## Redux の罠

---

### 非同期処理

---

#### 処理が終わった後にディスパッチする

```javascript
const fetch = (dispatch) => fetch('/path/to/remote')
    .then((response) => dispatch(fetchSuccess(response)));
```

---

### Action Dispatcher の単体テスト

#### Action Creator をテストするのは簡単

```javascript
const fetchAction = (targetDate) => ({
  type: 'FETCH_ACTION',
  payload: targetDate,
});
```

```javascript
expect(fetchAction('2017-03-27')).toBe({
  type: 'FETCH_ACTION',
  payload: '2017-03-27',
});
```

---

#### Action Dispatcher はどうテストする?

```javascript
const hogeActionDispatcher = (targetDate) => (dispatch) => {
  dispatch(showLoadingSpinnerAction());
  dispatch(fetchAction(targetDate));
  dispatch(hideLoadingSpinnerAction());
};
```

---

#### スパイ Dispatcher を作りましょう

```javascript
const spyDispatchLog = [];

const spyDispatch = (o) => {
  if (typeof o === 'function') {
    o(spyDispatch);
  } else {
    spyDispatchLog.push(o);
  }
};

expect(hogeActionDispatcher('2017-03-27')(spyDispatch)).toBe([{
  type: 'SHOW_LOADING_SPINNER_ACTION',
}, {
  type: 'FETCH_ACTION',
  payload: '2017-03-27',
}, {
  type: 'HIDE_LOADING_SPINNER_ACTION',
}]);
```

---

### ビュー or ドメイン

---

#### コンポーネントやモジュールにどのような階層をもたせる?

---

#### 労務管理アプリの例

* 勤怠 (Attendance) -> ドメインの関心事
* 勤務表 (Timesheet) -> ビューの関心事

---

#### ビュー or ドメイン

* 打刻ウィジェット -> 勤怠 (ドメイン) の関心事
* 勤務表のヘッダー -> 勤務表 (ビュー) の関心事

---

#### ビューに紐づくコンポーネント/モジュール

* 1つのビューから参照される

---

#### ドメインに紐づくコンポーネント/モジュール

* 複数のビューから参照される

---

#### 真の汎用コンポーネント/モジュール

* 複数の画面で使うコンポーネントではない。
* 製品のコアドメインに関係のないコンポーネント。 (Button, Icon, user-setting)

---

### 理想のディレクトリ構造

---

```text
- project/
  - domain/
    - attendance/
      - modules/
  - application/
    - common/
      - components/
        - button.js
      - modules/
        - user-setting.js
    - timesheet-for-pc/
      - components/
      - modules/
        - daily-record.js
        - pending-request.js
```

---

### ステートの正規化

---

#### ステートにあるリストアイテムの選択

---

#### 非正規ステート

* 配列の添字番号で取得。
* ディスパッチした時点と実行時で添字が変わっていないか...?

```jsx
// No
const getListItem = (state, index) => ({
  return state.entities[index];
}); 
```

---

#### 正規ステート

* アイテム固有のIDで取得。
* 実行タイミングに疑心暗鬼にならなくてよい。
* アイテム間に参照関係をもたせるのに便利!

```jsx
// Yes
const getListItem = (state, id) => {
  return state.entity.byId[id];
}; 
```

---

#### バックエンドには DB のIDを返してもらう

* 分散KVSなどでは UUIDv4 を返す。
* シーケンスで管理しているRDBならシーケンス番号を返す。
* ステートの正規化ができて、コンポーネントの `key` にも使える!

---

#### 配列を正規化しよう

* 僕はアクションの作成前にやっています。
* Reducer で複雑なステートの更新をやる際に早速役に立つ。
* 以下、滑らない配列の正規化の実装。

```jsx
const convertEntities = (entities) => ({
  allIds: entities.map(entity => entity.id),
  byId: Object.assign({}, entities.map(entity => ({ [entity.id]: entity }))),
});
```

---

#### メモ化しつつ非正規化する

* ビューで扱う際には非正規に戻しておく。
* `reselect` を使うことによって、必要時だけキャッシュを更新する非正規化関数を作れる。
* 以下、滑らない正規ステートの非正規化。

```jsx
import { createSelector } from 'reselect';

const entitiesSelector = createSelector(
  state => state.entity.allIds,
  state => state.entity.byId,
  (allIds, byId) => allIds.map(id => byId[id])
)
```

```jsx
const mapStateToProps = (state) => ({
  entities: entitiesSelector(state),
});
```

### アクションが Reduce されたときに別のアクションを Dispatch したい

#### 例

リストでアイテムを選択した時に詳細を表示したい。 (詳細を取得するAPIにリクエストしたい)

#### 諦めるという選択肢

* 依存性のあるアクションをまとめる。
* よっぽどのことがない限りこちらを推奨。

```jsx
const selectItemAndFetchDetailsDispatcher = (targetId) => (dispatch) => {
  dispatch(selectItem());
  dispatch(fetchDetailsDispatcher());
};
```

#### 自作ミドルウェアという選択肢

* よっぽどのことがあれば。

```jsx
const fetchDetailsOnSelectItemMiddleware = ({ dispatch }) => (next) => (action) => {
  next(action);

  if (action.type === 'SELECT_ITEM') {
    dispatch(fetchDetailsDispatcher());
  }
};

```

### その他

* ステートの再代入。
* コンテナーの更新条件。

---

## 型チェック

---

### Typescript

* 公式の言語サーバーが充実。
* NPM レポジトリにモジュールの型定義ファイルも充実。

---

### Flow

* オプトイン設計。プロジェクトに簡単に導入でき、簡単に捨てられる。
* ファイル単位での導入が可能。
* ESLint も使える。
* 大規模プロジェクトをスキャンすると非力なノートPCが暖房器具に。

---

### React + Redux で型をどう使う?

---

#### Props の型定義

---

##### 子コンポーネント

* ジェネリクスを使って `propTypes` を定義。
* `export` しておこう。

```jsx
export type Props = {
  childProp: string,
};

export default class ChildComponent extends React.Component<Props> {
}
```

---

##### 親コンポーネント

* 型合成を使って伝搬する `propTypes` の定義を楽にする。

```jsx
export type Props = ChildProps & {
  parentProp: string,
};

export default class ParentComponent extends React.Component<Props> {
}
```

---

#### アクション

```javascript
type FetchAction = {
  type: 'FETCH_ACTION',
};

type FetchSuccessAction = {
  type: 'FETCH_SUCCESS_ACTION',
  payload: {
    allIds: string[],
    byId: { [string]: Entity },
  }
};

export type Action = FetchAction | FetchSuccessAction;
```

---

### Reducer

* `payload` を使う場合はアクションが判明した時点でキャストする。 

```javascript
const reducer = (state = initialState, action: Action) => {
  switch (action.type) {
    case 'FETCH_SUCCESS_ACTION':
      return {
        ...state,
        entity: (action: FetchSuccessAction).payload,
      };

    default:
      return state;
  }
}
```

---

## まとめ

---

### React は素晴らしい

---

### んが

---

### フロントエンドはつらいよ

---

## Thank you for listening!
