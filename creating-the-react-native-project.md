## Creating the React-Native project

> TODO: explanation about React-Native and why we use it

The first step will be to create the React-Native project, following the [steps from the React-Native official documentation](https://facebook.github.io/react-native/releases/0.42/docs/getting-started.html).

One important thing that will have to be done differently is to install version 0.42 of React-Native rather than the latest one. The reason for this is that starting from version 0.43, React-Native uses React 16, but React-Native-Web that we will also use in this project is only compatible with React 15.

To create the project using React-Native 0.42, replace this command from React-Native's getting started guide:

```bash
react-native init AwesomeProject
```

by

```bash
react-native init --version 0.42.3 AwesomeProject
```

Then follow the other steps from the guide, that should end with having your app running on Android and iOS.



### Static types in JavaScript using Flow

Static type are not supported in JavaScript, but they can be very useful for large applications or as teams get bigger. Facebook created [Flow](https://flow.org/), a static type checker that is used in many of their projects, including React-Native.

This guide will sometimes use Flow types in order to give a better idea of expected function argument, object payloads and similar simple cases, but will not try to provide a fully-checked code.

If you're interested in static typing, you may be interested in [TypeScript](https://www.typescriptlang.org/), by Microsoft. This guide uses Flow rather than TypeScript as a convenience to only opt-in for static types in certain cases rather than adapting the entire environment to a different source language.

If you are using the Nuclide IDE, chances are it will detect your created project is using Flow and ask for the binary, if so you can add it to your project using the following command:

```bash
yarn add --dev flow-bin@^0.38.0
```

Note that we are using version `0.38` here because it is the one supported by React-Native `0.42`.





