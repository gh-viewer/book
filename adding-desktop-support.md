## Adding desktop support

Now that the project is setup for Android and iOS thanks to React-Native, we will add desktop support using [Electron](https://electron.atom.io/). If you are not already familiar with Electron, it is a framework for creating Linux, macOS and Windows applications using Web technologies, in our case React.

In order to use the same APIs for desktop as we do for mobile, we will leverage the React-Native-Web library, that provides the same primitives, and React-Native-Electron, extending React-Native-Web with Electron APIs.

Run the following command to install these dependencies, make sure to specify the same version ranges to make sure there is no incompatibilities in their support:

```bash
yarn add babel-regenerator-runtime@^6.5.0 electron@~1.6.11 react-dom@~15.4.2 react-native-web@^0.0.104 react-native-electron@^0.0.16
```

We will also need some more tooling in order to run the desktop app, starting by installing the following dependencies:

```bash
yarn add --dev babel-loader@^7.1.0 webpack@^2.6.1 webpack-dev-server@^2.5.0
```

### The UI entry point

Create an `index.web.js` file in the root folder, with the following contents:

```js
import React, { Component } from 'react'
import { AppRegistry, StyleSheet, Text, View } from 'react-native'

export default class GHViewer extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native and Electron!
        </Text>
      </View>
    )
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
})

AppRegistry.registerComponent('GHViewer', () => GHViewer)
AppRegistry.runApplication('GHViewer', {
  rootTag: document.getElementById('root'),
})
```

This is essentially the same as what happens in `index.android.js` and `index.ios.js`, with the difference in the last line that we have to run the application by providing the DOM node to render it.

You may notice that we are loading the `AppRegistry` and other APIs from the `react-native` library. This is not a mistake, but on the contrary what allows us to create cross-platform components without having to think much about it. We will see later in this page how this is achieved.

### The HTML shell

Now we will need to create the HTML page that will be loaded by Electron, and that will render the application.

Create a new `desktop` folder in the root folder, and add an `index.html` file in it with the following contents:

```
<!DOCTYPE html>
<html>
  <head>
    <title>GH Viewer</title>
    <meta charset="utf-8">
  </head>
  <body>
    <div id="root"></div>
    <script src="http://localhost:8082/dist/bundle.js"></script>
  </body>
</html>
```

As you can see it is very simple. We provide a title that Electron will use for the application title, the `root` div where the React components will be rendered, and a script loading our application from a local server, that we will setup later in this page.

### The application entry point

> TODO: explanations about Electron main and renderer processes

In the `desktop` folder, add a `main.js` file with the following contents:

```js
const { app, BrowserWindow } = require('electron')

let mainWindow = null

const createWindow = () => {
  mainWindow = new BrowserWindow({
    minWidth: 300,
    minHeight: 400,
    maxWidth: 400,
    width: 300,
    height: 600,
    show: false,
  })

  mainWindow.on('closed', () => {
    mainWindow = null
  })
  mainWindow.once('ready-to-show', () => {
    mainWindow.show()
  })

  mainWindow.loadURL(`file://${__dirname}/index.html`)
}

app.on('ready', () => {
  createWindow()
})
```

There are a few things going on here, as we need to create the window containing our application and load it, in our case the `index.html` file we previously created. This is done in Electron by creating an instance of `BrowserWindow` and loading the wanted URL for it.

When creating the `BrowserWindow`, we set the initial dimensions and some constraints as our UI will be mostly vertical to match the mobile experience. Setting the `show` parameter to `false` prevents the window from appearing immediately, instead we will wait for the application to be ready. This is a good practice to provide a better user experience of not displaying a white screen before the contents are rendered.

We also need to wait for the `ready` event from the application to create this window, as it won't be possible before this event is fired.

### The build configuration

Back in the root folder, let's create a `webpack.config.js` file with the following contents:

```js
const path = require('path')

module.exports = {
  entry: ['babel-regenerator-runtime', './index.web.js'],
  output: {
    path: path.resolve(__dirname, 'desktop', 'dist'),
    filename: 'bundle.js',
    publicPath: 'dist/',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader',
      },
    ],
  },
  target: 'electron-renderer',
  resolve: {
    alias: {
      'react-native': 'react-native-electron',
    },
    extensions: ['.web.js', '.js', '.json'],
  },
  node: {
    __filename: true,
    __dirname: true,
  },
  devServer: {
    contentBase: path.resolve(__dirname, 'desktop'),
    overlay: true,
    port: 8082,
  },
}
```

Most of it is a standard webpack configuration: it provides the entry point \(here the `index.web.js` file, with `babel-regenerator-runtime` added to support the `async/await` syntax\) and output \(here the `desktop/dist` folder\) and some transformation rules. Here we use `babel-loader` to transform the JS sources.

You may notice that unlike many usual configurations, we do not exclude the `node_modules` from the JS transformation. This is because React-Native by default compiles the third-party libraries, and therefore plugins authors usually provide sources rather than compiled files. We need to provide the same behavior here.

In this configuration, we are also indicating the Electron renderer is the target environment, and the requirement for some node globals, but the main part that makes building a cross-platform application possible is the `resolve` part of the configuration: in the `alias` section, we are aliasing any import of `react-native` to `react-native-electron`, and we add support for the `.web.js` extension with a higher resolution priority than `.js`, to provide the same behavior the React-Native packager does with `.android.js` and `.ios.js` files.

Finally, we setup the `devServer` configuration to serve the contents dynamically.

### Useful scripts

In the `package.json`, let's add some entries to the `scripts` to support some commands we'll use to compile and run the app:

```js
"android": "react-native run-android",
"ios": "react-native run-ios",
"desktop": "electron ./desktop/main.js",
"webpack": "webpack --progress",
"webpack-server": "webpack-dev-server --hot --progress",
```

You should already have the `start` and `test` scripts added during the React-Native installation.

Ready to try this out? Run `yarn run webpack-server` in one terminal to start the server that will compile the JavaScript sources for desktop, and run `yarn run desktop` in another terminal to start the Electron process for the application.

If the application window does not open, don't worry it is probably simply waiting for the server to be ready, you can check the progress in the terminal running `webpack-dev-server`, and the application window should appear once it reaches 100%.

