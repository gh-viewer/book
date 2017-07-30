## Additional resources

This guide only serves as an introduction to different libraries that can be used to create a minimum product, but there are many other aspects to consider, such as releasing and updating your app. These additional resources present some other libraries that may be useful in building more advanced apps.

### Code style and linting

Code style does not really matter when being the single developer of a project, but it can become a waste of time when multiple people have different code styles. [Prettier](https://prettier.io/) is a great way to avoid this problem as it supports JS and multiple related languages \(JSON, JSX, GraphQL, CSS...\) and editor integrations \(formatting on save is awesome!\). It is used to format all the code in this guide.

Linting is incredibly useful to avoid common mistakes and weird edge cases, and [ESLint](http://eslint.org/) is pretty much the go-to solution these days. Plugins adding rules for [React](https://github.com/yannickcr/eslint-plugin-react), [React Native](https://github.com/Intellicode/eslint-plugin-react-native), [Flow](https://github.com/gajus/eslint-plugin-flowtype) and so many others make it even more useful.

### Testing

[Jest](https://facebook.github.io/jest/) is a great solution for unit testing, notably because it [supports React Native](https://facebook.github.io/jest/docs/en/tutorial-react-native.html#content). [Enzyme](http://airbnb.io/enzyme/) is a great complementary solution.

For integration testing, Electron [supports Selenium and WebDriver](https://electron.atom.io/docs/tutorial/using-selenium-and-webdriver/). For React Native, there are different tools such as [detox](https://github.com/wix/detox) and [cavy](https://github.com/pixielabs/cavy)[.](https://github.com/pixielabs/cavy)

### Releasing

Using React Native, the Android and iOS apps created are native so the relevant release process applies.

For desktop, there are many options depending on your target platform\(s\), and one great project to handle building and releasing electron apps is [Electron Builder](https://github.com/electron-userland/electron-builder).

### Updating

[CodePush](https://github.com/Microsoft/react-native-code-push) is a great solution to apply over-the-air updates to your React Native app, but unfortunately unavailable for Electron.

Auto-updates can be implemented for Electron [using Electron Builder](https://github.com/electron-userland/electron-builder/wiki/Auto-Update), providing the [Electron Updater](https://www.npmjs.com/package/electron-updater).

### Internationalisation

[React Intl](https://github.com/yahoo/react-intl) is a great library to support internationalisation, and since its [v2.2.0 it supports React Native](https://github.com/yahoo/react-intl/releases/tag/v2.2.0) as well.

### Other awesome resources

* [Awesome React](https://github.com/enaqx/awesome-react)
* [Awesome React Native](https://github.com/jondot/awesome-react-native)
* [Awesome Electron](https://github.com/sindresorhus/awesome-electron)



