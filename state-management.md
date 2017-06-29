## Adding navigation

By now we are able to authenticate the user and retrieve her GitHub information, so we can start creating more screens to display the data. First, let's add the React-Navigation library to handle navigation in the app:

```bash
yarn add react-navigation@1.0.0-beta.11
```

We will also need to update our `webpack.config.js` file by adding an entry to `resolve.alias`, as we previously did to alias `react-native` to `react-native-electron` for the desktop app:

```js
'react-navigation': 'react-navigation/lib-rn/react-navigation.js',
```

We need to create this alias because React-Navigation also has browser support, but in our case we want to use the React-Native APIs.

Now let's create a `Navigation.js` file in `src/components`, with the following contents:

```js
import { StackNavigator } from 'react-navigation'

import HomeScreen from './HomeScreen'

export default StackNavigator(
  {
    Home: { screen: HomeScreen },
  },
  {
    initialRouteName: 'Home',
    navigationOptions: {
      headerStyle: {
        backgroundColor: '#24292e',
      },
      headerTitleStyle: {
        color: 'white',
      },
    },
  },
)
```

And we can replace the `HomeScreen` component in `App.js` by the newly created navigator:

```js
import React from 'react'

import EnvironmentProvider from './EnvironmentProvider'
import StoreProvider from './StoreProvider'
import Navigation from './Navigation'

const App = () =>
  <StoreProvider>
    <EnvironmentProvider>
      <Navigation />
    </EnvironmentProvider>
  </StoreProvider>

export default App
```

We are also going to use Relay's `QueryRenderer` component and the loading logic for more screens, so let's extract the logic from `HomeScreen` to a new file, `ScreenRenderer.js` in `src/components`, with the following contents:

```js
// @flow

import React, { Component, createElement } from 'react'
import { NetInfo, View } from 'react-native'
import { Button, Icon, Text } from 'react-native-elements'
import { QueryRenderer } from 'react-relay'

import { EnvironmentPropType } from '../Environment'

import ScreenLoader from './ScreenLoader'
import { sharedStyles } from './styles'

type QueryErrorProps = {
  error: Error,
  retry: () => void,
}

class QueryError extends Component {
  props: QueryErrorProps
  state: {
    waitingNetwork: boolean,
  }
  connectionListener: ?{
    remove: () => void,
  }

  constructor(props: QueryErrorProps) {
    super(props)

    const waitingNetwork = props.error.message === 'Failed to fetch'
    if (waitingNetwork) {
      this.addConnectionListener()
    }

    this.state = { waitingNetwork }
  }

  addConnectionListener() {
    this.connectionListener = NetInfo.isConnected.addEventListener(
      'change',
      (isConnected: boolean) => {
        if (isConnected) {
          this.props.retry()
        }
      },
    )
  }

  componentWillUnmount() {
    if (this.connectionListener) {
      this.connectionListener.remove()
    }
  }

  render() {
    return this.state.waitingNetwork
      ? <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
          <View style={sharedStyles.mainContents}>
            <Text h3 style={sharedStyles.textCenter}>
              Waiting for network...
            </Text>
          </View>
        </View>
      : <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
          <View style={sharedStyles.mainContents}>
            <Text h3 style={sharedStyles.textCenter}>
              {this.props.error.message || 'Request failed'}
            </Text>
          </View>
          <View style={sharedStyles.bottomContents}>
            <Button onPress={this.props.retry} title="Retry" />
          </View>
        </View>
  }
}

export default class ScreenRenderer extends Component {
  static contextTypes = {
    environment: EnvironmentPropType.isRequired,
  }

  props: {
    container: Component<*, *, *>,
    navigation: Object,
    query: mixed,
    variables?: Object,
  }

  render() {
    return (
      <QueryRenderer
        environment={this.context.environment}
        query={this.props.query}
        variables={this.props.variables}
        render={({ error, props, retry }) => {
          if (error) {
            return <QueryError error={error} retry={retry} />
          } else if (props) {
            return createElement(this.props.container, {
              navigation: this.props.navigation,
              ...props,
            })
          } else {
            return <ScreenLoader />
          }
        }}
      />
    )
  }
}
```

As you can see, we add a bit of logic in the `QueryError` component in order to apply a specific logic when the error is that the query couldn't be fetched. This happens when the app doesn't have network access, so we add a listener that will trigger the query to be retried when the app gets connected. We also display a different UI to the user, without the "retry" button that would fail anyways.

The `ScreenRenderer` in itself simply renders Relay's `QueryRenderer` using the environment it gets from the context, so that components using `ScreenRenderer` don't need to care about it. These components will need to provide the GraphQL `query` and the `container` component to render, as well as the `navigation` if needed by the `container`, and the `variables` used by the `query`.

> TODO: add headerIcon to sharedStyles

Now let's update the `HomeScreen` to use this `ScreenRenderer`:

```js
// @flow

import React, { Component } from 'react'
import { ScrollView, View } from 'react-native'
import { Icon, List, ListItem, Text } from 'react-native-elements'
import { createFragmentContainer, graphql } from 'react-relay'

import ScreenRenderer from './ScreenRenderer'
import { sharedStyles } from './styles'

import type { HomeScreen_repository as Repository } from './__generated__/HomeScreen_repository.graphql'
import type { HomeScreen_viewer as Viewer } from './__generated__/HomeScreen_viewer.graphql'

type RepositoryItemProps = {
  navigation: Object,
  repository: Repository,
}

const RepositoryItem = ({ navigation, repository }: RepositoryItemProps) =>
  <ListItem
    containerStyle={styles.itemContainer}
    title={repository.name}
    subtitle={repository.owner.isViewer ? null : repository.owner.login}
    rightIcon={{ name: 'chevron-right', type: 'octicon' }}
    onPress={() => {
      navigation.navigate('Repository', {
        id: repository.id,
        name: repository.name,
      })
    }}
  />

const RepositoryItemContainer = createFragmentContainer(RepositoryItem, {
  repository: graphql`
    fragment HomeScreen_repository on Repository {
      id
      name
      owner {
        login
        ...on User {
          isViewer
        }
      }
    }
  `,
})

type HomeProps = {
  navigation: Object,
  viewer: Viewer,
}

const Home = ({ navigation, viewer }: HomeProps) => {
  const repositories = viewer.repositories.nodes.length
    ? viewer.repositories.nodes.map(r =>
        <RepositoryItemContainer
          key={r.id}
          navigation={navigation}
          repository={r}
        />,
      )
    : <View style={sharedStyles.mainContents}>
        <Text h3>No repository!</Text>
      </View>

  return <ScrollView style={sharedStyles.scene}>{repositories}</ScrollView>
}

const HomeContainer = createFragmentContainer(Home, {
  viewer: graphql`
    fragment HomeScreen_viewer on User {
      repositories (
        first: 10
        orderBy: {
          field: UPDATED_AT,
          direction: DESC,
        }
      ) {
        nodes {
          ...HomeScreen_repository
          id
        }
      }
    }
  `,
})

export default class HomeScreen extends Component {
  static navigationOptions = {
    title: 'Repositories',
  }

  props: {
    navigation: Object,
  }

  render() {
    return (
      <ScreenRenderer
        container={HomeContainer}
        navigation={this.props.navigation}
        query={graphql`
          query HomeScreenQuery {
            viewer {
              ...HomeScreen_viewer
            }
          }
        `}
      />
    )
  }
}
```

Let's go through a few of the changes, starting with the `HomeScreen` component: we add the static `navigationOptions` object with a `title` property, that will be used by the navigation to display the header, and you can notice the `query` provided to `ScreenRenderer` doesn't define all the data requirements itself anymore, but rather use the `HomeScreen viewer` fragment. This is a fundamental concept of Relay: data requirements are collocated to the components needing the data, which make them very easy to maintain. As you can see in this case, the `HomeScreen` query will contain the `HomeScreen viewer` fragment used by the `HomeContainer` component, and this fragment itself will contain the `HomeScreen repository` fragment used by the `RepositoryItem` component. Relay's compiler will resolve all these fragments so that the query satisfies all these data requirements.

> TODO: run Relay compiler

The `RepositoryItem` has an `onPress` handler that will cause the app to navigate to the `Repository` screen, providing the repository's `id` and `name` values, that we will now use by creating the `RepositoryScreen.js` file in `src/components`, with the following contents:

```js
// @flow

import React, { Component } from 'react'
import { ScrollView, StyleSheet, View } from 'react-native'
import { Icon, Text } from 'react-native-elements'
import { createFragmentContainer, graphql } from 'react-relay'

import ScreenRenderer from './ScreenRenderer'
import { sharedStyles } from './styles'

import type { RepositoryScreen_repository as RepositoryType } from './__generated__/HomeScreen_viewer.graphql'

const Repository = ({ repository }: { repository: RepositoryType }) => {
  return (
    <ScrollView style={sharedStyles.scene}>
      <View style={sharedStyles.mainContents}>
        <Text>
          Repository screen: {repository.owner.login}/{repository.name}
        </Text>
      </View>
    </ScrollView>
  )
}

const RepositoryContainer = createFragmentContainer(Repository, {
  repository: graphql`
    fragment RepositoryScreen_repository on Repository {
      name
      owner {
        login
      }
    }
  `,
})

export default class RepositoryScreen extends Component {
  static navigationOptions = ({ navigation }: Object) => ({
    headerLeft: (
      <Icon
        name="chevron-left"
        type="octicon"
        color="white"
        underlayColor="black"
        onPress={() => navigation.goBack()}
        style={sharedStyles.headerIcon}
      />
    ),
    title: navigation.state.params.name,
  })

  props: {
    navigation: Object,
  }

  render() {
    return (
      <ScreenRenderer
        container={RepositoryContainer}
        query={graphql`
          query RepositoryScreenQuery ($id: ID!) {
            repository: node(id: $id) {
              ...RepositoryScreen_repository
            }
          }
        `}
        variables={{
          id: this.props.navigation.state.params.id,
        }}
      />
    )
  }
}
```

This `RepositoryScreen` is a bit different from the `HomeScreen` because it uses dynamic parameters injected by the navigation: the `title` displayed in the header is the `name` of the repository, and the `query` needs the `id` of the repository node to fetch its data, here injected in the variables.

Last step is simply to add this screen to the `Navigation.js` file, the same way it already supports the `HomeScreen`:

```js
import { StackNavigator } from 'react-navigation'

import HomeScreen from './HomeScreen'
import RepositoryScreen from './RepositoryScreen'

export default StackNavigator(
  {
    Home: { screen: HomeScreen },
    Repository: { screen: RepositoryScreen },
  },
  {
    initialRouteName: 'Home',
    navigationOptions: {
      headerStyle: {
        backgroundColor: '#24292e',
      },
      headerTitleStyle: {
        color: 'white',
      },
    },
  },
)
```

That's it for this chapter! We can now list the 10 repositories last updated and navigate to a separate screen to display details about this repository, as we're going to implement in the next chapter.

