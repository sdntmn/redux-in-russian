---
id: usage-with-typescript
title: Usage With TypeScript
sidebar_label: Usage With TypeScript
hide_title: true
---

&nbsp;

# Usage With TypeScript (Использование с TypeScript)

- Подробная информация о том, как использовать каждый API-интерфейс Redux Toolkit с TypeScript

## Introduction (Введение)

Redux Toolkit написан на TypeScript, а его API разработан для обеспечения отличной интеграции с приложениями TypeScript.

На этой странице приведены конкретные сведения о каждом из различных API, включенных в Redux Toolkit, и о том, как правильно вводить их с помощью TypeScript.

**Смотрите [TypeScript Quick Start tutorial page](../tutorials/typescript.md) для получения краткого обзора того, как настроить и использовать Redux Toolkit и React Redux для работы с TypeScript.**.

:::info

Если у вас возникнут какие-либо проблемы с типами, которые не описаны на этой странице, пожалуйста [open an issue](https://github.com/reduxjs/redux-toolkit/issues?q=is%3Aissue+is%3Aopen+sort%3Aupdated-desc) для обсуждения.

:::

## `configureStore`

Основы использования configureStoreприведены на [TypeScript Quick Start tutorial page](../tutorials/typescript.md). Вот некоторые дополнительные сведения, которые могут оказаться вам полезными.

### Getting the `State` type (Получение State типа)

Самый простой способ получить `State` тип - заранее определить корневой редуктор и извлечь его `ReturnType`.
Рекомендуется присвоить типу другое имя, `RootState` чтобы избежать путаницы, поскольку имя типа `State` обычно используется слишком часто.

```typescript
import { combineReducers } from '@reduxjs/toolkit'
const rootReducer = combineReducers({})
// highlight-start
export type RootState = ReturnType<typeof rootReducer>
// highlight-end
```

В качестве альтернативы, если вы решите не создавать a `rootReducer` самостоятельно, а вместо этого передавать редукторы фрагментов напрямую `configureStore()`, вам нужно немного изменить типизацию, чтобы правильно определить корневой редуктор:

```ts
import { configureStore } from '@reduxjs/toolkit'
// ...
const store = configureStore({
  reducer: {
    one: oneSlice.reducer,
    two: twoSlice.reducer,
  },
})
export type RootState = ReturnType<typeof store.getState>

export default store
```

Если вы передаете редукторы напрямую `configureStore()` и не определяете корневой редуктор явно, ссылки на него нет `rootReducer`. 
Вместо этого вы можете обратиться к `store.getState`, чтобы получить `State` тип.

```typescript
import { configureStore } from '@reduxjs/toolkit'
import rootReducer from './rootReducer'
const store = configureStore({
  reducer: rootReducer
})
export type RootState = ReturnType<typeof store.getState>
```


### Getting the `Dispatch` type

Если вы хотите получить `Dispatch` тип из своего хранилища, вы можете извлечь его после создания хранилища. Рекомендуется присвоить типу другое имя, `AppDispatch` чтобы избежать путаницы, поскольку имя типа `Dispatch` обычно используется слишком часто. Вы также можете счесть более удобным экспортировать хук, как `useAppDispatch` показано ниже, а затем использовать его везде, куда бы вы ни позвонили `useDispatch`.

```typescript
import { configureStore } from '@reduxjs/toolkit'
import { useDispatch } from 'react-redux'
import rootReducer from './rootReducer'

const store = configureStore({
  reducer: rootReducer,
})

// highlight-start
export type AppDispatch = typeof store.dispatch
export const useAppDispatch: () => AppDispatch = useDispatch // Экспортирует перехват, который можно повторно использовать для разрешения типов
// highlight-end

export default store
```

### Correct typings for the `Dispatch` type (Правильные типизации для Dispatchтипа)

Тип типа `dispatch` функции будет напрямую выведен из `middleware` параметра. Итак, если вы добавляете правильно введенные промежуточные программы, `dispatch` они уже должны быть правильно введены.

Поскольку TypeScript часто расширяет типы массивов при объединении массивов с помощью оператора spread, мы предлагаем использовать методы `.concat(...)` и `.prepend(...)` `MiddlewareArray` возвращаемого by `getDefaultMiddleware()`.

```ts
import { configureStore } from '@reduxjs/toolkit'
import additionalMiddleware from 'additional-middleware'
import logger from 'redux-logger'
// @ts-ignore
import untypedMiddleware from 'untyped-middleware'
import rootReducer from './rootReducer'

export type RootState = ReturnType<typeof rootReducer>
const store = configureStore({
  reducer: rootReducer,
  // highlight-start
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware()
      .prepend(
        // использование промежуточной ПРОГРАММЫ 
        additionalMiddleware,
        // вы также можете ввести промежуточные программы вручную
        untypedMiddleware as Middleware<
          (action: Action<'specialAction'>) => number,
          RootState
        >
      )
      // вызовы prepend и concat могут быть объединены в цепочку
      .concat(logger),
  // highlight-end
})

export type AppDispatch = typeof store.dispatch

export default store
```

#### Using `MiddlewareArray` without `getDefaultMiddleware` (Использование MiddlewareArray без getDefaultMiddleware)

Если вы хотите вообще отказаться от использования `getDefaultMiddleware`, вы все равно можете использовать `MiddlewareArray` для типобезопасной конкатенации вашего `middleware` массива. Этот класс расширяет `Array` тип JavaScript по умолчанию, только с измененными наборами для `.concat(...)` и дополнительным `.prepend(...)` методом.

Однако, как правило, это не требуется, так как вы, вероятно, не столкнетесь с какими-либо проблемами с расширением типа массива, пока вы используете `as const` и не используете оператор spread .

Таким образом, следующие два вызова будут эквивалентны:

```ts
import { configureStore, MiddlewareArray } from '@reduxjs/toolkit'

configureStore({
  reducer: rootReducer,
  middleware: new MiddlewareArray().concat(additionalMiddleware, logger),
})

configureStore({
  reducer: rootReducer,
  middleware: [additionalMiddleware, logger] as const,
})
```

### Using the extracted `Dispatch` type with React Redux (Использование извлеченного Dispatchтипа с помощью React Redux)

По умолчанию `useDispatch` перехват React Redux не содержит никаких типов, учитывающих промежуточные программы. Если вам нужен более конкретный тип для `dispatch` функции при отправке, вы можете указать тип возвращаемой `dispatch` функции или создать пользовательскую типизированную версию `useSelector.`. See [the React Redux documentation](https://react-redux.js.org/using-react-redux/static-typing#typing-the-usedispatch-hook) for details.

## `createAction`

Для большинства случаев использования нет необходимости иметь буквальное определение `action.type`, поэтому можно использовать следующее:

```typescript
createAction<number>('test')
```

Это приведет к тому, что созданное действие будет иметь тип `PayloadActionCreator<number, string>`.

Однако в некоторых настройках вам понадобится литеральный тип для `action.type`, однако. К сожалению, определения типов в TypeScript не допускают сочетания параметров типа, определенных вручную, и выведенных параметров типа, поэтому вам придется указывать `type` как в общем определении, так и в фактическом коде JavaScript:

```typescript
createAction<number, 'test'>('test')
```

Если вы ищете альтернативный способ написания этого без дублирования, вы можете использовать обратный вызов prepare, чтобы оба параметра типа могли быть выведены из аргументов, устраняя необходимость указывать тип действия.

```typescript
function withPayloadType<T>() {
  return (t: T) => ({ payload: t })
}
createAction('test', withPayloadType<string>())
```

### Alternative to using a literally-typed `action.type` (Альтернатива использованию буквально введенного action.type)

Если вы используете `action.type` в качестве дискриминатора дискриминируемое объединение, например, для правильного ввода полезной нагрузки в caseоператорах, вас может заинтересовать эта альтернатива:

У создателей созданных действий есть `match` метод, который действует как a [type predicate](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates):

```typescript
const increment = createAction<number>('increment')
function test(action: Action) {
  if (increment.match(action)) {
    // action.payload inferred correctly here
    action.payload
  }
}
```

Этот `match` метод также очень полезен в сочетании с redux-observablefilterметодом и RxJS.

## `createReducer`

Способ вызова по умолчанию `createReducer` будет с помощью "таблицы поиска" / "объекта карты", например:

```typescript
createReducer(0, {
  increment: (state, action: PayloadAction<number>) => state + action.payload,
})
```

К сожалению, поскольку ключи представляют собой только строки, использование этого API TypeScript не может ни определять, ни проверять типы действий для вас:

```typescript
{
  const increment = createAction<number, 'increment'>('increment')
  const decrement = createAction<number, 'decrement'>('decrement')
  createReducer(0, {
    [increment.type]: (state, action) => {
      // action is any here
    },
    [decrement.type]: (state, action: PayloadAction<string>) => {
      // even though action should actually be PayloadAction<number>, TypeScript can't detect that and won't give a warning here.
    },
  })
}
```

В качестве альтернативы RTK включает типобезопасный API-интерфейс reducer builder.

### Building Type-Safe Reducer Argument Objects

Вместо того чтобы использовать простой объект в качестве аргумента createReducer, вы также можете использовать обратный вызов , который получает `ActionReducerMapBuilder` экземпляр:

```typescript {3-10}
const increment = createAction<number, 'increment'>('increment')
const decrement = createAction<number, 'decrement'>('decrement')
createReducer(0, (builder) =>
  builder
    .addCase(increment, (state, action) => {
      // action is inferred correctly here
    })
    .addCase(decrement, (state, action: PayloadAction<string>) => {
      // this would error out
    })
)
```

Мы рекомендуем использовать этот API, если при определении объектов аргументов редуктора требуется более строгая безопасность типов.

#### Typing `builder.addMatcher`

В качестве первого matcherаргумента `builder.addMatcher` следует использовать функцию, a [type predicate](https://www.typescriptlang.org/docs/handbook/advanced-types.html#using-type-predicates) function should be used.
В результате `action` аргумент для второго reducerаргумента может быть выведен с помощью TypeScript:

```ts
function isNumberValueAction(action: AnyAction): action is PayloadAction<{ value: number }> {
  return typeof action.payload.value === 'number'
}

createReducer({ value: 0 }, builder =>
   builder.addMatcher(isNumberValueAction, (state, action) => {
      state.value += action.payload.value
   })
})
```

## `createSlice`

As `createSlice` создает ваши действия, а также ваш редуктор для вас, поэтому вам не нужно беспокоиться о безопасности типов здесь. Типы действий могут быть просто предоставлены встроенными:

```typescript
const slice = createSlice({
  name: 'test',
  initialState: 0,
  reducers: {
    increment: (state, action: PayloadAction<number>) => state + action.payload,
  },
})
// now available:
slice.actions.increment(2)
// also available:
slice.caseReducers.increment(0, { type: 'increment', payload: 5 })
```

Если у вас слишком много редукторов регистра и их встроенное определение было бы запутанным, или вы хотите повторно использовать редукторы регистра в срезах, вы также можете определить их вне `createSlice` вызова и ввести их как `CaseReducer`:

```typescript
type State = number
const increment: CaseReducer<State, PayloadAction<number>> = (state, action) =>
  state + action.payload

createSlice({
  name: 'test',
  initialState: 0,
  reducers: {
    increment,
  },
})
```

### Defining the Initial State Type (Определение типа начального состояния)

Возможно, вы заметили, что не стоит передавать ваш `SliceState` тип как общий `createSlice`. Это связано с тем, что почти во всех случаях `createSlice` необходимо определить дополнительные общие параметры, а TypeScript не может смешивать явное объявление и вывод общих типов в одном и том же "общем блоке".

Стандартный подход заключается в объявлении интерфейса или типа для вашего состояния, создании начального значения состояния, которое использует этот тип, и передаче начального значения состояния `createSlice`. Вы также можете использовать конструкцию `initialState: myInitialState as SliceState`.

```ts {1,4,8,15}
type SliceState = { state: 'loading' } | { state: 'finished'; data: string }

// First approach: define the initial state using that type
const initialState: SliceState = { state: 'loading' }

createSlice({
  name: 'test1',
  initialState, // type SliceState is inferred for the state of the slice
  reducers: {},
})

// Or, cast the initial state as necessary
createSlice({
  name: 'test2',
  initialState: { state: 'loading' } as SliceState,
  reducers: {},
})
```

что приведет к a `Slice<SliceState, ...>`.

### Defining Action Contents with `prepare` Callbacks (Определение содержимого действия с prepareпомощью обратных вызовов)

Если вы хотите добавить `metaerror` свойство или к своему действию или настроить `payload` его, вы должны использовать `prepare` обозначение.

Использование этой нотации с TypeScript выглядит следующим образом:

```ts {5-16}
const blogSlice = createSlice({
  name: 'blogData',
  initialState,
  reducers: {
    receivedAll: {
      reducer(
        state,
        action: PayloadAction<Page[], string, { currentPage: number }>
      ) {
        state.all = action.payload
        state.meta = action.meta
      },
      prepare(payload: Page[], currentPage: number) {
        return { payload, meta: { currentPage } }
      },
    },
  },
})
```

### Generated Action Types for Slices (Сгенерированные типы действий для срезов)

Поскольку TS не может объединить два строковых литерала (`slice.name` и ключ `actionMap` ) в новый литерал, все созданные `createSliceActionCreators` имеют тип 'string'. Обычно это не проблема, поскольку эти типы редко используются в качестве литералов.

В большинстве случаев, который `type` потребуется в качестве литерала, `slice.action.myAction.match` предикат типа должен быть жизнеспособной альтернативой:

```ts {10}
const slice = createSlice({
  name: 'test',
  initialState: 0,
  reducers: {
    increment: (state, action: PayloadAction<number>) => state + action.payload,
  },
})

function myCustomMiddleware(action: Action) {
  if (slice.actions.increment.match(action)) {
    // `action` is narrowed down to the type `PayloadAction<number>` here.
  }
}
```

Если вам действительно нужен этот тип, к сожалению, нет другого способа, кроме ручного приведения.

### Type safety with `extraReducers` (Безопасность типа с extraReducers)

Таблицы подстановки редуктора, которые сопоставляют строку действия `type` с функцией редуктора, нелегко полностью правильно ввести. Это влияет как `createReducer` на, так и на `extraReducers` аргумент для `createSlice`. Итак, как и в случае с `createReducer`, [you may also use the "builder callback" approach](#building-type-safe-reducer-argument-objects) для определения аргумента объекта `reducer` .

Это особенно полезно, когда редуктору среза необходимо обрабатывать типы действий, сгенерированные другими срезами или сгенерированные конкретными вызовами `createAction` (например, действиями, сгенерированными [`createAsyncThunk`](../api/createAsyncThunk.mdx)).

```ts {27-30}
const fetchUserById = createAsyncThunk(
  'users/fetchById',
  // if you type your function argument here
  async (userId: number) => {
    const response = await fetch(`https://reqres.in/api/users/${userId}`)
    return (await response.json()) as Returned
  }
)

interface UsersState {
  entities: []
  loading: 'idle' | 'pending' | 'succeeded' | 'failed'
}

const initialState = {
  entities: [],
  loading: 'idle',
} as UsersState

const usersSlice = createSlice({
  name: 'users',
  initialState,
  reducers: {
    // fill in primary logic here
  },
  extraReducers: (builder) => {
    builder.addCase(fetchUserById.pending, (state, action) => {
      // both `state` and `action` are now correctly typed
      // based on the slice state and the `pending` action creator
    })
  },
})
```

Как и конструктор в `createReducer`, этот конструктор также принимает `addMatcher` (see [typing `builder.matcher`](#typing-builderaddmatcher)) и `addDefaultCase`.

### Wrapping `createSlice` (Обертывание `createSlice`)

Если вам нужно повторно использовать логику редуктора, обычно пишется ["higher-order reducers"](https://redux.js.org/recipes/structuring-reducers/reusing-reducer-logic#customizing-behavior-with-higher-order-reducers) которые оборачивают функцию-редуктор дополнительным общим поведением. Это также можно сделать с помощью `createSlice`, но из-за сложности типов для `createSlice` вам придется использовать типы `SliceCaseReducers` и` ValidateSliceCaseReducers` очень специфическим образом.

Вот пример такого "универсального" обернутого вызова createSlice:

```ts
interface GenericState<T> {
  data?: T
  status: 'loading' | 'finished' | 'error'
}

const createGenericSlice = <
  T,
  Reducers extends SliceCaseReducers<GenericState<T>>
>({
  name = '',
  initialState,
  reducers,
}: {
  name: string
  initialState: GenericState<T>
  reducers: ValidateSliceCaseReducers<GenericState<T>, Reducers>
}) => {
  return createSlice({
    name,
    initialState,
    reducers: {
      start(state) {
        state.status = 'loading'
      },
      /**
       * If you want to write to values of the state that depend on the generic
       * (in this case: `state.data`, which is T), you might need to specify the
       * State type manually here, as it defaults to `Draft<GenericState<T>>`,
       * which can sometimes be problematic with yet-unresolved generics.
       * This is a general problem when working with immer's Draft type and generics.
       */
      success(state: GenericState<T>, action: PayloadAction<T>) {
        state.data = action.payload
        state.status = 'finished'
      },
      ...reducers,
    },
  })
}

const wrappedSlice = createGenericSlice({
  name: 'test',
  initialState: { status: 'loading' } as GenericState<string>,
  reducers: {
    magic(state) {
      state.status = 'finished'
      state.data = 'hocus pocus'
    },
  },
})
```

## `createAsyncThunk`

In the most common use cases, you should not need to explicitly declare any types for the `createAsyncThunk` call itself.

Just provide a type for the first argument to the `payloadCreator` argument as you would for any function argument, and the resulting thunk will accept the same type as its input parameter.
The return type of the `payloadCreator` will also be reflected in all generated action types.

```ts
interface MyData {
  // ...
}

const fetchUserById = createAsyncThunk(
  'users/fetchById',
  // highlight-start
  // Declare the type your function argument here:
  async (userId: number) => {
    // highlight-end
    const response = await fetch(`https://reqres.in/api/users/${userId}`)
    // Inferred return type: Promise<MyData>
    // highlight-next-line
    return (await response.json()) as MyData
  }
)

// the parameter of `fetchUserById` is automatically inferred to `number` here
// and dispatching the resulting thunkAction will return a Promise of a correctly
// typed "fulfilled" or "rejected" action.
const lastReturnedAction = await store.dispatch(fetchUserById(3))
```

The second argument to the `payloadCreator`, known as `thunkApi`, is an object containing references to the `dispatch`, `getState`, and `extra` arguments from the thunk middleware as well as a utility function called `rejectWithValue`. If you want to use these from within the `payloadCreator`, you will need to define some generic arguments, as the types for these arguments cannot be inferred. Also, as TS cannot mix explicit and inferred generic parameters, from this point on you'll have to define the `Returned` and `ThunkArg` generic parameter as well.

To define the types for these arguments, pass an object as the third generic argument, with type declarations for some or all of these fields:

```ts
type AsyncThunkConfig = {
  /** return type for `thunkApi.getState` */
  state?: unknown
  /** type for `thunkApi.dispatch` */
  dispatch?: Dispatch
  /** type of the `extra` argument for the thunk middleware, which will be passed in as `thunkApi.extra` */
  extra?: unknown
  /** type to be passed into `rejectWithValue`'s first argument that will end up on `rejectedAction.payload` */
  rejectValue?: unknown
  /** return type of the `serializeError` option callback */
  serializedErrorType?: unknown
  /** type to be returned from the `getPendingMeta` option callback & merged into `pendingAction.meta` */
  pendingMeta?: unknown
  /** type to be passed into the second argument of `fulfillWithValue` to finally be merged into `fulfilledAction.meta` */
  fulfilledMeta?: unknown
  /** type to be passed into the second argument of `rejectWithValue` to finally be merged into `rejectedAction.meta` */
  rejectedMeta?: unknown
}
```

```ts
const fetchUserById = createAsyncThunk<
  // highlight-start
  // Return type of the payload creator
  MyData,
  // First argument to the payload creator
  number,
  {
    // Optional fields for defining thunkApi field types
    dispatch: AppDispatch
    state: State
    extra: {
      jwt: string
    }
  }
  // highlight-end
>('users/fetchById', async (userId, thunkApi) => {
  const response = await fetch(`https://reqres.in/api/users/${userId}`, {
    headers: {
      Authorization: `Bearer ${thunkApi.extra.jwt}`,
    },
  })
  return (await response.json()) as MyData
})
```

If you are performing a request that you know will typically either be a success or have an expected error format, you can pass in a type to `rejectValue` and `return rejectWithValue(knownPayload)` in the action creator. This allows you to reference the error payload in the reducer as well as in a component after dispatching the `createAsyncThunk` action.

```ts
interface MyKnownError {
  errorMessage: string
  // ...
}
interface UserAttributes {
  id: string
  first_name: string
  last_name: string
  email: string
}

const updateUser = createAsyncThunk<
  // Return type of the payload creator
  MyData,
  // First argument to the payload creator
  UserAttributes,
  // Types for ThunkAPI
  {
    extra: {
      jwt: string
    }
    rejectValue: MyKnownError
  }
>('users/update', async (user, thunkApi) => {
  const { id, ...userData } = user
  const response = await fetch(`https://reqres.in/api/users/${id}`, {
    method: 'PUT',
    headers: {
      Authorization: `Bearer ${thunkApi.extra.jwt}`,
    },
    body: JSON.stringify(userData),
  })
  if (response.status === 400) {
    // Return the known error for future handling
    return thunkApi.rejectWithValue((await response.json()) as MyKnownError)
  }
  return (await response.json()) as MyData
})
```

While this notation for `state`, `dispatch`, `extra` and `rejectValue` might seem uncommon at first, it allows you to provide only the types for these you actually need - so for example, if you are not accessing `getState` within your `payloadCreator`, there is no need to provide a type for `state`. The same can be said about `rejectValue` - if you don't need to access any potential error payload, you can ignore it.

In addition, you can leverage checks against `action.payload` and `match` as provided by `createAction` as a type-guard for when you want to access known properties on defined types. Example:

- In a reducer

```ts
const usersSlice = createSlice({
  name: 'users',
  initialState: {
    entities: {},
    error: null,
  },
  reducers: {},
  extraReducers: (builder) => {
    builder.addCase(updateUser.fulfilled, (state, { payload }) => {
      state.entities[payload.id] = payload
    })
    builder.addCase(updateUser.rejected, (state, action) => {
      if (action.payload) {
        // Since we passed in `MyKnownError` to `rejectValue` in `updateUser`, the type information will be available here.
        state.error = action.payload.errorMessage
      } else {
        state.error = action.error
      }
    })
  },
})
```

- In a component

```ts
const handleUpdateUser = async (userData) => {
  const resultAction = await dispatch(updateUser(userData))
  if (updateUser.fulfilled.match(resultAction)) {
    const user = resultAction.payload
    showToast('success', `Updated ${user.name}`)
  } else {
    if (resultAction.payload) {
      // Since we passed in `MyKnownError` to `rejectValue` in `updateUser`, the type information will be available here.
      // Note: this would also be a good place to do any handling that relies on the `rejectedWithValue` payload, such as setting field errors
      showToast('error', `Update failed: ${resultAction.payload.errorMessage}`)
    } else {
      showToast('error', `Update failed: ${resultAction.error.message}`)
    }
  }
}
```

## `createEntityAdapter`

Typing `createEntityAdapter` only requires you to specify the entity type as the single generic argument.

The example from the `createEntityAdapter` documentation would look like this in TypeScript:

```ts
interface Book {
  bookId: number
  title: string
  // ...
}

// highlight-next-line
const booksAdapter = createEntityAdapter<Book>({
  selectId: (book) => book.bookId,
  sortComparer: (a, b) => a.title.localeCompare(b.title),
})

const booksSlice = createSlice({
  name: 'books',
  initialState: booksAdapter.getInitialState(),
  reducers: {
    bookAdded: booksAdapter.addOne,
    booksReceived(state, action: PayloadAction<{ books: Book[] }>) {
      booksAdapter.setAll(state, action.payload.books)
    },
  },
})
```

### Using `createEntityAdapter` with `normalizr`

When using a library like [`normalizr`](https://github.com/paularmstrong/normalizr/), your normalized data will resemble this shape:

```js
{
  result: 1,
  entities: {
    1: { id: 1, other: 'property' },
    2: { id: 2, other: 'property' }
  }
}
```

The methods `addMany`, `upsertMany`, and `setAll` all allow you to pass in the `entities` portion of this directly with no extra conversion steps. However, the `normalizr` TS typings currently do not correctly reflect that multiple data types may be included in the results, so you will need to specify that type structure yourself.

Here is an example of how that would look:

```ts
type Author = { id: number; name: string }
type Article = { id: number; title: string }
type Comment = { id: number; commenter: number }

export const fetchArticle = createAsyncThunk(
  'articles/fetchArticle',
  async (id: number) => {
    const data = await fakeAPI.articles.show(id)
    // Normalize the data so reducers can responded to a predictable payload.
    // Note: at the time of writing, normalizr does not automatically infer the result,
    // so we explicitly declare the shape of the returned normalized data as a generic arg.
    const normalized = normalize<
      any,
      {
        articles: { [key: string]: Article }
        users: { [key: string]: Author }
        comments: { [key: string]: Comment }
      }
    >(data, articleEntity)
    return normalized.entities
  }
)

export const slice = createSlice({
  name: 'articles',
  initialState: articlesAdapter.getInitialState(),
  reducers: {},
  extraReducers: (builder) => {
    builder.addCase(fetchArticle.fulfilled, (state, action) => {
      // The type signature on action.payload matches what we passed into the generic for `normalize`, allowing us to access specific properties on `payload.articles` if desired
      articlesAdapter.upsertMany(state, action.payload.articles)
    })
  },
})
```
