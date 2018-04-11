# React + Redux

---

## About me

* A little self introduction.

---

## Why I love React

### Not a framework, but a library

* No syntax sugar of its own.
* You can use the modules you like!

---

## Difference between local state, store, and global state

* How is lifecycle of each?

---

## Traps in React

---

### Special attributes

* Props are not HTML attributes!
* For instances, "className" and "htmlFor" because they're reserved words in JavaScript.

---

### "key" prop

* Rerender all components?
* Levenshtein distance?
* You can use UUID v1. 

---

### "className" prop

* Use COMPONENT_ID as a prefix to avoid confliction.
* Use "classNames()" to accept classnames as a component prop.

---

### How to do something with form values?

* Dispatch an action from input event and update Redux state.

---

### "display: none" vs "null"

* "autoFocus" prop.

---

### Whose action is it?

* Which component fires?
* Which component to know?

---

## Traps in Redux

---

### "getState" or "mergeProps"?

* "getState" seems simpler, but how to test it?
* The winner is "mergeProps".

---

### Asynchronous processing

* Use action-dispatcher.

---

### How to dispatch actions on action?

* Create middleware or use "redux-thunk"? 
* Is it neccessary to be dispatchable?

---

### How to test action-dispatchers?

* Let's create a spy dispatcher to test. It's very easy!

---

### State structure and file/directory structure

---

### Which is it related, domain or page?

---

### Normalize the state

* Normalize a list by record IDs.
* Memoize using "reselect".

---

## Type checking

---

### Typescript

* There're good language tools!
* There're also many type definitions!

---

### Flow

* Easy to install. Easy to uninstall.
* You can still use ESLint.

---

### Why type checking is needed for React development?

---

### How to add type annotations for React + Redux?

* How to define types of a component event?
* How to cast nullable to non-null type?
* Use an union type to merge parent props and child props.
* Prohibit reassignments of the state using type definition.
* Cast the type of an action in reducer to payload.
