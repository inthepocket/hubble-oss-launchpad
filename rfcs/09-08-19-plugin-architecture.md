# Plugin architecture for hubble-app

## Why do we need plugins

We need a way to dynamically load content/code in the Hubble.app so we/community can extend functionality.

Some examples for this are:

- `hubble-plugin-styledictionary` (transform tokens to Style Dictionary)
- `hubble-plugin-theo` (transform tokens to Theo)
- `hubble-plugin-google-cloud` (interact with Google Cloud APIs, like uploading to buckets)
- `hubble-plugin-aws` (interact with AWS APIs, like uploading to S3)

## Plugin flow/architecture

![UML diagram](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/fe4e051a-ca0a-4466-94e3-be54c5c81d1b/Hubble_Plugin_Architecture.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=ASIAT73L2G45A6TWFU7X%2F20190809%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20190809T103457Z&X-Amz-Expires=86400&X-Amz-Security-Token=AgoJb3JpZ2luX2VjEP7%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIQCuKdnZOda5mMvwUI%2FcIHJWz1jL3u1a9w69r1xee6ChmwIgTKdyLdAKLYbmkUvsgRJWvbh%2BFdxnmm7Ptymggi6k2oEq4wMIh%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARAAGgwyNzQ1NjcxNDkzNzAiDNkWzCjImvmKb6n3Ziq3A8jwMoNQNbMQ4V2vdzsClGTsIH3Ro2DtpwOBqHFlbxg%2F0VWKGcvtTKoCby%2BbQXhmsl9yI5fPS8Q4hYEr5bp8fdb4ytIvAaZl%2B3gk0e1CmlkynUnGquTCEzz7U1gaDvZIaoQ2SkgXcbeq5OtGTWNliZbOusLrn7Fnyg%2FI%2BjpaanrBGo3FJwYOW22fgilmqS1%2FDzR50UC0WK6hnx0tdmuRFipQkPrpoYG93E%2BxuUd%2BpeT7C41u9PsHshyJF7PnLm2kuQNOzqBUY%2Fw8wP5A2gr0BamWh0t%2BDfqCj2SxhwyrEf1eW6qbpXRxvBo%2BOWVPgZz5v%2FILUH%2B9FVlzVjVlSEHNnu0VE7hure1lBygchUL%2FwMIJ3tSXz9jBQ%2BsA3TRdyQ%2F%2FLh2yP6f1dZV6DZ3vMT7CyyVsbQr%2FlB2OhfomPVEnY1nCAd3imuVY5tRhmxt%2BPz7lf7js89a9uJlNeVeAmcKwAA4wm3kvaVMBNc3YzbIbahY6ijxRtUDTEaBRo6R0Fp8RchVBxvbQwVcH7TeU2wwyIHJHDAHAm6O3Ou%2BGxndJWRxBSlvRiBvKdid8H3g%2F6RXblOj%2BCv9j%2FG8wxJ606gU6tAHS7VPJjouhchRmcgTjkGzD5SVtxjTGS%2B3hOPAMQ1OSWmLl8ukuSDAVrnunn2WYF70SJMzb96QbGFmJUcbxDvKTa15TowU6RkLLjie3hICqta2th4ik48tHV%2FvCmW%2F6x%2B90ObouPuV9g81zP1K7sBNXG42pvdNU%2FfH8IfWnLS3yd8%2F7lvaRORqFoU9otkGl1xZ8Df8gQFN6ETr2Dv9cDg4hNCMGdx%2B1C1A9d%2FGBAUjvobNwpcQ%3D&X-Amz-Signature=3e07f581df83035a60b719c183d619cdfb3d9f6430d70b17a00902c51035f658&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Hubble_Plugin_Architecture.png%22)

## Plugin structure

Plugins are individual directories contained in the app data subfolder `Plugins` (e.g `~/Library/Application Support/Hubble/Plugins` on macOS).
They can also be tarballs, which would allow is to use the NPM REST API to lookup the URL pointing to an npm module's tarball and download it to the directory, unpacking and loading it.

Plugin directories contain the following files:

- `manifest.json`: Metadata used to visualise the plugin inside the application
  - Defines properties: `icon, description, name`
- `package.json`: Metadata used to get technical informating about the plugin
  - Defines: `plugin version, main module`
- `icon.svg`: A vector icon to show in the Hubble.app
- `<index>.js`: The main entry for the JS module which can be looked up from `package.json`'s `main` property. This module will be loaded by Hubble.app

## Hubble API provided to plugins

```ts
interface Store {
  dispatch: (type: string, ...args: any) => ReduxAction<string, any>;
  getState: () => ReduxStore;
}

interface Hubble {
  registerPreference: (
    prefName: string,
    renderFn: (store: Store) => ReactComponent
  ) => void;

  registerHook: (
    hook: 'export:before' | 'export:after',
    cb: (store: Store) => Promise
  ) => void;
}
```

- API provides the option to register additional UI for preferences where the plugin can be configured.
- API provides the option to register pre-defined hooks which will be triggered at the correct point in time
  - Starting out with export hooks: `export:before` & `export:after`
  - Once the hook is finished, the plugin should return a resolved promise
- The Redux store from Hubble.app is passed to plugins so they can store values inside the Redux store and persistent electron-store of the application
  - Ideally this should be contained to only allow dispatches on Hubble.app's plugins reducer and only get state from that reducer. We do not want plugins to modify the state outside of their sandbox.

A high level pseudo-code example of how this could look:

```ts
const Preferences = ({ store }) => (
  <input
    onSubmit={e => store.dispatch({ data: e.target })}
  />
);

const sideEffect = async () => ...

const CustomPlugin = (Hubble, Electron) => {
  Hubble.registerPreference(
    'Custom Plugin',
    store => <Preferences store={store} />
  );

  Hubble.registerHook('export:before', () => {
    return new Promise(async (resolve, reject) => {
      try {
        const { data } = Hubble.getState();
        const result = await sideEffect();
        return resolve(result);
      } catch (e) {
        return reject(e);
      }
    });
  });
};

export default CustomPlugin;
```

## Pitfalls

- We could also just use npm via its node module or by executing it in a shell using the CLI, but this would be very likely to break for users that do not have node and npm available on their system
- Peer dependencies can not be declared. When we build our Electron application, the base Node runtime and any project dependencies are bundle inside the application and minified into 1 JS bundle. This means that we can not import modules in the plugin and rely on the fact that they are available from the main application. Plugins would be responsible for creating a bundle themselves that bundle those dependencies and the Hubble.app loads that bundle.
  - This means that, since plugins would heavily rely on React + Electron, we would need to pass those modules to the plugin via our Hubble API.
- Possible security flaws where plugins could cause harm to the application. Not identified yet.
- Introduce log level in application so we omit logs emitted from plugins when running in development

### Prior art / resources

- [davideicardi/live-plugin-manager](https://github.com/davideicardi/live-plugin-manager)
- [kanekotic/electron-plugin](https://github.com/kanekotic/electron-plugin#readme)
