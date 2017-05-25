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

Then follow the other steps from the guide.

