# Require Universal Module
This package provides the core server-side rendering tools for [React Universal Component](https://github.com/faceyspacey/react-universal-component) or any similar
async components/modules you'd like to make. It has been extracted/abstracted specifically so you don't have to rely on React Universal Component, React Loadable, etc. I.e. so you can make your own.

What you make with it is up to you--*it doesn't need to be limited to just React components; that's why it's called "Require Universal* ***Module***."

It provides 4 requirements for successful server-side rendering:
- synchronous rendering of modules on the server that otherwise are async on the client
- recording of those module IDs so, using tools like [Webpack Flush Chunks](https://github.com/faceyspacey/webpack-flush-chunks), you can convert them to the additional chunks you'd like to serve in initial requests
- the capability to synchronously render async components on the client as well, if their corresponding chunk was embedded in the initial request
- a paired async importing mechanism to enable you to dynamically toggle between async and sync importing as needed

## Installation

```
yarn add require-universal-module
```

## Motivation

The code has been cracked for while now for Server Side Rendering and Code-Splitting *individually*. Accomplishing both *simultaneously* has been an impossibility without jumping through major hoops (which few have succeeded at) or using a *framework*, specifically Next.js. This package does not in fact solve the problem--[Webpack Flush Chunks](https://github.com/faceyspacey/webpack-flush-chunks) does. But it's a pre-requisite, and the first general solution offering you "primitives" in an area of the "SSR code-splitting *stack*" where you may have custom needs. 

In addition to being an abstraction to create your own Async Components, it solves many of the problems of React Loadable and adds several new features:
- HMR for your async/sync "universal" modules
- support for Webpack's brand new *webpackChunkName "magic comment" feature*
- `onLoad` hook to extract other exports from the module to do things like call `store.replaceReducer()`
- a timeout option, at which point an error is thrown (inspired by Vue's latest code-splitting component)
- ability to use a function instead of a promise, e.g. a function that calls `require.ensure`. Note: Webpack's `require.ensure` still has several capabilities that the new `import().then` spec does not
- miscellaneous smaller features that clean up the API and automate many possible use-cases

Most importantly though, it does not assume you are creating a React component. This does not even need to be used with React. It removes all the complexity surrounding
dynamically toggling between async vs. sync loading/requiring, and preparing the modules actually required for server-side flushing into initially embedded chunks.

Let's take a look at the simplest implementation you might have.


## Usage
Below is a quick overview of the Usage. It's pretty abstract at first (as it's meant to be). 
Just give it a quick glance before checking out the [Basic Example](#basic-example) after.

*my-super-duper-cool-universal-package*:
```js
import requireUniversalModule from 'require-universal-module'

const tools = requireUniversalModule(import('./Foo'), {
  resolve: require.resolveWeak('./Foo')
})

const { requireSync, requireAsync, addModule, mod } = tools

// mod is the returned module from requireSync done once for you
if (mod) {
  doSomethingWithModule(mod)
}

// ...
// perhaps later in execution the module will be synchronsouly available (see reasons below)
mod = requireSync() // yes, with no params
doSomethingWithModule(mod)

// whenever you want to mark the module as used, such as when a corresponding React 
// component mounts, you simply call:
addModule() // again, with params
```

```js
// on the client as the user navigates your app, call:
requireAsync()
  .then(mod => doSomething(mod)) 
  .catch(error => handleTimeoutOrOtherError(error))
// again, no params; what's required will be retreived from a closure
```

*ssr.js in user-land:*
```js
import { flushModuleIds, flushChunkNames } from 'require-universal-module/server'

const app = ReactDOMServer.renderToString(<App />)

const moduleIds = flushModuleIds()
const chunkNames = flushChunkNames()

const scripts = convertModuleIdsToScripts(moduleIds)
// or const scripts = convertChunkNamesToScripts(moduleIds)

res.send(render(app, scripts))
```
> Note: for the 2 imported functions you should just re-export from your package for consistency :)

## Basic Example (where this finally makes sense)
The simplest but complete component--rather, HoC--you could make is as follows:

```js
import React from 'react'
import requireUniversalModule from 'require-universal-module'

export default function createSuperDuperForwardThinkingAsyncComponent(asyncComponent, opts) {
  const { loading: Loading, ...options } = opts
  const tools = requireUniversalModule(asyncComponent, options)
  const { requireSync, requireAsync, addModule, mod } = tools

  let Component = mod // initial syncronous require attempt done for us :)

  return class SuperDuperForwardThinkingAsyncComponent extends React.Component {
    constructor(props) {
      super(props)

      if (!Component) {
        // try one more syncronous require at render time,
        // in case chunk comes after main.js
        Component = requireSync()
      }

      this.state = { hasComponent: !!Component }
    }

    componentWillMount() {
      addModule() // record the module for SSR flushing :)

      if (this.state.hasComponent) return

      requireAsync()
        .then(mod => {
          Component = mod // for HMR updates component must be in closure, not state
          this.setState({ hasComponent: !!Component })
        })
    }

    render() {
      const { hasComponent } = this.state
      const props = this.props
      return hasComponent ? <Component {...props} /> : <Loading {...props} />
    }
  }
}
```
> We highly recommend you checkout the React Universal Component Implementation, https://github.com/faceyspacey/react-universal-component/blob/master/src/index.js , to see what you can do.

*And that's it! Now your components can render both asynchronously and synchronously, and you can record which modules were used which you can triangulate into which chunks were used.*

**The takeaway is this:** 
- call `requireUniversalModule(asyncComponent, options)` once
- receive these tools: `{ requireSync, requireAsync, addModule, mod }`
- use them at the appropriate times + places

If you're wondering what `mod` is and correctly assumed that it's the return of `requireSync`, you'd be correct. Internally, it's called once for you. 
If `mod` is defined, `requireUniversalModule` was able to synchcronously require the given module. If it was unsuccessful, it's likely because you are on
the client in your `main.js` script and your `0.js` (for example) bundle did not come before it :). This package does not assume that setup, though that's recommended. This package enables you to solve these problems as you see fit.

## More on Embedding Chunks in Your Initial Request
Basically it takes jumping through a few hoops--which [Webpack Flush Chunks](https://github.com/faceyspacey/webpack-flush-chunks) solves for you--to have your `0.js` bundle come before your `main.js`, which ultimately requires that webpack bootstrap code be broken into a 3rd script, `bootstrap.js`, that comes before the other 2.

So if you have not done that, and your page *instead* looks like this instead:

```js
<script src="main.js" />
<script src="0.js" />
<script src="1etc.js" />
<script>window.render()</script>
```
>yes, to make this work, `main.js` can't call `ReactDOM.render()`. Therefore you have to assign it to `window` and execute it after all bundles are evaluated.

Then you need to manually call `requireSync` again at render time, as you can see in the constructor of the above example component. If you are making a library to be used as a 3rd party package like [React Universal Component](https://github.com/faceyspacey/react-universal-component), you will want to cover both possibilities.

To be clear, the ideal setup is this:

```js
<script src="bootstrap.js" />
<script src="0.js" />
<script src="1etc.js" />
<script src="main.js" />
```

[Webpack Flush Chunks](https://github.com/faceyspacey/webpack-flush-chunks) will create this for you (along with figuring out which chunks to render based on modules flushed). This way you can call `ReactDOM.render()` as usual, and so your initial synchronous require on the client works, which perhaps more importantly is a requirement for *Hot Module Replacement*--if your component is immediately assigned to state and never stored in a closure, on re-renders triggered by HMR, the old component will still show.

Don't worry about all this right now. Chances are if you're reading this instead of just using [React Universal Component](https://github.com/faceyspacey/react-universal-component), you have a strong grasp of how these family of packages are supposed to be used :)

Let's take a look at the API (and all the `options` goodness you can supply).

## API + Options

```js
requireUniversalModule(() => import('./Foo'), options)
```
> note: the first argument can be a function that returns a promise, or a promise itself, or a function that takes a node-style callback, which it can call with the module (e.g: `requireUniversalModule(callback => require.ensure([], require => callback(null, require('./Foo'))), options)`. Most common though is a function that returns a promise, as that can be used to properly split your code into chunks without the user/developer having to create their own wrapper function or HoC. Ultimately this likely won't matter, as you'll pass this choice on to the developer.

**The Options:**

- `resolve`: `() => require.resolveWeak('./Foo')` || `require.resolveWeak('./Foo')` -- for Webpack
- `path`: `path.join(__dirname, './Example')` -- for Babel
- `key`: `'foo'` || `module => module.foo` || `null` -- default: babelInterop that chooses `default` on es6 modules and `module.exports` on es5 modules
- `chunkName`: `'myChunkName'` -- must match `import(/* webpackChunkName: "myChunkName" */ './Foo')`
- `timeout`: number of milliseconds before `requireAsync` throws -- default: `15000`
- `onLoad`: `module => store.replaceReducer({ ...otherReducers, foo: module.reducer })` -- you can obviously do anything here. Unlike the return of `requireSync` and `requireAsync`, it bypasses the `key` option and receives the entire module.

Ok, let's go over the options real quick. 

Basically `resolve` is the most important one and it's meant specifically for calling webpack's lesser known `require.resolveWeak` function with the same path as the first parameter to `requireUniversalModule(import('./Foo'))`. What it does is attempt to require the module without telling Webpack to mark it as a dependency. That means it won't be included in the chunk where it is called. In other words, it assumes that it's embedded in the page elsewhere, sort of like Webpack's `externals` feature. What this does that's so important is it allows `./Foo` to be split into a separate chunk, while allowing synchronous requires in other environments scenarios (i.e. server-side rendering, and the initial evaluation of your page where perhaps the chunk was pre-embedded into the page). Typically as *React Loadable* promoted, this should be a function that calls `require.resolveWeak`. However, that doesn't always need to be the case. Scenarios where you don't need to do this are where you create your own function that calls `requireUniversalModule`. At which point, it's up to you to not just call the function on initial evaluation of the page, but later in response to some action. *Hey, what can I say, having to make these options functions never felt as clean as what you can do in Vue or Next.js*. 

Next: the `path` option is only needed with a Babel server. While I recommend everyone uses a webpack server (because of it's awesome Universal HMR capabilities), that's not the case for everyone, and since this package is primarily for you to make packages, you should consider the possibility that your users have Babel servers. *Also note: even if you're using a Babel server, you must supply the webpack `resolve` option, as your code will be evaluated synchronously on the client as well. The weak dependency must be able to synchronsouly resolve on both the server and the client.*

The `key` is used to resolve which export you want from a module. It's pretty self-explanatory: provide a key name as a string or a function that does it, or `null` if you want the whole module. That said, the `key` option most often isn't needed, as by default it will automatically find the `default` export for ES6 modules, and the value of `module.exports` for ES5 modules. However, there is a very useful thing you can do with it in combination with the `onLoad` option: 

- you can create a module that has several exports
- use the `key` option to find the export you want to be the component
- use your `onLoad` function which *always* receives the entire module to find a different export, and do something with it such as call `store.replaceReducer({ ...reducers, foo: module.otherExport })`

The `timeout` is a feature inspired by [Vue's latest async component](https://vuejs.org/v2/guide/components.html#Advanced-Async-Components), which itself was inspired by React Loadable. It lets you specify essentially a maximum time the async module has to load before an error is thrown. In *React Universal Component*, it is used to trigger the rendering of an `<Error />` component.

Lastly, `chunkName` is to be used with [Webpack's latest magic comment feature](https://webpack.js.org/guides/code-splitting-async/#chunk-names) which only came out several weeks ago in version 2.4.1. Essentially you use a magic comment as shown above to name your chunk. Your chunk will be given that name (though not the individual scripts). It will be accessible in stats at: `webpackStats.assetsByChunkName['myChunkName'] === ['0.js', '0.js.map', '0.css', '0.css.map']`. This greatly simplifies the rendering of your chunks on the server, which [Webpack Flush Chunks](https://github.com/faceyspacey/webpack-flush-chunks) further simplifies. Accordingly, an array/set will be kept of the `chunkNames` used instead of `moduleIds`, which brings us to the final section:

## Flushing

All you have to do is flush the `moduleIds` or `chunkNames` after you render the app on the server to take this to the next stage (i.e. what *Webpack Flush Chunks* does for you):

A basic example was above, but I'll show it one more time in depth:

```js
import { flushModuleIds, flushChunkNames } from 'require-universal-module/server'
import flushChunks from 'webpack-flush-chunks'

export default function serverRender(req, res) => {
  const app = ReactDOMServer.renderToString(<App />)

  const { js, styles } = flushChunks(webpackStats, { 
    chunkNames: flushChunkNames(), // do one of the 2
    // moduleIds: flushModuleIds(),
  })

  res.send(
    `<!doctype html>
      <html>
        <head>
          ${styles} // will contain stylesheets for: main.css, 0.css, 1.css, etc
        </head>
        <body>
          <div id="root">${app}</div>
          ${js} // will contain scripts for: bootstrap.js, 0.js, 1.js, etc, main.js
        </body>
      </html>`
  )
}
```
*et voila!*

If you're wondering, how to get the webpack stats, well go checkout [Webpack Flush Chunks](https://github.com/faceyspacey/webpack-flush-chunks) and it's corresponding boilerplates. The boilerplates offer some fresh and idiomatic takes on handling the stats and generally how you should put together your Node server. Enjoy!

## Contribution

