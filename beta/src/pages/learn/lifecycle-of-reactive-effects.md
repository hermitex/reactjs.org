---
title: 'Lifecycle of Reactive Effects'
---

<Intro>

Effects have a different lifecycle from components. Components may mount, update, or unmount. An Effect can only do two things: to start synchronizing something, and later to stop synchronizing it. This cycle can happen multiple times if your Effect depends on props and state that change over time. React provides a linter rule to check that you've specified your Effect's dependencies correctly. This keeps your Effect synchronized to the latest props and state.

</Intro>

<YouWillLearn>

- How an Effect's lifecycle is different from a component's lifecycle
- How to think about each individual Effect in isolation
- When your Effect needs to re-synchronize, and why
- How your Effect's dependencies are determined
- What it means for a value to be reactive
- How React verifies your dependencies are correct with a linter
- What to do when you disagree with the linter
- What an empty dependency array means

</YouWillLearn>

## The lifecycle of an Effect {/*the-lifecycle-of-an-effect*/}

Every React component goes through the same lifecycle:

- A component _mounts_ when it's added to the screen.
- A component _updates_ when it receives new props or state. This usually happens in response to an interaction.
- A component _unmounts_ when it's removed from the screen.

**It's a good way to think about components, but _not_ about Effects.** Instead, try to think about each Effect independently from your component's lifecycle. An Effect describes how to [synchronize an external system](/learn/synchronizing-with-effects) to the current props and state. As your code changes, this synchronization will need to happen more or less often.

To illustrate this point, consider this Effect connecting your component to a chat server:

```js
const serverUrl = 'https://mychat.com';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]);
  // ...
}
```

Your Effect's body specifies how to **start synchronizing:**

```js {2-3}
    // ...
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
    // ...
```

The cleanup function returned by your Effect specifies how to **stop synchronizing:**

```js {5}
    // ...
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
    // ...
```

Intuitively, you might think that React would **start synchronizing** when your component mounts and **stop synchronizing** when your component unmounts. However, this is not the end of the story! Sometimes, it may also be necessary to **start and stop synchronizing multiple times** while the component remains mounted.

Let's look at _why_ this is necessary, _when_ it happens, and _how_ you can control this behavior.

<Note>

Some Effects don't return a cleanup function at all. [More often than not,](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development) you'll want to return one--but if you don't, React will behave as if you returned an empty cleanup function that doesn't do anything.

</Note>

### Why synchronization may need to happen more than once {/*why-synchronization-may-need-to-happen-more-than-once*/}

Imagine this `ChatRoom` component receives a `roomId` prop that the user picks in a dropdown. Let's say that initially the user picks the `"general"` room as the `roomId`. Your app displays the `"general"` chat room:

```js {3}
const serverUrl = 'https://mychat.com';

function ChatRoom({ roomId /* "general" */ }) {
  // ...
  return <h1>Welcome to the {roomId} room!</h1>;
}
```

After the UI is displayed, React will run your Effect to **start synchronizing**. It connects to the `"general"` room:

```js {3,4}
function ChatRoom({ roomId /* "general" */ }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Connects to the "general" room
    connection.connect();
    return () => {
      connection.disconnect(); // Disconnects from the "general" room
    };
  }, [roomId]);
  // ...
```

So far, so good.

Later, the user picks a different room in the dropdown (for example, `"travel"`). First, React will update the UI:

```js {1}
function ChatRoom({ roomId /* "travel" */ }) {
  // ...
  return <h1>Welcome to the {roomId} room!</h1>;
}
```

Pause to think about what should happen next. The user sees that `"travel"` is the selected chat room in the UI. However, the Effect that ran the last time is still connected to the `"general"` room. **The `roomId` prop has changed, so whatever your Effect did back then (connecting to the `"general"` room) no longer matches the UI.**

At this point, you want React to do two things:

1. Stop synchronizing with the old `roomId` (disconnect from the `"general"` room)
2. Start synchronizing with the new `roomId` (connect to the `"travel"` room)

**Luckily, you've already taught React how to do both of these things!** Your Effect's body specifies how to start synchronizing, and your cleanup function specifies how to stop synchronizing. All that React needs to do now is to call them in the correct order and with the correct props and state. Let's see how exactly that happens.

### How React re-synchronizes your Effect {/*how-react-re-synchronizes-your-effect*/}

Recall that your `ChatRoom` component has received a new value for its `roomId` prop. It used to be `"general"`, and now it is `"travel"`. React needs to re-synchronize your Effect to re-connect you to a different room.

To **stop synchronizing,** React will call the cleanup function that your Effect returned after connecting to the `"general"` room. Since `roomId` was `"general"`, the cleanup function disconnects from the `"general"` room:

```js {6}
function ChatRoom({ roomId /* "general" */ }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Connects to the "general" room
    connection.connect();
    return () => {
      connection.disconnect(); // Disconnects from the "general" room
    };
    // ...
```

Then React will run the Effect that you've provided during this render. This time, `roomId` is `"travel"` so it will **start synchronizing** to the `"travel"` chat room (until its cleanup function is eventually called too):

```js {3,4}
function ChatRoom({ roomId /* "travel" */ }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Connects to the "travel" room
    connection.connect();
    // ...
```

Thanks to this, you're now connected to the same room that the user chose in the UI. Disaster averted!

Every time after your component re-renders with a different `roomId`, your Effect will re-synchronize. For example, let's say the user changes `roomId` from `"travel"` to `"music"`. React will again **stop synchronizing** your Effect by calling its cleanup function (disconnecting you from the `"travel"` room). Then it will **start synchronizing** again by running its body with the new `roomId` prop (connecting you to the `"music` room).

Finally, when the user goes to a different screen, `ChatRoom` unmounts. Now there is no need to stay connected at all. React will **stop synchronizing** your Effect one last time and disconnect you from the `"music"` chat room.

### Thinking from the Effect's perspective {/*thinking-from-the-effects-perspective*/}

Let's recap everything that's happened from the `ChatRoom` component's perspective:

1. `ChatRoom` mounted with `roomId` set to `"general"`
1. `ChatRoom` updated with `roomId` set to `"travel"`
1. `ChatRoom` mounted with `roomId` set to `"music"`
1. `ChatRoom` unmounted

During each of these points in the component's lifecycle, your Effect did different things:

1. Your Effect connected to the `"general"` room
1. Your Effect disconnected from the `"general"` room and connected to the `"travel"` room
1. Your Effect disconnected from the `"travel"` room and connected to the `"music"` room
1. Your Effect disconnected from the `"music"` room

Now let's think about what happened from the perspective of the Effect itself:

```js
  useEffect(() => {
    // Your Effect connected to the room specified with roomId...
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      // ...until it disconnected
      connection.disconnect();
    };
  }, [roomId]);
```

This code's structure might inspire you to see what happened as a sequence of non-overlapping time periods:

1. Your Effect connected to the `"general"` room (until it disconnected)
1. Your Effect connected to the `"travel"` room (until it disconnected)
1. Your Effect connected to the `"music"` room (until it disconnected)

In other words, there is one "connect and later disconnect" cycle for each render with a distinct value of `roomId`. Compare this way of thinking with the earlier list from the component's point of view. Which one do you prefer?

When you write the rendering logic that returns the JSX, you don't think about whether a component is mounting or updating. You only describe what you want to see on the screen, and React [figures out the rest.](/learn/reacting-to-input-with-state)

**Similarly, when you write Effects, you don't think about whether a component is mounting, updating, or unmounting. You only describe how to start synchronization and how to stop it, and React figures out the rest.**

### How React knows that it needs to re-synchronize the Effect {/*how-react-knows-that-it-needs-to-re-synchronize-the-effect*/}

You might be wondering how React knew that your Effect needed to re-synchronize after `roomId` changes. It's because *you told React* that this Effect's code depends on `roomId` by including it in the [list of dependencies:](/learn/synchronizing-with-effects#step-2-specify-the-effect-dependencies)

```js {1,3,8}
function ChatRoom({ roomId }) { // The roomId prop may change over time
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // This Effect reads roomId 
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]); // So you tell React that this Effect "depends on" roomId
  // ...
```

Here's how this works:

1. You knew `roomId` is a prop, which means it can change over time.
2. You knew that your Effect reads `roomId` (so its logic depends on a value that may change later).
3. This is why you specified it as your Effect's dependency (so that it re-synchronizes when `roomId` changes).

Every time after your component re-renders, React will look at the array of dependencies that you have passed. If any of the values in the array is different from the value at the same spot that you passed during the previous render, React will re-synchronize your Effect. For example, if you passed `["general"]` during the initial render, and later you passed `["travel"]` during the next render, React will compare `"general"` and `"travel"`. These are different values (compared with [`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is)), so React will re-synchronize your Effect. On the other hand, if your component re-renders but `roomId` has not changed, your Effect will remain connected to the same room.

### Each Effect represents a separate synchronization process {/*each-effect-represents-a-separate-synchronization-process*/}

Resist adding unrelated logic to your Effect only because this logic needs to run at the same time as an Effect you already wrote. For example, let's say you want to send an analytics event when the user visits the room. You already have an Effect that depends on `roomId`, so you might feel tempted to add the analytics call right there:

```js {3}
function ChatRoom({ roomId }) {
  useEffect(() => {
    logVisit(roomId);
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]);
  // ...
}
```

But imagine you later add another dependency to this Effect that needs to re-establish the connection. If this Effect re-synchronizes, it will also call `logVisit(roomId)` for the same room, which you did not intend. Logging the visit **is a separate process** from connecting. This is why they should be written as two separate Effects:

```js {2-4}
function ChatRoom({ roomId }) {
  useEffect(() => {
    logVisit(roomId);
  }, [roomId]);

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    // ...
  }, [roomId]);
  // ...
}
```

In the above example, deleting one Effect wouldn’t break the other Effect's logic. This is a good indication that they synchronize different things, and so it made sense to split them up. On the other hand, if you split up a cohesive piece of logic into separate Effects, the code may look "cleaner" but will be [more difficult to maintain.](/learn/you-might-not-need-an-effect#chains-of-computations) This is why you should think whether the processes are same or separate, not whether the code looks cleaner.

## Effects "react" to reactive values {/*effects-react-to-reactive-values*/}

Your Effect reads two variables (`serverUrl` and `roomId`), but you only specified `roomId` as a dependency:

```js {5,10}
const serverUrl = 'https://mychat.com';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]);
  // ...
}
```

Why doesn't `serverUrl` need to be a dependency?

This is because the `serverUrl` never changes due to a re-render. It's always the same no matter how many times and with which props and state the component re-renders. Since `serverUrl` never changes, it wouldn't make sense to specify it as a dependency. After all, dependencies only do something when they change over time!

On the other hand, `roomId` may be different on a re-render. **Props, state, and other values declared inside the component are _reactive_ because they're calculated during rendering and participate in the React data flow.**

If `serverUrl` was a state variable, it would be reactive. Reactive values must be included in dependencies:

```js {2,4,9}
function ChatRoom({ roomId }) { // Props change over time
  const [serverUrl, setServerUrl] = useState('https://mychat.com'); // State may change over time
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Your Effect reads props and state
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId, serverUrl]); // So you tell React that this Effect "depends on" on props and state
  // ...
}
```

This ensures that your Effect re-synchronizes whenever the reactive value changes.

### All variables declared in the component body are reactive {/*all-variables-declared-in-the-component-body-are-reactive*/}

Props and state aren't the only reactive values. Values that you calculate from them are also reactive. If the props or state change, your component will re-render, and the values calculated from them will also change. This is why all variables from the component body used by the Effect should also be in the Effect dependency list.

Let's say that the user can pick a chat server in the dropdown, but they can also configure a default server in settings. Suppose you've already put the settings state in a [context](/learn/scaling-up-with-reducer-and-context) so you read the `settings` from that context. Now you calculate the `serverUrl` based on the selected server from props and the default server from context:

```js {3,5,10}
function ChatRoom({ roomId, selectedServerUrl }) { // roomId is reactive
  const settings = useContext(SettingsContext); // settings is reactive
  const serverUrl = selectedServerUrl ?? settings.defaultServerUrl; // serverUrl is reactive
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId); // Your Effect reads roomId and serverUrl
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId, serverUrl]); // So it needs to re-synchronize when either of them changes!
  // ...
}
```

In this example, `serverUrl` is not a prop or a state variable. It's a regular variable that you calculate during rendering. But it's calculated during rendering, so it can change due to a re-render. This is why it's reactive.

**All values inside the component (including props, state, and variables in your component's body) are reactive. Any reactive value can change on a re-render, so you need to include reactive values as Effect's dependencies.**

In other words, Effects "react" to all values from the component body.

<DeepDive title="Can global or mutable values be dependencies?">

Mutable values aren't reactive.

**A mutable value like [`location.pathname`](https://developer.mozilla.org/en-US/docs/Web/API/Location/pathname) can't be a dependency.** It's mutable, so it can change at any time completely outside of the React rendering data flow. Changing it wouldn't trigger a re-render of your component. Therefore, even if you specified it in the dependencies, React *wouldn't know* to re-synchronize the Effect when it changes. This also breaks the rules of React because reading mutable data during rendering (which is when you calculate the dependencies) breaks [purity of rendering.](/learn/keeping-components-pure) Instead, you should read and subscribe to an external mutable value with [`useSyncExternalStore`](/learn/you-might-not-need-an-effect#subscribing-to-an-external-store).

**A mutable value like [`ref.current`](/apis/useref#reference) or things you read from it also can't be a dependency.** The ref object returned by `useRef` itself can be a dependency, but its `current` property is intentionally mutable. It lets you [keep track of something without triggering a re-render.](/learn/referencing-values-with-refs) But since changing it doesn't trigger a re-render, it's not a reactive value, and React won't know to re-run your Effect when it changes.

As you'll learn below on this page, a linter will check for these issues automatically.

</DeepDive>

### React verifies that you specified every reactive value as a dependency {/*react-verifies-that-you-specified-every-reactive-value-as-a-dependency*/}

If your linter is [configured for React,](/learn/editor-setup#linting) it will check that every reactive value used by your Effect's code is declared as its dependency. For example, this is a lint error because both `roomId` and `serverUrl` are reactive:

```js {9}
function ChatRoom({ roomId }) { // roomId is reactive
  const [serverUrl, setServerUrl] = useState('https://mychat.com'); // serverUrl is reactive
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // 🔴 React Hook useEffect has missing dependencies: 'roomId', 'serverUrl'
  // ...
}
```

This may look like a React error, but really React is pointing out a bug in your code. Both `roomId` and `serverUrl` may change over time, but you're forgetting to re-synchronize your Effect when they change. As a result, you will remain connected to the initial `roomId` and `serverUrl` even after the user picks different values in the UI.

To fix the bug, follow the linter's suggestion to specify `roomId` and `serverUrl` as dependencies of your Effect:

```js {9}
function ChatRoom({ roomId, serverUrl }) {
  const [serverUrl, setServerUrl] = useState('https://mychat.com'); // serverUrl is reactive
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [serverUrl, roomId]); // ✅ All dependencies declared
  // ...
}
```

In some cases, React *knows* that a value never changes even though it's declared inside the component. For example, the [`set` function](/apis/usestate#setstate) returned from `useState` and the ref object returned by [`useRef`](/apis/useref) are *stable*--they are guaranteed by React to never change between the re-renders. The linter lets you omit them from the list.

<Gotcha>

The linter is your friend, but its powers are limited.

The linter only knows when the dependencies are *wrong*. It doesn't always know *the best* way to solve each case. (Sometimes, it might even be to [remove the Effect altogether!](/learn/you-might-not-need-an-effect)) If the linter rejects your code, we strongly recommend that you fix it. **Suppressing the linter errors leads to bugs that are very difficult to find.** If you're not sure how to fix your Effect, these pages contain many common solutions:

- [Opting Out of Reactivity with Events](/learn/opting-out-of-reactivity-with-events)
- [Removing Unnecessary Dependencies](/learn/removing-effect-dependencies)

</Gotcha>

### What to do when you don't want to re-synchronize {/*what-to-do-when-you-dont-want-to-re-synchronize*/}

In the previous example, you've fixed the lint error by listing `roomId` and `serverUrl` as dependencies.

**However, you could instead "prove" to the linter that these values aren't reactive values,** i.e. that they *can't* change as a result of a re-render. For example, if `serverUrl` and `roomId` don't depend on rendering and always have the same values, you can move them outside the component. Now they don't need to be dependencies:

```js {1,2,11}
const serverUrl = 'https://mychat.com'; // serverUrl is not reactive
const roomId = 'general'; // roomId is not reactive

function ChatRoom() {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

You can also move them *inside the Effect.* They aren't calculated during rendering, so they're not reactive:

```js {3,4,10}
function ChatRoom() {
  useEffect(() => {
    const serverUrl = 'https://mychat.com'; // serverUrl is not reactive
    const roomId = 'general'; // roomId is not reactive
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

**These strategies become especially important for objects and functions.** If you create objects and functions during rendering and then read them from an Effect, they will be different on every render. This will cause your Effect to re-synchronize every time. [Read more about removing unnecessary dependencies](/learn/removing-effect-dependencies) from your Effects.

**Effects are reactive blocks of code.** They re-synchronize when the values you read inside of them change. Unlike event handlers, which only run once per interaction, Effects run whenever synchronization is necessary. You have some control over that with dependencies, but your dependencies must satisfy the linter rule, so you don't really "choose" your dependencies. If you're not happy with your dependencies, you need to fix the code around them.

**If you want to read the latest value of props or state without "reacting" to it and re-synchronizing the Effect,** you can split your Effect into a reactive part (which you'll keep in the Effect) and a non-reactive part (which you'll extract into something called an _Event function_). [Read more about opting out of reactivity with Events.](/learn/opting-out-of-reactivity-with-events) 

### What empty dependencies mean {/*what-empty-dependencies-mean*/}

In the previous example, the Effect did not depend on any reactive values:

```js {11}
const serverUrl = 'https://mychat.com';
const roomId = 'general';

function ChatRoom() {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

What do empty `[]` dependencies mean exactly?

If you think from the component's perspective, the empty `[]` dependency array means this Effect connects to the chat room only when the component mounts, and disconnects only when the component unmounts.

However, if you [think from the Effect's perspective,](#thinking-from-the-effects-perspective) you don't need to think about mounting and unmounting at all. What's important is you've specified what your Effect does to start and stop synchronizing. Today, it has no reactive dependencies. But if you ever make `roomId` or `serverUrl` reactive, your Effect's code won't change. The only thing you'll need to adjust is the dependency array, and the linter will ensure that you don't forget that.

<Note>

During development, React will [remount every component once immediately after mount.](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development) This causes Effects with empty `[]` dependencies to re-synchronize an extra time in development. While it may feel frustrating, this behavior encourages you to *always* write Effects that are resilient to re-synchronizing, and to implement the cleanup function well. Even if your Effect does not use any reactive values today, it might need so in the future. In development, your component will also remount (and re-synchronize the `[]` Effects) whenever you save the component file. This lets you see the results of your edits to the component in a running app without destoying the component's state or fully refreshing the page.

</Note>

<Recap>

TODO

</Recap>

### Challenges {/*challenges*/}

TODO