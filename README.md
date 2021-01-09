# react-three-game-engine

[![Version](https://img.shields.io/npm/v/react-three-game-engine?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/react-three-game-engine)
[![Downloads](https://img.shields.io/npm/dt/react-three-game-engine.svg?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/react-three-game-engine)

A very early experimental work-in-progress package to help provide game engine functionality for [react-three-fiber](https://github.com/pmndrs/react-three-fiber).

### Key Features
- [planck.js](https://github.com/shakiba/planck.js/) 2d physics integration
- Physics update rate independent of frame rate
- `OnFixedUpdate` functionality
- Additional React app running in web worker, sync'd with physics, for handling game logic etc

### Note
The planck.js integration currently isn't fully fleshed out. I've only been adding in support 
for functionality on an as-needed basis for my own games.

Note: if you delve into the source code for this package, it's a bit messy!

## Get Started

### General / Physics

1. Install required packages

`npm install react-three-game-engine`

plus

`npm install three react-three-fiber planck-js`

2. Import and add `<Engine/>` component within your r3f `<Canvas/>` component. 

```jsx
import { Engine } from 'react-three-game-engine'
import { Canvas } from 'react-three-fiber'
```

```jsx
<Canvas>
    <Engine>
        {/* Game components go here */}
    </Engine>
</Canvas>
```

3. Create a planck.js driven physics body

```jsx
import { useBody, BodyType, BodyShape } from 'react-three-game-engine'
import { Vec2 } from 'planck-js'
```

```jsx
const [ref, api] = useBody(() => ({
    type: BodyType.dynamic,
    position: Vec2(0, 0),
    linearDamping: 4,
    fixtures: [{
        shape: BodyShape.circle,
        radius: 0.55,
        fixtureOptions: {
            density: 20,
        }
    }],
}))
```

4. Control the body via the returned api

```jsx
api.setLinearVelocity(Vec2(1, 1))
```

5. Utilise `useFixedUpdate` for controlling the body

```jsx
import { useFixedUpdate } from 'react-three-game-engine'
```

```jsx

const velocity = Vec2(0, 0)

// ...

const onFixedUpdate = useCallback(() => {

    // ...

    velocity.set(xVel * 5, yVel * 5)
    api.setLinearVelocity(velocity)

}, [api])

useFixedUpdate(onFixedUpdate)

```

### React Logic App Worker

A key feature provided by react-three-game-engine is the separate React app running 
within a web worker. This helps keep the main thread free to handle rendering etc, 
helps keep performance smooth.

To utilise this functionality you'll need to create your own web worker. You can 
check out my repo [react-three-game-starter](https://github.com/simonghales/react-three-game-starter) 
for an example of how you can do so with `create-react-app` without having to eject.

1.

Create a React component to host your logic React app, export a new component wrapped with 
`withLogicWrapper`

```jsx
import {withLogicWrapper} from "react-three-game-engine";

const App = () => {
    // ... your new logic app goes here
}

export const LgApp = withLogicWrapper(App)
```

2.

Set up your web worker like such 
(note requiring the file was due to jsx issues with my web worker compiler)

```jsx
/* eslint-disable no-restricted-globals */
import {logicWorkerHandler} from "react-three-game-engine";

// because of some weird react/dev/webpack/something quirk
(self).$RefreshReg$ = () => {};
(self).$RefreshSig$ = () => () => {};

logicWorkerHandler(self, require("../path/to/logic/app/component").LgApp)
```

3.

Import your web worker (this is just example code)

```jsx
const [logicWorker] = useState(() => new Worker('../path/to/worker', { type: 'module' }))
```

4.

Pass worker to `<Engine/>`

```jsx
<Engine logicWorker={logicWorker}>
    {/* ... */}
</Engine>
```

You should now have your Logic App running within React within your web worker, 
synchronised with the physics worker as well. 

### Controlling a body through both the main, and logic apps.

To control a body via either the main or logic apps, you would create the body 
within one app via `useBody` and then within the other app you can get api 
access via `useBodyApi`.

However you need to know the `uuid` of the body you wish to control. By default 
the uuid is one generated via threejs, but you can specify one yourself.

1.

Create body

```jsx
useBody(() => ({
    type: BodyType.dynamic,
    position: Vec2(0, 0),
    linearDamping: 4,
    fixtures: [{
        shape: BodyShape.circle,
        radius: 0.55,
        fixtureOptions: {
            density: 20,
        }
    }],
}), {
    uuid: 'player'
})
```

2.

Get body api

```jsx
const api = useBodyApi('player')

// ...

api.setLinearVelocity(Vec2(1, 1))

```

So with this approach, you can for example initiate your player body via the logic app, 
and then get api access via the main app, and use that to move the body around.

3.

Additionally, if you are creating your body in main / logic, you'll likely want to have 
access to the position / rotation of the body as well.

You can use `useSubscribeMesh` and pass in a ref you've created, which will synchronize 
with the physics body.

```jsx
const ref = useRef<Object3D>(new Object3D())
useSubscribeMesh('player', ref.current, false)

// ...

return (
    <group ref={ref}>
        {/*...*/}
    </group>
)

```
