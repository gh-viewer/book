## Sharing UI components

By now we have a basic setup to render a welcome message on the different platforms we support, but the UI is still defined in 3 different entry files. Let's start sharing our UI components across all platforms.

First, let's add some dependencies to simplify building our UI. We will use [React Native Elements](https://react-native-training.github.io/react-native-elements/) as our framework of choice. There are many other frameworks you can choose from, and you can also decide to build your core UI components yourself, but React Native Elements has the advantage of being simple and working well for our desktop usage as well.

React Native Elements also requires [React Native Vector Icons](https://github.com/oblador/react-native-vector-icons) to display icons, so we will install it as well.

```bash
yarn add react-native-elements@^0.18.4 react-native-vector-icons@^4.4.2
```

### Mobile setup

React Native Vector Icons requires native dependencies and provides fonts that need to be available for the application. To automatically setup these resources, run `react-native link` or [follow these installation instructions](https://github.com/oblador/react-native-vector-icons#installation) if the previous command does not work as expected.

### Desktop setup

In order to makes these libraries work on desktop, we'll first need to add an extra loader for webpack:

```bash
yarn add --dev file-loader@^1.1.5
```

and add update the `module.rules` array of our `webpack.config.js` to compile these libraries and add support for binary assets:

```js
rules: [
  {
    test: /\.js$/,
    exclude: /react-native-web/,
    loader: 'babel-loader',
  },
  {
    test: /\.(png|ttf)$/,
    loader: 'file-loader',
  },
],
```

We also need to make the icon font available to our app. To achieve this, let's add the following code to the `index.web.js` file:

```js
import Octicons from 'react-native-vector-icons/Fonts/Octicons.ttf'

const style = document.createElement('style')
style.type = 'text/css'
style.appendChild(
  document.createTextNode(
    `@font-face { src: url(${Octicons}); font-family: "Octicons"; }`,
  ),
)
document.head.appendChild(style)
```

Here we are importing the `Octicons` font file from `react-native-vector-icons/Fonts` and adding it as a font face in our CSS. This will allow our application to render these icons by using the font.

We are only importing the `Octicons` font as it is the only one we will be using, but importing other fonts works the same way, and it is possible to import multiple ones, as long as they are properly configured.

### Shared styles

Let's start our shared UI by defining some common styles that can be used by various components. Let's create a `styles.js` file inside a new `src/components` folder. We will use this `src` folder to put our application code, and the `components` one to define the React components and associated UI modules, like `styles.js`.

```js
import { Platform, StyleSheet } from 'react-native'

export default StyleSheet.create({
  statusBar: {
    ...Platform.select({
      ios: {
        backgroundColor: '#24292e',
        height: 20,
      },
    }),
  },
  scene: {
    backgroundColor: 'white',
    flex: 1,
  },
  fill: {
    flex: 1,
  },
  mainContents: {
    padding: 10,
  },
  centerContents: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  textCenter: {
    textAlign: 'center',
  },
})
```

Here we are importing the `Platform` and `StyleSheet` modules from React-Native. `StyleSheet` allows us to create styles using a similar API to CSS, as presented in [React Native's documentation](https://facebook.github.io/react-native/releases/0.42/docs/style.html).

You may see examples inserting styles as plain JS objects in components rather than rely on styles defined by `StyleSheet`, and it will work as expected, but one major difference is that using `StyleSheet.create()` compiles the styles and sends them to the native side only once, later using references rather than sending the full object, which is more performant. As a good practice, prefer using `StyleSheet` whenever you can.

The styles we are creating are going to be common to all platforms, but as you can see, we are using `Platform.select()` to make the \`statusBar\` style only defining styles for iOS. `Platform.select()` allows us to define styles per platform, supporting keys being `android`, `ios` and `web`. In this case, we are adding this style on iOS to account for the status bar overlaying our application.

### The first shared component

Now that all the dependencies are in place, let's start using them in our first shared component! Let's create a `HomeScreen.js` file in `src/components` with the following contents:

```js
import React, { Component } from 'react'
import { View } from 'react-native'
import { Icon, Text } from 'react-native-elements'

import sharedStyles from './styles'

export default class HomeScreen extends Component {
  render() {
    return (
      <View style={[sharedStyles.scene, sharedStyles.centerContents]}>
        <View style={sharedStyles.mainContents}>
          <Icon name="octoface" size={60} type="octicon" />
          <Text h2 style={sharedStyles.textCenter}>Welcome!</Text>
        </View>
      </View>
    )
  }
}
```

Finally, let's edit the `index.js` file to have the following contents:

```js
import { AppRegistry } from 'react-native'

import GHViewer from './src/components/HomeScreen'

AppRegistry.registerComponent('GHViewer', () => GHViewer)
```

and the `index.web.js` file can be simplified the same way:

```js
import { AppRegistry } from 'react-native'
import Octicons from 'react-native-vector-icons/Fonts/Octicons.ttf'

import GHViewer from './src/components/HomeScreen'

const style = document.createElement('style')
style.type = 'text/css'
style.appendChild(
  document.createTextNode(
    `@font-face { src: url(${Octicons}); font-family: "Octicons"; }`
  ),
)
document.head.appendChild(style)

AppRegistry.registerComponent('GHViewer', () => GHViewer)
AppRegistry.runApplication('GHViewer', {
  rootTag: document.getElementById('root'),
})
```

Now run `yarn run webpack` to compile the desktop assets. They will be added to the `desktop/dist` folder.

That's it! All platforms are now rendering the `HomeScreen` using shared components and styles! You can try it out by running `yarn start` + `yarn run android` or `yarn run ios` for mobile, and `yarn run webpack-server` + `yarn run desktop` in your terminal.

### Related resources

* [Style](http://facebook.github.io/react-native/releases/0.42/docs/style.html) and [layout](http://facebook.github.io/react-native/releases/0.42/docs/flexbox.html) in React Native
* [Components provided by React Native Elements](https://react-native-training.github.io/react-native-elements/API/HTML_style_headings/)
* [Icons provided by React Native Vector Icons](https://oblador.github.io/react-native-vector-icons/)



