---
title: 'Lifecycle of Reactive Effects'
---

<Intro>



</Intro>

<YouWillLearn>

</YouWillLearn>

## When causes Effects to re-run? {/*when-causes-effects-to-re-run*/}

Effects let you [synchronize with an external system.](/learn/synchronizing-with-effects) Effects re-run whenever synchronization is needed:

* It always runs when the component is added to the screen (and cleans up when it's removed).
* If your Effect reads values like props and state, **your Effect must re-run after every re-render with different values** of those props and state (and cleans up before re-running).

### Effects initially run on mount and clean up on unmount {/*effects-initially-run-on-mount-and-clean-up-on-unmount*/}

When your component "mounts" (i.e. gets added to the screen), its Effects will run for the first time. Press "Open chat" to mount this `ChatRoom` component [you wrote earlier,](/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed) and take a look at the console inside the sandbox:

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

function ChatRoom() {
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => connection.disconnect();
  }, []);
  return <h1>Welcome to the chat!</h1>;
}

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Close' : 'Open'} chat
      </button>
      {show && <hr />}
      {show && <ChatRoom />}
    </>
  );
}
```

```js chat.js
export function createConnection() {
  // A real implementation would actually connect to the server.
  return {
    connect() {
      console.log('Connecting...');
    },
    disconnect() {
      console.log('Disconnected.');
    }
  };
}
```

```css
body { min-height: 180px; }
```

</Sandpack>

In production, you would only see a single `connect()` call which would produce a single log:

1. `Connecting...`

However, in a development environment like the sandbox above you will see two extra logs before it:

1. `Connecting...` *(development-only)*
1. `Disconnected.` *(development-only)*
1. `Connecting...`

**In development, React stress-tests your component by remounting it one extra time.** This is similar to how you might verify that a door handle works by opening and closing it one extra time before walking into a room. If your Effect is well-written, the extra development-only remounting won't produce any user-visible change in behavior (we'll assume the user doesn't read the console). [Learn how to write Effects that are resilient to remounting.](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development)

Now press "Close chat" in the sandbox above. You will see a single log:

1. `Disconnected.`

When the component is removed from the screen ("unmounted"), its Effects are cleaned up. However, this is not the *only* case in which Effects get cleaned up! Let's see what happens when your Effect has a dependency.

### Effects clean up and re-run after a re-render with different dependencies {/*effects-clean-up-and-re-run-after-a-re-render-with-different-dependencies*/}

Let's say you want to let the user connect to different chat rooms. You've added a select box that lets the user pick the room. You pass the selected room down to the `ChatRoom` component as the `roomId` prop. Try editing the line 6 below from the hardcoded `createConnection('travel')` to `createConnection(roomId)`:

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection('travel'); // Edit this line!
    connection.connect();
    return () => connection.disconnect();
  }, []);
  return <h1>Welcome to the "{roomId}" room!</h1>;
}

export default function App() {
  const [show, setShow] = useState(false);
  const [roomId, setRoomId] = useState('travel');
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Close' : 'Open'} chat
      </button>
      <br />
      <select value={roomId} onChange={e => setRoomId(e.target.value)}>
        <option value="travel">travel</option>
        <option value="social">social</option>
        <option value="music">music</option>
      </select>
      {show && <hr />}
      {show && <ChatRoom roomId={roomId} />}
    </>
  );
}
```

```js chat.js
export function createConnection(roomId) {
  // A real implementation would actually connect to the server.
  return {
    connect() {
      console.log('Connecting to "' + roomId + '"...');
    },
    disconnect() {
      console.log('Disconnected from "' + roomId + '".');
    }
  };
}
```

```css
body { min-height: 180px; }
select { margin-top: 10px; }
```

</Sandpack>

The sandbox above has a [linter configured for React](/learn/editor-setup#linting), so you should see a lint error after editing the code:

```js {3,6}
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, []); // 🔴 React Hook useEffect has a missing dependency: 'roomId'
  return <h1>Welcome to the "{roomId}" room!</h1>;
}
```

This looks like a React error, but really it's React pointing out a logical mistake in this code. By specifying empty `[]` dependencies, you're telling React that your Effect does not depend on *any* value and it's enough to only run it on mount. But this is not true anymore! Your Effect's code uses the `roomId` prop, which may change over time.

If the `roomId` changes, the old `connection` for the previous room is no longer relevant. You want to set up a new `connection` for each new `roomId`. Follow the linter suggestion and add the `roomId` prop to dependencies:

```js {3,6}
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // ✅ All dependencies declared
  return <h1>Welcome to the "{roomId}" room!</h1>;
}
```

This fixes the issue. Now your Effect re-runs after each change to `roomId`. Press "Open chat".

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);
  return <h1>Welcome to the "{roomId}" room!</h1>;
}

export default function App() {
  const [show, setShow] = useState(false);
  const [roomId, setRoomId] = useState('travel');
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Close' : 'Open'} chat
      </button>
      <br />
      <select value={roomId} onChange={e => setRoomId(e.target.value)}>
        <option value="travel">travel</option>
        <option value="social">social</option>
        <option value="music">music</option>
      </select>
      {show && <hr />}
      {show && <ChatRoom roomId={roomId} />}
    </>
  );
}
```

```js chat.js
export function createConnection(roomId) {
  // A real implementation would actually connect to the server.
  return {
    connect() {
      console.log('Connecting to "' + roomId + '"...');
    },
    disconnect() {
      console.log('Disconnected from "' + roomId + '".');
    }
  };
}
```

```css
body { min-height: 180px; }
select { margin-top: 10px; }
```

</Sandpack>

When the component mounts, you'll see similar logs [as before.](#effects-run-on-mount-and-clean-up-on-unmount)

Now select the `"social"` room in the dropdown. **The Effect will clean up and re-run:**

1. `Disconnected from "travel".`
1. `Connecting to "social"...`

Select the `"music"` room in the dropdown. **The Effect will clean up again and re-run again:**

1. `Disconnected from "social".`
1. `Connecting to "music"...`

Finally, press the "Close chat" button. **The Effect will clean up one last time:**

1. `Disconnected from "music".`

When your component re-renders, React checks if any dependencies changed since the last render. If `roomId` changes (e.g. from `"travel"` to `"music"`), React cleans up the last Effect (disconnecting from `"travel"`) and runs the next Effect (connecting to `"music"`). When the component unmounts, React cleans up the last Effect.

<DeepDive title="What happens if you ignore or suppress the linter error?">

If you worked on a React project before, you might have seen or written a line like this:

```js
// eslint-disable-next-line react-hooks/exhaustive-deps
```

This disables the linter for the next line. **This is *not* a valid solution, and is *never* recommended to do.** It leads to bugs that are difficult to debug and come up months later in the ways that you wouldn't expect.

If you're curious, here's the previous example with empty `[]` dependencies and a suppressed lint error:

<Sandpack>

```js
import { useState, useEffect } from 'react';
import { createConnection } from './chat.js';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => {
      connection.disconnect();
    };
    // 🔴 Avoid doing this:
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);
  return <h1>Welcome to the "{roomId}" room!</h1>;
}

export default function App() {
  const [show, setShow] = useState(false);
  const [roomId, setRoomId] = useState('travel');
  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Close' : 'Open'} chat
      </button>
      <br />
      <select value={roomId} onChange={e => setRoomId(e.target.value)}>
        <option value="travel">travel</option>
        <option value="social">social</option>
        <option value="music">music</option>
      </select>
      {show && <hr />}
      {show && <ChatRoom roomId={roomId} />}
    </>
  );
}
```

```js chat.js
export function createConnection(roomId) {
  // A real implementation would actually connect to the server.
  return {
    connect() {
      console.log('Connecting to "' + roomId + '"...');
    },
    disconnect() {
      console.log('Disconnected from "' + roomId + '".');
    }
  };
}
```

```css
body { min-height: 180px; }
select { margin-top: 10px; }
```

</Sandpack>

**Notice how this example is buggy.** Initially, when you open the chat, it gets connected to the selected room. However, when you change the selected room, nothing happens. Since you've "lied" to React about what your Effect depends on, the active connection no longer reflects the UI on the screen.

Even if you really *don't want* the Effect to re-run when a dependency changes, there is always a better solution than suppressing the linter. Keep reading this page to learn about the most common ones.

</DeepDive>

## Effects "react" to props and state {/*effects-react-to-props-and-state*/}

Let's take a moment to appreciate what has happened:

1. You've started using the `roomId` prop in your Effect's code without adjusting its `[]` dependencies.
2. The linter discovered a bug in your code: your Effect only ran on mount, so it would ignore updates to `roomId`.
3. You've edited the dependencies to `[roomId]`. This fixed the bug by making  the Effect re-run more often.

**In other words, Effects are _reactive_.** They "react" to the values from the component body (like props and state). This might remind you of how React always keeps the UI synchronized to the current props and state of your component. By writing Effects, you teach React to [synchronize *other* systems](/learn/synchronizing-with-effects) to the current props and state.

**Values directly inside your component are called _reactive values_.** They participate in the rendering data flow. They include [props](/learn/passing-props-to-a-component), [state](/learn/state-as-a-snapshot), [context](/learn/passing-data-deeply-with-context), and all other values declared directly inside your component's body.

Your Effect must declare every reactive value used inside it as a dependency:

```js {5-6,8-10,15-16,19,24}
// Not a reactive value (outside the component)
import { createConnection } from './chat.js';

function ChatRoom({
  // Props are reactive values
  roomId
}) {
  // Any declarations directly within the component are reactive values
  const [isEncrypted, setIsEncrypted] = useState(true);
  const port = isEncrypted ? 1234 : 5678;

  useEffect(() => {
    // Not a reactive value (it's inside the Effect rather than directly within the component)
    const options = {
      port: port, // A dependency on a reactive value
      isEncrypted: isEncrypted // A dependency on a reactive value
    };
    const connection = createConnection(
      roomId, // A dependency on a reactive value
      options
    );
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, isEncrypted, port]); // ✅ All dependencies are declared

  // ...
}
```

This guarantees when the reactive values change, your Effect synchronizes them with your external system.

## You can't "choose" your dependencies {/*you-cant-choose-your-dependencies*/}





