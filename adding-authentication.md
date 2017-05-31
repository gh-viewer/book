Authenticating users with GitHub

The goal of this chapter is to implement GitHub's OAuth flow in order to get a an access token we'll use in the app to access GitHub's API.

To achieve this, we'll do 3 things: set up an authorisation server, implement a way to store the user's access token in the app, and finally implement the full authentication flow in the app.

Setting up the authorisation server

First, let's setup a server implementing [GitHub's OAuth flow](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/about-authorization-options-for-oauth-apps/#web-application-flow). If you haven't done it already, you'll need to [register your app on GitHub](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/registering-oauth-apps/). This flow can be implemented by any HTTP server using your language and framework of choice, but in this guide we'll use JavaScript with node.

The authentication server implementation is open-source, available in this repository. If you want to set it up quickly, you can [deploy it to Heroku using this link](https://heroku.com/deploy?template=https://github.com/PaulLeCam/gh-viewer-server). It will be free if you choose to use the smallest dyno.

Adding Redux

Setting up the app's flow

