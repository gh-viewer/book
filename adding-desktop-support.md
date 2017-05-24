## Adding desktop support

Now that the project is setup for Android and iOS thanks to React-Native, we will add desktop support using [Electron](https://electron.atom.io/). If you are not already familiar with Electron, it is a framework for creating Linux, macOS and Windows applications using Web technologies, in our case React.

In order to use the same APIs for desktop as we do for mobile, we will leverage the React-Native-Web library, that provides the same primitives, and React-Native-Electron, extending React-Native-Web with Electron APIs.

Run the following command to install these dependencies, make sure to specify the same version ranges to make sure there is no incompatibilities in their support:

```bash
yarn add electron@~1.6.8 react-dom@~15.4.2 react-native-web@^0.0.94 react-native-electron@^0.0.13
```

We will also need some more tooling in order to run the desktop app, starting by installing the following dependencies:

```bash
yarn add --dev babel-loader webpack
```



