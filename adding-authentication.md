## Authenticating users with GitHub

The goal of this chapter is to implement GitHub's OAuth flow in order to get a an access token we'll use in the app to access GitHub's API.

To achieve this, we'll do 3 things: set up an authentication server, implement a way to store the user's access token in the app, and finally implement the full authentication flow in the app.

### Setting up the authentication server

First, let's setup a server implementing [GitHub's OAuth flow](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/about-authorization-options-for-oauth-apps/#web-application-flow). If you haven't done it already, you'll need to [register your app on GitHub](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/registering-oauth-apps/). This flow can be implemented by any HTTP server using your language and framework of choice, but in this guide we'll use JavaScript with node.

The authentication server implementation is open-source, [available in this repository](https://github.com/PaulLeCam/gh-viewer-server). Here is the server code:

```js
const got = require('got')
const { send } = require('micro')
const redirect = require('micro-redirect')
const { stringify } = require('querystring')
const { parse } = require('url')

const { CLIENT_ID, CLIENT_SECRET, SCOPE } = process.env
const AUTH_PARAMS = stringify({
  client_id: CLIENT_ID,
  scope: SCOPE ? SCOPE : 'public_repo read:org',
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

Let's go through the different sections, first we import the external dependencies and built-in node modules. We use [micro](https://github.com/zeit/micro) as it is very simple to setup: we simply need to export a function as the request handler that will support the following endpoints:

* `/authorize`: This is the URL that will be loaded by the client, all we need to do is redirect it to GitHub's authorization page with our client ID and optionally a scope, defined by the `AUTH_URL` constant.
* `/callback`: This is the callback URL that will be called by GitHub after the user successfully authorized our application. It must match the "Authorization callback URL" provided in your GitHub app's settings. This callback will receive a temporary authentication code that must be exchanged for an access token by GitHub's server.
* `/success`: This is the URL the server will redirect the client to when the access token is retrieved from GitHub's server, and will be provided in the query parameters so that the client can read it and start using it.

### Deploying the authentication server

Using the following commands, you can easily deploy your own authentication server using [Zeit's now service](https://zeit.co/now).

```bash
# Prerequisites
yarn global add now
now --login
# Only needed once
now secrets add gh-client-id [your client id]
now secrets add gh-client-secret [your client secret]
# Deploy when needed
now gh-viewer/server#master -e CLIENT_ID=@gh-client-id -e CLIENT_SECRET=@gh-client-secret -e SCOPE='public_repo read:org' --public
```

You can also [deploy it to Heroku using this link](https://heroku.com/deploy?template=https://github.com/gh-viewer/server), the free plan is enough to support our use case.

If you haven't done it already, don't forget to set the "Authorization callback URL" of your GitHub app's configuration to `https://[your-domain.tld]/callback` so GitHub will properly redirect you app's users once authenticated.

### Adding Redux

Before implementing the authentication flow in the client, we need to implement a way to read and write the access token and possibly other information at will, and to persist it after the user leaves the application to avoid having to go through the flow every time the app is used.

To achieve it, we'll use [Redux](http://redux.js.org/) and [Redux Persist](https://github.com/rt2zz/redux-persist). As for many other libraries in this guide, these choices are out of simplicity because I have already used these libraries before and am familiar with how to implement solutions using them. Depending on you use cases, other libraries or a custom solution of your choice might offer better solutions.

Let's add these libraries to the project:

```bash
yarn add react-redux@^5.0.5 redux@^3.7.2 redux-persist@^4.8.2
```

#### Creating the store

Now let's setup our application store with persistence, by creating a `Store.js` file in the `src` folder, with the following contents:

```js
// @flow

import { AsyncStorage } from 'react-native'
import { combineReducers, createStore } from 'redux'
import { createPersistor, getStoredState } from 'redux-persist'

export type Action =
  | {
      type: 'AUTH_INVALID',
    }
  | {
      type: 'AUTH_SUCCESS',
      auth: {
        access_token: string,
        scope?: string,
      },
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

As you can notice, we'll use Flow in this module to define some types. It is convenient when working with Redux to make sure the actions payloads are properly defined and handled.

In order to store the application state, we use Redux-Persist and configure it to use the `AsyncStorage` API provided by React Native and React Native for Web. The `getInitialState()` function will try to get the existing state from storage, or return an empty Object.

The `authReducer()` will handle the authentication actions to update the store, and is part of the main `reducer`. This is not really necessary at this point as we only have one reducer, but namespacing the auth state like this is convenient to avoid more refactoring than necessary once we add more unrelated data to the state.

Finally, this module exports a `create()` function that will create the store using the initial state, and persist it.

#### Providing the state to React components

Before we go into the actual provider implementation, let's extract the `QueryLoader` component we created in the  `HomeScreen` module into a separate module, as we're going to start using it in more places. Let's create a `ScreenLoader.js` file in `src/components`, with the following contents:

```js
// @flow

import React from 'react'
import { ActivityIndicator, View } from 'react-native'
import { Text } from 'react-native-elements'

import sharedStyles from './styles'

const ScreenLoader = ({ text }: { text?: string }) =>
  <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
    <View style={sharedStyles.mainContents}>
      <ActivityIndicator animating size="large" />
      <Text h2 style={sharedStyles.textCenter}>{text || 'Loading...'}</Text>
    </View>
  </View>

export default ScreenLoader
```

Now let's create a `StoreProvider.js` file in the `src/component` folder, with the following contents:

```js
// @flow

import React, { Component, type Element } from 'react'
import { Provider } from 'react-redux'

import { create } from '../Store'

import ScreenLoader from './ScreenLoader'

export default class StoreProvider extends Component {
  props: {
    children?: Element<*>,
  }
  state: {
    store?: Object,
  } = {}

  componentDidMount() {
    this.createStore()
  }

  async createStore() {
    this.setState({ store: await create() })
  }

  render() {
    return this.state.store
      ? <Provider store={this.state.store}>{this.props.children}</Provider>
      : <ScreenLoader />
  }
}
```

In this module, we use the `Provider` component from React Redux to inject our application's store. Because creating the store is asynchronous, we display the `ScreenLoader` until the store is available.

### Setting up the app's authentication flow

Now that we have a store to save and retrieve the access token, let's implement the authentication flow in the client, using the server we previously created.

First, let's update the `Environment.js` file to use the access token. We will also provide an `onUnauthorized()` callback in the `create()` function, called when GitHub's API return a response with status `401 Unauthorized`. This will happen if the user decides to remove our app, the access token will no longer be valid.

In this module, we also export an `EnvironmentPropType` that will be used by the components.

```js
// @flow

import PropTypes from 'prop-types'
import { Environment, Network, RecordSource, Store } from 'relay-runtime'

export const EnvironmentPropType = PropTypes.instanceOf(Environment)

export const create = (access_token: string, onUnauthorized: () => void) => {
  const fetchQuery = (operation: Object, variables?: Object) => {
    return fetch('https://api.github.com/graphql', {
      method: 'POST',
      headers: {
        authorization: `bearer ${access_token}`,
        'content-type': 'application/json',
      },
      body: JSON.stringify({
        query: operation.text,
        variables,
      }),
    }).then(res => {
      if (res.ok) return res.json()
      else if (res.status === 401) {
        onUnauthorized()
      } else {
        const error: Object = new Error(res.statusText || 'Request error')
        error.status = res.status
        throw error
      }
    })
  }

  const network = Network.create(fetchQuery)
  const source = new RecordSource()
  const store = new Store(source)

  return new Environment({ network, store })
}
```

Now let's create a create a new file, `EnvironmentProvider.js`, in the `src/components` folder:

```js
// @flow

import React, { Component, type Element } from 'react'
import { StyleSheet, View, WebView } from 'react-native'
import { Button, Text } from 'react-native-elements'
import { connect } from 'react-redux'
import type { Environment } from 'relay-runtime'
import { parse } from 'url'

import { create, EnvironmentPropType } from '../Environment'
import type { Action } from '../Store'

import ScreenLoader from './ScreenLoader'
import sharedStyles from './styles'

type AuthState = 'UNAUTHORIZED' | 'LOADING' | 'AUTHORIZE' | 'AUTHORIZED'
type NavigationState = {
  loading: boolean,
  url: string,
}
type Props = {
  access_token: ?string,
  children: Element<*>,
  dispatch: (action: Action) => void,
}

// Edit here if you want to use your own authentication server
const AUTH_SERVER = 'ghviewer.herokuapp.com'

class EnvironmentProvider extends Component {
  static childContextTypes = {
    environment: EnvironmentPropType,
  }

  props: Props
  state: {
    auth: AuthState,
    environment: ?Environment,
  }

  constructor(props: Props) {
    super(props)
    this.state = this.getState(props)
  }

  getState(props: Props) {
    return props.access_token
      ? {
          auth: 'AUTHORIZED',
          environment: create(props.access_token, this.onUnauthorized),
        }
      : {
          auth: 'UNAUTHORIZED',
          environment: null,
        }
  }

  getChildContext() {
    return {
      environment: this.state.environment,
    }
  }

  componentWillReceiveProps(nextProps: Props) {
    if (nextProps.access_token !== this.props.access_token) {
      this.setState(this.getState(nextProps))
    }
  }

  onUnauthorized = () => {
    this.props.dispatch({ type: 'AUTH_INVALID' })
  }

  onNavigationStateChange = (state: NavigationState) => {
    const { host, pathname, query } = parse(state.url, true)
    if (
      host === AUTH_SERVER &&
      pathname === '/success' &&
      query &&
      query.access_token
    ) {
      this.props.dispatch({
        type: 'AUTH_SUCCESS',
        auth: query,
      })
    }
  }

  onLoadEnd = () => {
    if (this.state.auth === 'LOADING') {
      this.setState({ auth: 'AUTHORIZE' })
    }
  }

  onPressAuthorize = () => {
    this.setState({ auth: 'LOADING' })
  }

  render() {
    if (this.state.environment) {
      return this.props.children
    }
    const { auth } = this.state

    if (auth === 'UNAUTHORIZED') {
      return (
        <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
          <View style={sharedStyles.mainContents}>
            <Text h3 style={styles.title}>Welcome to GH Viewer!</Text>
            <Text style={styles.contents}>
              In order to use this application, you need to authorize it to
              access some of your GitHub data
            </Text>
            <Button
              backgroundColor="#28a745"
              icon={{ name: 'shield', type: 'octicon' }}
              onPress={this.onPressAuthorize}
              title="Authorize with GitHub"
            />
          </View>
        </View>
      )
    }

    const webView = (
      <WebView
        onLoadEnd={this.onLoadEnd}
        onNavigationStateChange={this.onNavigationStateChange}
        source={{ uri: `https://${AUTH_SERVER}/authorize` }}
        style={auth === 'AUTHORIZE' ? sharedStyles.scene : styles.webviewHidden}
      />
    )
    const loader = auth === 'AUTHORIZE' ? null : <ScreenLoader />

    return (
      <View style={styles.container}>
        {loader}
        {webView}
      </View>
    )
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  contents: {
    marginVertical: 15,
  },
  title: {
    textAlign: 'center',
  },
  webviewHidden: {
    height: 0,
  },
})

export default connect(state => ({
  access_token: (state.auth && state.auth.access_token) || null,
}))(EnvironmentProvider)
```

This component is responsible for providing the Relay environment to children components, but in order to do so it needs the access token, that will be provided by the store. When the access token is already available, this provider will create the environment and make it available to its children using its context.

When the access token is not available, this component will be responsible for handling the authentication flow, by displaying a [WebView](https://facebook.github.io/react-native/releases/0.42/docs/webview.html) loading the `/authorize` endpoint of our authentication server. The flow is applied as follow:

1. The app is in the `UNAUTHORIZED` state and displays a welcome message to the user, with a button asking to authorize the app with GitHub.
2. When the user clicks the button, it calls the `onPressAuthorize()` handler, that sets the state to `LOADING`. The component will render the WebView to load the authorization endpoint, and display the `ScreenLoader` until the page is loaded.
3. Once the page is loaded, the `onLoadEnd()` callback will be triggered, changing the state to `AUTHORIZE`, which will cause the component to hide the loader and display the contents of the WebView, GitHub's authorization page.
4. The user will then go through GitHub's and our server's authorization flow, that will end-up to being redirected to the `/success` endpoint of our authorization server, with the access token provided in the query params. This state change will be handled by the `onNavigationStateChange()` callback, that will dispatch the `AUTH_SUCCESS` action to our Redux store.
5. The store will then be able to provide the access token to our component, allowing it to create the Relay environment and render its child component.

The next step will be to update our `HomeScreen.js` file to use the environment provided rather than create its own, and the `ScreenLoader` we previous extracted:

```js
// @flow

import React, { Component } from 'react'
import { View } from 'react-native'
import { Button, Icon, Text } from 'react-native-elements'
import { graphql, QueryRenderer } from 'react-relay'

import { EnvironmentPropType } from '../Environment'

import ScreenLoader from './ScreenLoader'
import sharedStyles from './styles'

type QueryErrorProps = {
  error: Error,
  retry: () => void,
}
const QueryError = ({ error, retry }: QueryErrorProps) =>
  <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
    <View style={sharedStyles.mainContents}>
      <Text h2 style={sharedStyles.textCenter}>
        {error.message || 'Request error'}
      </Text>
    </View>
    <View style={sharedStyles.bottomContents}>
      <Button onPress={retry} title="Retry" />
    </View>
  </View>

type HomeProps = {
  viewer: {
    login: string,
  },
}
const Home = ({ viewer }: HomeProps) =>
  <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
    <View style={sharedStyles.mainContents}>
      <Icon name="octoface" size={60} type="octicon" />
      <Text h2 style={sharedStyles.textCenter}>
        Welcome, {viewer.login}!
      </Text>
    </View>
  </View>

export default class HomeScreen extends Component {
  static contextTypes = {
    environment: EnvironmentPropType.isRequired,
  }

  render() {
    return (
      <QueryRenderer
        environment={this.context.environment}
        query={graphql`
          query HomeScreenQuery {
            viewer {
              login
            }
          }
        `}
        render={({ error, props, retry }) => {
          if (error) {
            return <QueryError error={error} retry={retry} />
          } else if (props) {
            return <Home {...props} />
          } else {
            return <ScreenLoader />
          }
        }}
      />
    )
  }
}
```

Now that we have all the pieces, we need to assemble them together, let's create an `App.js` file in `src/components`, with the following contents:

```js
import React from 'react'

import EnvironmentProvider from './EnvironmentProvider'
import StoreProvider from './StoreProvider'
import HomeScreen from './HomeScreen'

const App = () =>
  <StoreProvider>
    <EnvironmentProvider>
      <HomeScreen />
    </EnvironmentProvider>
  </StoreProvider>

export default App
```

The final task is then simply to replace the component import in `index.android.js`, `index.ios.js` and `index.web.js` files from:

```js
import GHViewer from './src/components/HomeScreen'
```

to

```js
import GHViewer from './src/components/App'
```

That's finally it for this chapter! The app is now authenticating the user with GitHub and using the access token to make calls to GitHub's API.

### Related resources

* [Redux' documentation](http://redux.js.org/) - always a great read
* [Redux Persist](https://github.com/rt2zz/redux-persist)
* [Setting up GitHub OAuth apps documentation](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/)



