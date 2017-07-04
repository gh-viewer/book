## Requesting data using Relay

By now we have an application with a shared UI across platforms, but it's only displaying static contents. Let's add some dynamic data by requesting it from GitHub's GraphQL API.

### Getting a personal access token

The first thing we'll need is a personal access token from GitHub. Go to [https://github.com](https://github.com) and authenticate with your account, then in your account menu, open the **Settings** page. In the **Developer settings** section of the sidebar, open the **Personal access tokens** page, then click on the **Generate new token** button.

You will be asked for your password, a descriptions of the token \(put anything you want there\) and the scopes. There is no need to enable any scope at this stage, you can simply click the **Generate token** button at the end of the form. Your token will now appear in clear. Copy it immediately to a safe place, remember a personal access token is like a password, so do not share it or store it in an insecure location. We will later reference to this token using the `[your personal access token]` placeholder, replace it by your token in these places.

> TODO: add screenshots

### Adding dependencies

Now let's add Relay and related tooling to our app:

```bash
yarn add react-relay@^1.1.0 relay-runtime@^1.1.0
```

```bash
yarn add --dev babel-plugin-relay@^1.1.0 graphql-fetch-schema@^0.6.0 relay-compiler@^1.1.0
```

Also add the following `scripts` to your `package.json`:

```js
"relay-compile": "relay-compiler --schema schema.graphql --src src",
"relay-watch": "yarn run relay-compile -- --watch",
"relay-schema": "graphql-fetch-schema --graphql https://api.github.com/graphql?access_token=$ACCESS_TOKEN",
```

### Setting up the tools

Now run:

```bash
ACCESS_TOKEN=[your personal access token] yarn run relay-schema
```

If your access token is valid, this will download GitHub's GraphQL schema to the `schema.graphql` file. In this guide, this is the only time we'll need to run this command, but in case the schema changes later, you can run this command again to update your local schema.

We will also need to update Babel's configuration to use Relay's plugin, so let's change the `.babelrc` file to:

```js
{
  "presets": ["react-native"],
  "plugins": ["relay"]
}
```

### Configuring Relay

By now all the tooling should be setup, so let's add Relay to the application code.

First, let's create an `Environment.js` file in the `src` folder, with the following contents:

```js
import PropTypes from 'prop-types'
import { Environment, Network, RecordSource, Store } from 'relay-runtime'

export const create = () => {
  const fetchQuery = async (operation, variables) => {
    const res = await fetch('https://api.github.com/graphql', {
      method: 'POST',
      headers: {
        authorization: 'bearer [your personal access token]',
        'content-type': 'application/json',
      },
      body: JSON.stringify({
        query: operation.text,
        variables,
      }),
    })
    if (res.ok) {
      return res.json()
    } else {
      throw new Error(res.statusText || 'Request error')
    }
  }

  const network = Network.create(fetchQuery)
  const source = new RecordSource()
  const store = new Store(source)

  return new Environment({ network, store })
}
```

Here, we are creating the [Relay Environment](https://facebook.github.io/relay/docs/relay-environment.html) following Relay's documentation, and providing the `fetchQuery()` function containing the personal access token. This is just a temporary solution used as an example, **do not commit code containing your personal access token**.

Now, let's edit our `HomeScreen.js` file to perform a GraphQL query and render the data, by using [Relay's QueryRenderer](https://facebook.github.io/relay/docs/query-renderer.html):

```js
// @flow

import React from 'react'
import { ActivityIndicator, View } from 'react-native'
import { Button, Icon, Text } from 'react-native-elements'
import { graphql, QueryRenderer } from 'react-relay'

import { create } from '../Environment'

import sharedStyles from './styles'

const environment = create()

type QueryErrorProps = {
  error: Error,
  retry: () => void,
}
const QueryError = ({ error, retry }: QueryErrorProps) => (
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
)

const QueryLoader = () => (
  <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
    <View style={sharedStyles.mainContents}>
      <ActivityIndicator animating size="large" />
      <Text h2 style={sharedStyles.textCenter}>Loading...</Text>
    </View>
  </View>
)

type HomeProps = {
  viewer: {
    login: string,
  },
}
const Home = ({ viewer }: HomeProps) => (
  <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
    <View style={sharedStyles.mainContents}>
      <Icon name="octoface" size={60} type="octicon" />
      <Text h2 style={sharedStyles.textCenter}>
        Welcome, {viewer.login}!
      </Text>
    </View>
  </View>
)

const HomeScreen = () => (
  <QueryRenderer
    environment={environment}
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
        return <HomeScreen {...props} />
      } else {
        return <QueryLoader />
      }
    }}
  />
)

export default HomeScreen
```

There are a few new things introduced here, so let's start from the top of the file, where we define the `// @flow` pragma. As written in a previous chapter, Flow support is optional and is not used in all parts of this guide, but it is convenient in this module to describe the `props` expected by the components.

We then import `graphql` and `QueryRenderer` from `react-relay`. The imported `graphql` function is used by Relay's Babel plugin and compiler to identify and process GraphQL fragments and operations, using [tagged templates literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#Tagged_template_literals). When writing GraphQL query, we must always use it.

The `QueryRenderer` is a React component that handles requesting the data according to the `query` property, and provide the loaded data or request error to the function provided in the `render()` property, using the `environment` created from the `Environment` module.

We also define 3 other components, `QueryError`, `QueryLoader` and `HomeScreen` that are rendered according to the request state.

There is one last step needed for this code to work: we need to compile the GraphQL query. This may not seem relevant at this point because we're only defining a very simple query, but Relay and its compiler work with the idea of GraphQL fragments defining data requirements collocated to the components displaying their data, and therefore usually split across different files. Relay's compiler processes all these files to find fragments used in queries, so that each query can get all the data needed in a single request. This also has the advantage of validating any GraphQL operation defined in the application at compile time, preventing lots of possible runtime errors.

To compile the GraphQL query, run the script we added at the start of this chapter:

```bash
yarn run relay-compile
```

This script will produce a `HomeScreenQuery.graphql.js` file in the `src/components/__generated__` folder. Do not alter this file, it is needed by the application.

When working on the application, likely changing files, fragments and queries, it is usually more convenient to have the compiler run automatically every time the files change. This is provided by Relay's compiler CLI using the `--watch` option, that you can simply run from the scripts we previously added using:

```bash
yarn run relay-watch
```



