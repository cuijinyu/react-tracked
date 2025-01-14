---
id: recipes
title: Recipes
sidebar_label: Recipes
---

React Tracked provides a primitive API,
and there are various ways to use it for apps.

## Recipes for createContainer

The argument `useValue` in `createContainer` is so flexible
and there are various usages.

### useReducer (props)

This is the most typical usage.
You define a generic reducer and pass `reducer` and `initialState` as props.

```javascript
const {
  Provider,
  useTracked,
  // ...
} = createContainer(({ reducer, initialState, init }) => useReducer(reducer, initialState, init));

const reducer = ...;

const App = ({ initialState }) => (
  <Provider reducer={reducer} initialState={initialState}>
    ...
  </Provider>
);
```

### useReducer (embedded)

For most cases, you would have a static reducer.
In this case, define useValue with the reducer in advance.
The `initialState` can be defined in useValue like the following example,
or can be taken from props: `({ initialState }) => useReducer(...)`

This is good for TypeScript because the hooks returned by `createContainer` is already typed.

```javascript
const reducer = ...;
const initialState = ...;

const {
  Provider,
  useTracked,
  // ...
} = createContainer(() => useReducer(reducer, initialState));


const App = () => (
  <Provider>
    ...
  </Provider>
);
```

### useState (props)

If you don't need reducer, useState would be simpler.

```javascript
const {
  Provider,
  useTracked,
  // ...
} = createContainer(({ initialState }) => useState(initialState);


const App = ({ initialState }) => (
  <Provider initialState={initialState}>
    ...
  </Provider>
);
```

### useState (empty object)

You could even start with completely an empty object.

This might not be TypeScript friendly. Although, you could do this: `useState<State>({})`

```javascript
const {
  Provider,
  useTracked,
  // ...
} = createContainer(() => useState({});

const App = () => (
  <Provider>
    ...
  </Provider>
);
```

### useState (custom actions)

You can also use a custom hook.
The `update` can be anything, so for example it can be a set of action functions.

```javascript
const useValue = () => {
  const [state, setState] = useState({ count1: 0, count2: 0 });
  const increment1 = useCallback(() => {
    setState(s => ({ ...s, count1: s.count1 + 1 }));
  }, [setState]);
  const increment2 = useCallback(() => {
    setState(s => ({ ...s, count2: s.count2 + 2 }));
  }, [setState]);
  const actions = useMemo(() => (
    { increment1, increment2 },
  ), [increment1, increment2]);
  return [state, actions];
};

const {
  Provider,
  useTracked,
  // ...
} = createContainer(useValue);

const App = () => (
  <Provider>
    ...
  </Provider>
);
```

### useReducer (with persistence)

Here's how to persist state in localStorage.

```javascript
const reducer = ...;
const initialState = ...; // used only if localStorage is empty.
const storageKey = 'persistedState';

const init = () => {
  let preloadedState;
  try {
    preloadedState =  JSON.parse(window.localStorage.getItem(storageKey));
    // validate preloadedState if necessary
  } catch (e) {
    // ignore
  }
  return preloadedState || initialState;
};

const useValue = () => {
  const [state, dispatch] = useReducer(reducer, null, init);
  useEffect(() => {
    window.localStorage.setItem(storageKey, JSON.stringify(state));
  }, [state]);
  return [state, dispatch];
};

const {
  Provider,
  useTracked,
  // ...
} = createContainer(useValue);

const App = () => (
  <Provider>
    ...
  </Provider>
);
```

Using async storage is a bit tricky.
See [the thread](https://github.com/dai-shi/react-tracked/issues/8#issuecomment-548095476) for an example.

### useState (with propState)

If you already have a state and would like to use Provider with it,
you can sync a container state with a state from props.

```javascript
const useValue = ({ propState }) => {
  const [state, setState] = useState(propState);
  useEffect(() => { // or useLayoutEffect
    setState(propState);
  }, [propState]);
  return [state, setState];
};

const {
  Provider,
  useTracked,
  // ...
} = createContainer(useValue);

const App = ({ propState }) => (
  <Provider propState={propState}>
    ...
  </Provider>
);
```

Note that `propState` has to be updated immutably.

## Recipes for useTrackedState and useTracked

The `useTrackedState` and `useTracked` hooks are useful as is,
but new hooks can also be created based on them.

### useTrackedSelector

Selector interface is useful to share selection logic.
You can create a selector hook with state usage tracking very easily.

```javascript
const useTrackedSelector = selector => selector(useTrackedState());
```

Note: This is different from `useSelector` which has no tracking support
and triggers re-render based on the ref equality of selected value.

### useTrackedByName (based on useState)

Sometimes, you might want to select a state by its property name.
Here's a custom hook to return a tuple `[value, setValue]` selected by a name.

```javascript
const useTrackedByName = (name) => {
  const [state, setState] = useTracked();
  const update = useCallback((newVal) => {
    setState(oldVal => ({
      ...oldVal,
      [name]: typeof newVal === 'function' ? newVal(oldVal[name]) : newVal,
    }));
  }, [setState, name]);
  return [state[name], update];
};
```

### useTrackedWithImmer (based on useState)

Updating a property deep in a state object is troublesome.
Here's a custom hook to use [immer](https://github.com/immerjs/immer) for setState.

```javascript
import produce from 'immer';

const useTrackedWithImmer = () => {
  const [state, setState] = useTracked();
  const update = useCallback((updater) => {
    setState(oldVal => produce(oldVal, updater));
  }, [setState]);
  return [state, update];
};
```

Note: This can also be done at `createContainer`.

## Recipes for useUpdate (useDispatch)

The `useUpdate` simply returns the second item
in a tuple returned by `useState` or `useReducer`.
It can also be extended as a custom hook.

### useSafeDispatch

This is a modified version of useDispatch that calls `getUntrackedObject`
recursively on an action object before dispatching it.

```javascript
import { getUntrackedObject } from 'react-tracked';

const untrackDeep = (obj) => {
  if (typeof obj !== 'object' || obj === null) return obj;
  const untrackedObj = getUntrackedObject(obj);
  if (untrackedObj !== null) return untrackedObj;
  const newObj = {};
  let modified = false;
  Object.entries(obj).forEach(([k, v]) => {
    newObj[k] = untrackDeep(v);
    if (newObj[k] !== null) {
      modified = true;
    } else {
      newObj[k] = v;
    }
  });
  return modified ? newObj : obj;
};

const useSafeDispatch = () => {
  const dispatch = useDispatch();
  return useCallback((action) => {
    dispatch(untrackDeep(action));
  }, [dispatch]);
};
```
