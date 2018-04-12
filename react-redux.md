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

---

#### 内部ステート

コンポーネントに紐付けるステート。コンポーネントのライフサイクルやイベントの発火に応じて変更する。

```jsx
this.setState({
  hoge: 'fuga',
});
```

---

#### State reference

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

### Dispatch an action with a form value

---

#### redux-thunk is one solution

```javascript
const loginAction = () => (dispatch, getState) => {
  const { userName, password } = getState().application.loginForm;
};
```

---

#### "mergeProps" seems better

Simpler and more testable.

```javascript
const mapStateToProps = (state) => ({
  userName: state.application.loginForm.userName,
  password: state.application.loginForm.password,
});

const dispatchProps = (dispatch) => bindActionCreators({
  onClickLoginButton: loginAction,
})

const mergeProps = (stateProps, dispatchProps, ownProps) => ({
  onClickLoginButton: dispatchProps.onClickLoginButton(stateProps.userName, stateProps.password),
});
```

---

### Bind event with component context

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
  this.onClickButton = this.props.onClickButton.bind(this);
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

### Asynchronous processing

---

#### Dispatch after the processing

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

expect(hogeActionDispatcher(spyDispatch)('2017-03-27')).toBe([{
  type: 'SHOW_LOADING_SPINNER_ACTION',
}, {
  type: 'FETCH_ACTION',
  payload: '2017-03-27',
}, {
  type: 'HIDE_LOADING_SPINNER_ACTION',
}]);
```

---

### Domain or view?

---

#### Domain module

* Referred from multiple views.
* e.g. Attendance

---

#### View modules

* Referred from single views.
* e.g. Timesheet

---

### Ideal directory structure

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

### Normalize state

---

#### How to select a list item?

---

#### Un-normalized state

```jsx
// No
const getListItem = (state, index) => ({
  return state.entities[index];
}); 
```

---

#### Normalize state

```jsx
// Yes
const getListItem = (state, id) => {
  return state.entity.byId[id];
}; 
```

---

#### Implement a converter

```jsx
const convertEntities = (entities) => ({
  allIds: entities.map(entity => entity.id),
  byId: Object.assign({}, entities.map(entity => ({ [entity.id]: entity }))),
});
```

---

#### Use reselect to memoize

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

### How to add type annotations for React + Redux?

---

#### Define types of a component event

---

##### In child component

```jsx
export type Props = {
  childProp: string,
};

export default class ChildComponent extends React.Component<Props> {
}
```

---

##### In parent component

Use union type to merge type definitions.

```jsx
export type Props = ChildProps & {
  parentProp: string,
};

export default class ParentComponent extends React.Component<Props> {
}
```

---

#### Define the type of action

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

### Define the type of reducer

```javascript
const reducer = (state = initialState, action: Action) => {
  switch (action.type) {
    case 'FETCH_SUCCESS_ACTION':
      return {
        ...state,
        entity: (action:FetchSuccessAction).payload,
      };

    default:
      return state;
  }
}
```
