## Creating the React Native project

[React Native](http://facebook.github.io/react-native/) is a framework built by Facebook to create native Android and iOS apps using JavaScript and React.

The first step in this guide will be to create the React-Native project, following the [steps from the official React Native documentation](https://facebook.github.io/react-native/releases/0.42/docs/getting-started.html).

One important thing that will have to be done differently is to install version 0.50 of React Native, as this is the one used ofr this guide. More recent versions might work as well, but you should use 0.50 if you want to be able to follow this guide without issue.

To create the project using React Native 0.50, replace this command from React-Native's getting started guide:

```bash
react-native init AwesomeProject
```

by

```bash
react-native init --version 0.50.4 AwesomeProject
```

Then follow the other steps from the guide, that should end with having your app running on Android and iOS.

### Static types in JavaScript using Flow

Static type are not supported by default in JavaScript, but they can be very useful for large applications or as teams get bigger. Facebook created [Flow](https://flow.org/), a static type checker that is used in many of their projects, including React Native.

This guide will sometimes use Flow types in order to give a better idea of expected function arguments, object payloads and similar simple cases, but will not try to provide a fully type-checked code.

If you're interested in static typing, you may want to check out [TypeScript](https://www.typescriptlang.org/), by Microsoft. This guide uses Flow rather than TypeScript as a convenience to only opt-in for static types in certain cases rather than adapting the entire environment to a different source language.

If you are using the [Nuclide IDE](https://nuclide.io/), chances are it will detect your created project is using Flow and ask for the binary, if so you can add it to your project using the following command:

```bash
yarn add --dev flow-bin@^0.56.0
```

Note that we are using version `0.56` here because it is the one supported by React-Native `0.50`.

Hopefully by now you have successfully setup your React Native project and are able to run you app for Android and iOS, so let's move on to the next chapter in order to add desktop support .

### Related resources

* [React-Native documentation](http://facebook.github.io/react-native/)
* [Create-React-Native-App](https://github.com/react-community/create-react-native-app) - a quick way to get started with React-Native, similar to [Create-React-App](https://github.com/facebookincubator/create-react-app)
* [Flow](https://flow.org/), [TypeScript](https://www.typescriptlang.org/), [an interesting article comparing the two on-boarding processes](http://thejameskyle.com/adopting-flow-and-typescript.html) and [why Reddit chose TypeScript](https://redditblog.com/2017/06/30/why-we-chose-typescript/)



