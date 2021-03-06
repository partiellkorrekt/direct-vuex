# direct-vuex

[![Build Status](https://travis-ci.com/paleo/direct-vuex.svg?branch=master)](https://travis-ci.com/paleo/direct-vuex)
[![Dependencies Status](https://david-dm.org/paleo/direct-vuex/status.svg)](https://david-dm.org/paleo/direct-vuex)
[![Codacy status](https://api.codacy.com/project/badge/Grade/c62abca7334941b5ac75e713cd60a0a4)](https://www.codacy.com/manual/paleo/direct-vuex?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=paleo/direct-vuex&amp;utm_campaign=Badge_Grade)
[![npm](https://img.shields.io/npm/dm/direct-vuex)](https://www.npmjs.com/package/direct-vuex)
![Type definitions](https://img.shields.io/npm/types/direct-vuex)
[![GitHub](https://img.shields.io/github/license/paleo/direct-vuex)](https://github.com/paleo/direct-vuex)

Use and implement your Vuex store with TypeScript types. Direct-vuex doesn't require classes, therefore it is compatible with the Vue 3 composition API.

**_Warning! BREAKING CHANGE in version 0.8: do not use `as const` with `createDirectStore`. Additionally, there is a new limitation on [how to declare a state](#a-limitation-on-how-to-declare-a-state). See also the section: [Implement a Vuex Store with typed helpers](#implement-a-vuex-store-with-typed-helpers)._**

## Install

First, add `direct-vuex` to a Vue application:

```sh
npm install direct-vuex
```

## Create the store

The store can be implemented almost in the same way as usual.

Create the store:

```ts
import Vue from "vue"
import Vuex from "vuex"
import { createDirectStore } from "direct-vuex"

Vue.use(Vuex)

const {
  store,
  rootActionContext,
  moduleActionContext,
  rootGetterContext,
  moduleGetterContext
} = createDirectStore({
  // … store implementation here …
})

// Export the direct-store instead of the classic Vuex store.
export default store

// The following exports will be used to enable types in the
// implementation of actions and getters.
export {
  rootActionContext,
  moduleActionContext,
  rootGetterContext,
  moduleGetterContext
}

// The following lines enable types in the injected store '$store'.
export type AppStore = typeof store
declare module "vuex" {
  interface Store<S> {
    direct: AppStore
  }
}
```

The classic Vuex store is still accessible through the `store.original` property. We need it to initialize the Vue application:

```ts
import Vue from "vue"
import store from "./store"

new Vue({
  store: store.original, // Inject the classic Vuex store.
  // …
}).$mount("#app")
```

## Use typed wrappers from outside the store

From a component, the direct store is accessible through the `direct` property of the classic store:

```ts
const store = context.root.$store.direct // or: this.$store.direct
```

Or, you can just import it:

```ts
import store from "./store"
```

Then, the old way to call an action:

```ts
store.dispatch("mod1/myAction", myPayload)
```

… is replaced by the following wrapper:

```ts
store.dispatch.mod1.myAction(myPayload)
```

… which is fully typed.

Typed getters and mutations are accessible the same way:

```ts
store.getters.mod1.myGetter
store.commit.mod1.myMutation(myPayload)
```

Notice: The underlying Vuex store can be used simultaneously if you wish, through the injected `$store` or `store.original`.

## A limitation on how to declare a State

In store and module options, the `state` property shouldn't be declared with the ES6 method syntax.

Valid:

```ts
  state: { p1: string } as Mod1State
```

```ts
  state: (): Mod1State => { p1: string }
```

```ts
  state: function (): Mod1State { return { p1: string } }
```

Invalid:

```ts
  state(): Mod1State { return { p1: string } }
```

I'm not sure why but TypeScript doesn't infer the state type correctly when we write that.

## Implement a Vuex Store with typed helpers

Direct-vuex provides several useful helpers for implementation of the store. They are all optional. However, if you want to keep your classic implementation of a Vuex Store, then direct-vuex needs to infer the literal type of the `namespaced` property. You can write `namespaced: true as true` where there is a `namespaced` property. But you don't need to worry about that if you use `defineModule`.

### In a Vuex Module

The function `defineModule` is provided solely for type inference. It is a no-op behavior-wise. It expects a module implementation and returns the argument as-is. This behaviour is similar to (and inspired from) the [function `defineComponent`](https://vue-composition-api-rfc.netlify.com/api.html#definecomponent) from the composition API.

The generated function `moduleActionContext` is a factory for creating a function `mod1ActionContext`, which converts the injected action context to the direct-vuex one.

Here is how to use `defineModule` and `moduleActionContext`:

```ts
import { defineModule } from "direct-vuex"
import { moduleActionContext } from "./store"

export interface Mod1State {
  p1: string
}

const mod1 = defineModule({
  state: (): Mod1State => {
    return {
      p1: ""
    }
  }
  getters: {
    p1OrDefault(...args) {
      const { state, getters, rootState, rootGetters } = mod1GetterContext(...args(state, ...args)
      // Here, 'getters', 'state', 'rootGetters' and 'rootState' are typed.
      // Without 'mod1GetterContext' only 'state' would be typed.
      return state.p1 || "default"
    }
  },
  mutations: {
    SET_P1(state, p1: string) {
      // Here, the type of 'state' is 'Mod1State'.
      state.p1 = p1
    }
  },
  actions: {
    loadP1(context, payload: { id: string }) {
      const { dispatch, commit, getters, state } = mod1ActionContext(context)
      // Here, 'dispatch', 'commit', 'getters' and 'state' are typed.
    }
  },
})

export default mod1
const mod1ActionContext = (context: any) => moduleActionContext(context, mod1)
const mod1GetterContext = (state: any, getters: any, rootState: any, rootGetters: any) => moduleGetterContext(state, getters, rootState, rootGetters, system)
```

Warning: Types in the context of actions implies that TypeScript should never infer the return type of an action from the context of the action. Indeed, this kind of typing would be recursive, since the context includes the return value of the action. When this happens, TypeScript passes the whole context to `any`. _Tl;dr; Declare the return type of actions where it exists!_

### Get the typed context of a Vuex Action, but in the root store

The generated function `rootActionContext` converts the injected action context to the direct-vuex one, at the root level (not in a module).

```ts
  actions: {
    async actionInTheRootStore(context, payload) {
      const { commit, state } = rootActionContext(context)
      // … Here, 'commit' and 'state' are typed.
    }
  }
```

### Use `defineGetters`

The function `defineGetters` is provided solely for type inference. It is a no-op behavior-wise. It is a factory for a function, which expects the object of a `getters` property and returns the argument as-is.

```ts
import { defineGetters } from "direct-vuex"
import { Mod1State } from "./mod1" // Import the local definition of the state (for example from the current module)

export default defineGetters<Mod1State>()({
  getter1(state) {
    const { state, getters, rootState, rootGetters } = mod1GetterContext(...args(state, ...args)
    // Here, 'getters', 'state', 'rootGetters' and 'rootState' are typed.
    // Without 'mod1GetterContext' only 'state' would be typed.
  },
})
```

Note: There is a limitation. The second parameters `getters` in a getter implementation, is not typed.

### Use `defineMutations`

The function `defineMutations` is provided solely for type inference. It is a no-op behavior-wise. It is a factory for a function, which expects the object of a `mutations` property and returns the argument as-is.

```ts
import { defineMutations } from "direct-vuex"
import { Mod1State } from "./mod1" // Import the local definition of the state (for example from the current module)

export default defineMutations<Mod1State>()({
  SET_P1(state, p1: string) {
    // Here, the type of 'state' is 'Mod1State'.
    state.p1 = p1
  }
})
```

### Use `defineActions`

The function `defineActions` is provided solely for type inference. It is a no-op behavior-wise. It expects the object of an `actions` property and returns the argument as-is.

```ts
import { defineActions } from "direct-vuex"

export default defineActions({
  loadP1(context, payload: { id: string }) {
    const { dispatch, commit, getters, state } = mod1ActionContext(context)
    // Here, 'dispatch', 'commit', 'getters' and 'state' are typed.
  }
})
```

## About Direct-vuex and Circular Dependencies

When the helper `moduleActionContext` is used, linters may warn about an issue: _"Variable used before it's assigned"_. I couldn't avoid circular dependencies. Module action contexts need to be inferred at the store level, because they contain `rootState` etc.

Here is an example of a Vuex module implementation:

```ts
import { moduleActionContext } from "./store"

const mod1 = {
  getters: {
    p1OrDefault(...args) {
      const { state, getters, rootState, rootGetters } = mod1GetterContext(...args(state, ...args)
      // …
    }
  },
  actions: {
    loadP1(context, payload: { id: string }) {
      const { commit, rootState } = mod1ActionContext(context)
      // …
    }
  }
}

export default mod1
const mod1ActionContext = (context: any) => moduleActionContext(context, mod1)
const mod1GetterContext = (state: any, getters: any, rootState: any, rootGetters: any) => moduleGetterContext(state, getters, rootState, rootGetters, system)
```

It works because `mod1ActionContext` is not executed at the same time it is declared. It is executed when an action is executed, ie. after all the store and modules are already initialized.

I suggest to disable the linter rule with a comment at the top of the source file.

With TSLint:

```js
// tslint:disable: no-use-before-declare
```

With ESLint:

```js
/* eslint-disable no-use-before-define */
```

**Notice: A consequence of these circular dependencies is that _the main store file must be imported first_ from the rest of the application. If a Vuex module is imported first, some part of your implementation could be `undefined` at runtime.**

## Contribute

With VS Code, our recommanded plugin is:

* **TSLint** from Microsoft (`ms-vscode.vscode-typescript-tslint-plugin`)
