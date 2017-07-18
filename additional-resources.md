## Additional resources

This guide only serves as an introduction to different libraries that can be used to create a minimum product, but there are many other aspects to consider, such as releasing and updating your app. These additional resources present some other libraries that may be useful in building more advanced apps.

### Releasing

Using React Native, the Android and iOS apps created are native so the relevant release process applies.

For desktop, there are many options depending on your target platform\(s\), and one great project to handle building and releasing electron apps is [Electron Builder](https://github.com/electron-userland/electron-builder).

### Updating

[CodePush](https://github.com/Microsoft/react-native-code-push) is a great solution to apply over-the-air updates to your React Native app, but unfortunately unavailable for Electron.

Auto-updates can be implemented for Electron [using Electron Builder](https://github.com/electron-userland/electron-builder/wiki/Auto-Update), providing the [Electron Updater](https://www.npmjs.com/package/electron-updater).

### Internationalisation

[React Intl](https://github.com/yahoo/react-intl) is a great library to support internationalisation, and since its [v2.2.0 it supports React Native](https://github.com/yahoo/react-intl/releases/tag/v2.2.0) as well.

