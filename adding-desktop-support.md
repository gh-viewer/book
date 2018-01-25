## Adding desktop support

Now that the project is setup for Android and iOS thanks to React Native, we will add desktop support using [Electron](https://electron.atom.io/). If you are not already familiar with Electron, it is a framework for creating Linux, macOS and Windows applications using Web technologies, in our case React.

In order to use the same APIs for desktop as we do for mobile, we will leverage the [React Native for Web](https://github.com/necolas/react-native-web) library, that provides the same primitives, and [React Native Electron](https://github.com/PaulLeCam/react-native-electron), extending React Native for Web with Electron APIs.

Let's start by running the following command to install these dependencies. Other versions than the ones provided below may work as well, but these versions should ensure there are no incompatibilities between these libraries:

```bash
yarn add babel-regenerator-runtime@^6.5.0 electron@~1.7.11 react-dom@16.2.0 react-native-web@^0.3.2 react-native-electron@^0.3.0
```

We will also need some more tooling in order to run the desktop app, starting by installing the following dependencies:

```bash
yarn add --dev babel-loader@^7.1.2 webpack@^3.10.0 webpack-dev-server@^2.11.1
```

### The UI entry point

Le'ts create an `index.web.js` file in the root folder, with the following contents:

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

This is essentially the same code a the one generated in `index.js`, with the difference in the last line that we have to run the application by providing the DOM node to render it.

You may notice that we are loading the `AppRegistry` and other APIs from the `react-native` library. This is not a mistake, but on the contrary what allows us to create cross-platform components without having to think much about it. We will see later in this chapter how this is achieved.

### The HTML shell

Now we will need to create the HTML page that will be loaded by Electron, and that will render the application.

Let's create a new `desktop` folder in the root folder, and add an `index.html` file in it with the following contents:

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

The next step is to create the entry point for the desktop application. Electron has the concept of "main process" and "renderer process". The main process will be created as soon as your application is started and is responsible for handling your application lifecycle and windows. These windows will run code in a dedicated renderer process for each window.

In this step, we will implement the script running in the main process, that will create a window in which to render our shared application code-base using React.

In the `desktop` folder, let's add a `main.js` file with the following contents:

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
    path: path.join(__dirname, 'desktop', 'dist'),
    filename: 'bundle.js',
    publicPath: 'dist/',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /react-native-web/,
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
    contentBase: path.join(__dirname, 'desktop'),
    overlay: true,
    port: 8082,
  },
}
```

Most of it is a standard [webpack](https://webpack.js.org/) configuration: it provides the entry point \(here the `index.web.js` file, with `babel-regenerator-runtime` added to support the `async/await` syntax\) and output \(here the `desktop/dist` folder\) and some transformation rules. Here we use `babel-loader` to transform the JS sources.

You may notice that like many usual configurations, we exclude the `node_modules` from the JS transformation. In the next chapter, we will have to change this because React-Native by default compiles the third-party libraries, and therefore plugins authors usually provide sources rather than compiled files, so we will need to provide the same behavior here.

In this configuration, we are also indicating the Electron renderer is the target environment, and the requirement for some node globals, but the main part that makes building a cross-platform application possible is the `resolve` option the configuration: in the `alias` section, we are aliasing any import of `react-native` to `react-native-electron`, and we add support for the `.web.js` extension with a higher resolution priority than `.js`, to provide the same behavior the React-Native packager does with `.android.js` and `.ios.js` files.

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

If everything went well, you should see the desktop application opening a window with the "Welcome to React Native and Electron!" message, and be ready to move on to the next chapter!

### Related resources

* [Electron's quick start guide](https://electron.atom.io/docs/tutorial/quick-start/) and [full documentation](https://electron.atom.io/docs/)
* [Adding DevTools to Electron](https://electron.atom.io/docs/tutorial/devtools-extension/) and the [Electron DevTools Installer](https://github.com/MarshallOfSound/electron-devtools-installer)
* [webpack concepts](https://webpack.js.org/concepts/) and [configuration options](https://webpack.js.org/configuration/)
* [React-Native-Web's getting started](https://github.com/necolas/react-native-web/blob/master/docs/guides/getting-started.md) guide and [known issues](https://github.com/necolas/react-native-web/blob/master/docs/guides/known-issues.md)



