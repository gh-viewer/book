## Authenticating users with GitHub

The goal of this chapter is to implement GitHub's OAuth flow in order to get a an access token we'll use in the app to access GitHub's API.

To achieve this, we'll do 3 things: set up an authentication server, implement a way to store the user's access token in the app, and finally implement the full authentication flow in the app.

### Setting up the authorisation server

First, let's setup a server implementing [GitHub's OAuth flow](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/about-authorization-options-for-oauth-apps/#web-application-flow). If you haven't done it already, you'll need to [register your app on GitHub](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/registering-oauth-apps/). This flow can be implemented by any HTTP server using your language and framework of choice, but in this guide we'll use JavaScript with node.

The authentication server implementation is open-source, [available in this repository](https://github.com/PaulLeCam/gh-viewer-server). If you want to set it up quickly, you can [deploy it to Heroku using this link](https://heroku.com/deploy?template=https://github.com/PaulLeCam/gh-viewer-server), the free plan is enough to support our use case. Here is the server code:

```js
const got = require('got')
const { send } = require('micro')
const redirect = require('micro-redirect')
const { stringify } = require('querystring')
const { parse } = require('url')

const { CLIENT_ID, CLIENT_SECRET, SCOPE } = process.env
const AUTH_PARAMS = stringify({
  client_id: CLIENT_ID,
  scope: SCOPE || '',
})
const AUTH_URL = `https://github.com/login/oauth/authorize?${AUTH_PARAMS}`
const TOKEN_URL = 'https://github.com/login/oauth/access_token'

module.exports = async (req, res) => {
  try {
    const { pathname, query } = parse(req.url, true)

    switch (pathname) {
      // Authorize request from client -> redirect to GitHub authorize URL
      case '/authorize':
        redirect(res, 303, AUTH_URL)
        return
      // Callback from GitHub -> retrieve the credentials using the code
      case '/callback': {
        const auth = await got(TOKEN_URL, {
          body: {
            client_id: CLIENT_ID,
            client_secret: CLIENT_SECRET,
            code: query.code,
          },
          json: true,
        })
        redirect(res, 303, `/success?${stringify(auth.body)}`)
        return
      }
      // Redirected to client-known success URL -> end of flow
      case '/success':
        return 'OK'

      default:
        send(res, 404, 'Not found')
    }
  } catch (err) {
    console.log(req.url, err)
    send(res, 500, 'Internal error')
  }
}
```

Let's go through the different sections, first we import the external dependencies and built-in node modules. We use [micro](https://github.com/zeit/micro) as it is very simple to setup: we simply need to export a function as the request handler that will support the following paths:

* `/authorize`: This is the URL that will be loaded by the client, all we need to do is redirect it to GitHub's authorization page with our client ID and optionally a scope, defined by the `AUTH_URL` constant.
* `/callback`: This is the callback URL that will be called by GitHub after the user successfully authorized our application. It must match the "Authorization callback URL" provided in your GitHub app's settings. This callback will receive a temporary authentication code that must be exchanged for an access token by GitHub's server.
* `/success`: This is the URL the server will redirect the client to when the access token is retrieved from GitHub's server, and will be provided in the query parameters so that the client can read it and start using it.

### Adding Redux

Before implementing the authentication flow in the client, we need to implement a way to read and write the access token and possibly other information at will, and to persist it after the user leaves the application to avoid having to go through the flow every time the app is used.

To achieve it, we'll use [Redux](http://redux.js.org/) and [Redux Persist](https://github.com/rt2zz/redux-persist). As with many other library choices in this guide, these choices are out of simplicity because I have already used these libraries before and am familiar with how to implement solutions using them. Depending on you use cases, other libraries, or a custom solution of your choice.

Let's add these libraries to the project:

```bash
yarn add react-redux redux redux-persist
```

Now let's setup our application store with persistence, by creating a `Store.js` file in the `src` folder, with the following contents:

```js
// @flow

import { AsyncStorage } from 'react-native'
import { combineReducers, createStore } from 'redux'
import { createPersistor, getStoredState } from 'redux-persist'

export type ActionType = 'AUTH_INVALID' | 'AUTH_SUCCESS'

export type Action = {
  type: ActionType,
  [key: string]: mixed,
}

const persistConfig = { storage: AsyncStorage }

const getInitialState = (): Promise<Object> => {
  return new Promise(resolve => {
    getStoredState(persistConfig, (err, state) => {
      if (err) {
        console.warn('Failed to get stored state', err)
        resolve({})
      } else {
        resolve(state)
      }
    })
  })
}

const authReducer = (state = null, action: Action) => {
  switch (action.type) {
    case 'AUTH_INVALID':
      return null
    case 'AUTH_SUCCESS':
      return action.auth
    default:
      return state
  }
}

const reducer = combineReducers({ auth: authReducer })

export const create = async () => {
  const store = createStore(reducer, await getInitialState())
  createPersistor(store, persistConfig)
  return store
}
```

> TODO: describe Store logic

> TODO: extract SceneLoader component

> TODO: add StoreProvider component

### Setting up the app's authentication flow

> TODO: add EnvironmentProvider component

> TODO: setup App component, update entry points



