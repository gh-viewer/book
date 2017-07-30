# Creating a cross-platform app using React

This guide aims to create a simple GitHub client for Android and iOS using React Native, and desktop using Electron and React Native for Web.

Both the [client app](https://github.com/gh-viewer/client) and [authentication server](https://github.com/gh-viewer/server) created for this guide are open source.

## Disclaimer

This is not a "definitive guide", "starter template" or any other kind of "best practices", but rather a \(hopefully\) simple introduction to some of the technologies that can be used to build this kind of project with straight-forward code examples, based on the experience I have had building the [Mainframe client](https://mainframe.com/) with the rest of the team, so please don't see it at anything else than one possible way to do things among many others.

Most of the libraries presented in this guide are chosen either because these are the ones used at Mainframe or in other projects, or I was just curious about, so it's not about "one being better than another". It doesn't matter if you prefer TypeScript instead of Flow, MobX instead of Redux or any other library you may want to use, as long as they are the tools to suit you and your team the best, please adapt this guide to your needs!

Many of these technologies are recent and while they provide a great developer experience and allow to iterate quickly, they also have drawbacks that need to be understood. Just because it is possible to build cross-platform applications like this doesn't mean it is the most appropriate in all cases. As always, it depends on your needs.

### Contributions

Please contribute! Whether it is to make the code better, fix some grammar or typos or make some explanations clearer, there is no small contribution!

To make changes to this book, please open a [change request on GitBook](https://www.gitbook.com/book/gh-viewer/creating-a-cross-platform-app-using-react/changes). If you are making changes to the code, please also open a pull request in [the relevant GitHub repository](https://github.com/gh-viewer).

All contributions must be made using the license already used \(CC BY-NC-SA 4.0 for the book contents and MIT for the code\).

## License

This guide is published under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International \(CC BY-NC-SA 4.0\)](https://creativecommons.org/licenses/by-nc-sa/4.0/).

![](http://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by-nc-sa.svg)

