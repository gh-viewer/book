## Adding features

This is the last chapter of this book, the only one not requiring to add and setup other dependencies, but rather to use some of Relay and React Native features to create a minimum product.

The first thing we're going to do it to add pagination to the `HomeScreen`, in order to load more than 10 repositories. To achieve this, we'll use [the `PaginationContainer` provided by Relay](https://facebook.github.io/relay/docs/pagination-container.html).

Here is the updated `HomeScreen.js` code:

```js
// @flow

import React, { Component } from 'react'
import { ScrollView, StyleSheet, View } from 'react-native'
import { Button, Icon, List, ListItem, Text } from 'react-native-elements'
import {
  createFragmentContainer,
  createPaginationContainer,
  graphql,
} from 'react-relay'

import ScreenRenderer from './ScreenRenderer'
import sharedStyles from './styles'

import type { HomeScreen_repository as Repository } from './__generated__/HomeScreen_repository.graphql'
import type { HomeScreen_viewer as Viewer } from './__generated__/HomeScreen_viewer.graphql'

type Variables = { [name: string]: any }
type Disposable = {
  dispose(): void,
}

const PAGE_SIZE = 20

type RepositoryItemProps = {
  navigation: Object,
  repository: Repository,
}

const RepositoryItem = ({ navigation, repository }: RepositoryItemProps) => (
  <ListItem
    containerStyle={styles.itemContainer}
    title={repository.name}
    subtitle={repository.owner.isViewer ? null : repository.owner.login}
    rightIcon={{ name: 'chevron-right', type: 'octicon' }}
    onPress={() => {
      navigation.navigate('Repository', {
        id: repository.id,
        name: repository.nameWithOwner,
      })
    }}
  />
)

const RepositoryItemContainer = createFragmentContainer(RepositoryItem, {
  repository: graphql`
    fragment HomeScreen_repository on Repository {
      id
      name
      nameWithOwner
      owner {
        login
        ... on User {
          isViewer
        }
      }
    }
  `,
})

type HomeProps = {
  navigation: Object,
  relay: {
    hasMore: () => boolean,
    isLoading: () => boolean,
    loadMore: (pageSize: number, cb?: Function) => ?Disposable,
  },
  viewer: Viewer,
}
type HomeState = {
  loading: boolean,
}

class Home extends Component<HomeProps, HomeState> {
  state = {
    loading: false,
  }

  onPressLoadMore = () => {
    if (this.props.relay.hasMore() && !this.state.loading) {
      this.setState({ loading: true })
      this.props.relay.loadMore(PAGE_SIZE, () => {
        this.setState({ loading: false })
      })
    }
  }

  render() {
    const { relay, viewer } = this.props

    if (
      !viewer.repositories ||
      !viewer.repositories.edges ||
      viewer.repositories.edges.length === 0
    ) {
      return (
        <View style={sharedStyles.scene}>
          <View style={sharedStyles.mainContents}>
            <Text h3>No repository!</Text>
          </View>
        </View>
      )
    }

    const rows = viewer.repositories.edges.map(edge => {
      return edge && edge.node ? (
        <RepositoryItemContainer
          key={edge.node.id}
          navigation={this.props.navigation}
          repository={edge.node}
        />
      ) : null
    })

    let footer = null
    if (this.state.loading) {
      footer = (
        <View style={styles.buttonContainer}>
          <Button disabled title="Loading..." />
        </View>
      )
    } else if (relay.hasMore()) {
      footer = (
        <View style={styles.buttonContainer}>
          <Button onPress={this.onPressLoadMore} title="Load more" />
        </View>
      )
    }

    return (
      <ScrollView>
        <List>
          {rows}
          {footer}
        </List>
      </ScrollView>
    )
  }
}

const HomeScreenQuery = graphql`
  query HomeScreenQuery($count: Int!, $cursor: String) {
    viewer {
      ...HomeScreen_viewer
    }
  }
`

const HomeContainer = createPaginationContainer(
  Home,
  {
    viewer: graphql`
      fragment HomeScreen_viewer on User {
        repositories(
          after: $cursor
          first: $count
          orderBy: { field: UPDATED_AT, direction: DESC }
        ) @connection(key: "HomeScreen_repositories") {
          edges {
            node {
              ...HomeScreen_repository
              id
            }
          }
          pageInfo {
            endCursor
            hasNextPage
          }
        }
      }
    `,
  },
  {
    getVariables: (props: HomeProps, paginationVariables: Variables) =>
      paginationVariables,
    query: HomeScreenQuery,
  },
)

export default class HomeScreen extends Component<{
  navigation: Object,
}> {
  static navigationOptions = {
    headerLeft: (
      <View style={sharedStyles.headerLeft}>
        <Icon
          name="repo"
          type="octicon"
          color="white"
          style={sharedStyles.headerIcon}
        />
      </View>
    ),
    title: 'Repositories',
  }

  render() {
    return (
      <ScreenRenderer
        container={HomeContainer}
        navigation={this.props.navigation}
        query={HomeScreenQuery}
        variables={{
          count: PAGE_SIZE,
        }}
      />
    )
  }
}

const styles = StyleSheet.create({
  buttonContainer: {
    marginVertical: 15,
  },
  itemContainer: {
    backgroundColor: 'white',
  },
})
```

The main change is the introduction of [Relay's PaginationContainer](https://facebook.github.io/relay/docs/pagination-container.html), that will be used to paginate over the list of repositories. It provides the functions `hasMore()`, `isLoading()` an `loadMore()` in the `relay` prop, that are used in this updated `Home` component to implement the wanted behavior in the `onPressLoadMore()` method and the footer logic.

We are also going to update the `RepositoryScreen.js` file, with the following contents:

```js
// @flow

import React, { Component } from 'react'
import { Linking, ScrollView, StyleSheet, View } from 'react-native'
import { List, ListItem, Icon, Text } from 'react-native-elements'
import { commitMutation, createFragmentContainer, graphql } from 'react-relay'

import { EnvironmentPropType } from '../Environment'

import ScreenRenderer from './ScreenRenderer'
import sharedStyles from './styles'

import type { RepositoryScreen_repository as RepositoryType } from './__generated__/HomeScreen_viewer.graphql'

const AddStarMutation = graphql`
  mutation RepositoryScreenAddStarMutation($input: AddStarInput!) {
    addStar(input: $input) {
      starrable {
        stargazers {
          totalCount
        }
        viewerHasStarred
      }
    }
  }
`

const RemoveStarMutation = graphql`
  mutation RepositoryScreenRemoveStarMutation($input: RemoveStarInput!) {
    removeStar(input: $input) {
      starrable {
        stargazers {
          totalCount
        }
        viewerHasStarred
      }
    }
  }
`

type Props = {
  navigation: Object,
  repository: RepositoryType,
}

class Repository extends Component<Props> {
  static contextTypes = {
    environment: EnvironmentPropType.isRequired,
  }

  onPressParent = () => {
    const { navigation, repository } = this.props
    navigation.navigate('Repository', {
      id: repository.parent.id,
      name: repository.parent.nameWithOwner,
    })
  }

  onPressStar = () => {
    const { repository } = this.props
    const isAdding = !repository.viewerHasStarred

    commitMutation(this.context.environment, {
      mutation: isAdding ? AddStarMutation : RemoveStarMutation,
      variables: {
        input: {
          starrableId: repository.id,
        },
      },
      optimisticResponse: isAdding
        ? {
            addStar: {
              starrable: {
                __typename: 'Repository',
                stargazers: {
                  totalCount: repository.stargazers.totalCount + 1,
                },
                viewerHasStarred: true,
              },
            },
          }
        : {
            removeStar: {
              starrable: {
                __typename: 'Repository',
                stargazers: {
                  totalCount: repository.stargazers.totalCount - 1,
                },
                viewerHasStarred: false,
              },
            },
          },
    })
  }

  onPressOpenLink = () => {
    const { repository } = this.props
    Linking.openURL(
      `https://github.com/${repository.owner.login}/${repository.name}`,
    )
  }

  render() {
    const { repository } = this.props

    const description = repository.description ? (
      <View style={sharedStyles.mainContents}>
        <Text h5>{repository.description}</Text>
      </View>
    ) : null

    const starCount = repository.stargazers.totalCount

    const items = [
      <ListItem
        hideChevron
        key="owner"
        leftIcon={{
          name:
            repository.owner.__typename === 'User' ? 'person' : 'organization',
          type: 'octicon',
        }}
        title={repository.owner.login}
      />,
    ]
    if (repository.isFork) {
      items.push(
        <ListItem
          key="parent"
          leftIcon={{ name: 'repo-forked', type: 'octicon' }}
          onPress={this.onPressParent}
          rightIcon={{ name: 'chevron-right', type: 'octicon' }}
          title={repository.parent.nameWithOwner}
        />,
      )
    }
    items.push(
      <ListItem
        key="star"
        leftIcon={{ name: 'star', type: 'octicon' }}
        onPress={this.onPressStar}
        rightIcon={{
          name: repository.viewerHasStarred ? 'x' : 'plus',
          type: 'octicon',
        }}
        title={`${starCount} star${starCount > 1 ? 's' : ''}`}
      />,
    )
    items.push(
      <ListItem
        key="open"
        leftIcon={{ name: 'mark-github', type: 'octicon' }}
        onPress={this.onPressOpenLink}
        rightIcon={{ name: 'link-external', type: 'octicon' }}
        title="Open in GitHub"
      />,
    )

    return (
      <ScrollView style={sharedStyles.scene}>
        {description}
        <List containerStyle={styles.listContainer}>{items}</List>
      </ScrollView>
    )
  }
}

const RepositoryContainer = createFragmentContainer(Repository, {
  repository: graphql`
    fragment RepositoryScreen_repository on Repository {
      description
      id
      isFork
      name
      owner {
        __typename
        login
      }
      parent {
        id
        nameWithOwner
      }
      stargazers {
        totalCount
      }
      viewerHasStarred
    }
  `,
})

export default class RepositoryScreen extends Component<{
  navigation: Object,
}> {
  static navigationOptions = ({ navigation }: Object) => ({
    headerLeft: (
      <View style={sharedStyles.headerLeft}>
        <Icon
          name="chevron-left"
          type="octicon"
          color="white"
          underlayColor="black"
          onPress={() => navigation.goBack()}
          style={sharedStyles.headerIcon}
        />
      </View>
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
        navigation={this.props.navigation}
        query={graphql`
          query RepositoryScreenQuery($id: ID!) {
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

const styles = StyleSheet.create({
  listContainer: {
    marginTop: 0,
  },
})
```

Beyond displaying more information, the main change in this module is the introduction of two [Relay mutations](https://facebook.github.io/relay/docs/mutations.html): `addStar` used in the `AddStarMutation` fragment, and `removeStar` used in the `RemoveStartMutation` fragment.

Both of these mutations are used conditionally in the `onPressStar()` method in order to toggle starring the repository for the user, using Relay's `commitMutation()`. By implementing an [optimistic response](https://facebook.github.io/relay/docs/mutations.html#updating-the-client-optimistically), we can ensure the UI is updated immediately by Relay using the payload provided.

That's it for this last chapter! This is still far from a full-feature app, but hopefully a good starting point to implement more of GitHub's APIs as you want!

Don't hesitate to check the additional resources page to discover complementary libraries to build this kind of project.

### Related resources

* [Better List Views in React Native](https://facebook.github.io/react-native/blog/2017/03/13/better-list-views.html) - blog post introducing VirtualizedList and other lists based on it
* Relay's [RefetchContainer](https://facebook.github.io/relay/docs/refetch-container.html) and [PaginationContainer](https://facebook.github.io/relay/docs/pagination-container.html)
* [Relay Mutations](https://facebook.github.io/relay/docs/mutations.html)



