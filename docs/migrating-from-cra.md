# Migrating from create-react-app (CRA) app

This guide is for those that want deploy (in a different server) a CRA-based application as a single-spa child.

## Eject
The first step is to eject your project from react-scripts to gain entire control over app configuration
```
npm run eject
```

## Entrypoint

Create a the single-spa entry point (`src/child.js`, for example) to make your App available as a s-spa child (see https://github.com/CanopyTax/single-spa-react).

```js
import React from 'react';
import ReactDOM from 'react-dom';
import rootComponent from './App.js'; // 
import singleSpaReact from 'single-spa-react';

const reactLifecycles = singleSpaReact({
  React,
  ReactDOM,
  rootComponent,
  domElementGetter: () => document.getElementById('main-content'),
});

export const bootstrap = [
  reactLifecycles.bootstrap,
];

export const mount = [
  reactLifecycles.mount,
];

export const unmount = [
  reactLifecycles.unmount,
];
```

## Webpack
After ejecting, modify the webpack configuration (`config/webpack.config.prod.js`) to:

1. Generate an AMD module.
2. Embed the CSS into your bundle
3. Make assets (images, fonts, etc) available to the hosting-context where your s-spa child-app will be loaded in

```js
...
// change the entrypoint
entry: [require.resolve('./polyfills'), 'src/child.js'], // user the path of your entrypoint file
...
// change output field to something like this
output: {
  libraryTarget: 'amd',
  path: process.cwd() + '/build',
  filename: '[name].js',
  chunkFilename: 'react-[id].chunk.js',
  publicPath: '/build/',
},

...
// remove ExtractTextPlugin from css, you can copy CSS configuration from webpack.config.dev.js
{
  test: /\.css$/,
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        importLoaders: 1,
      },
    },
    {
      loader: require.resolve('postcss-loader'),
      options: {
        // Necessary for external CSS imports to work
        // https://github.com/facebookincubator/create-react-app/issues/2677
        ident: 'postcss',
        plugins: () => [
          require('postcss-flexbugs-fixes'),
          autoprefixer({
            browsers: [
              '>1%',
              'last 4 versions',
              'Firefox ESR',
              'not ie < 9', // React doesn't support IE8 anyway
            ],
            flexbox: 'no-2009',
          }),
        ],
      },
    },
  ],
},
...

// configure a unique publicPath in file-loader plugin
{
    loader: require.resolve('file-loader'),
    exclude: [/\.js$/, /\.html$/, /\.json$/],
    options: {
        name: 'static/media/[name].[hash:8].[ext]',
        publicPath: '/my-child-app', // notice this path must be unique
    },
},
```

## package.json

change the `babel` section in your `package.json`
```json
  "babel": {
    "presets": [
      "react",
      "es2015",
      "stage-2"
    ]
  },
```

## CSS

To avoid CSS duplication and let you to test/dev your child individually, I recommend the following:

1. Import all common CSS (those your root App already imports) at the `src/index.js` (they'll be ignored on production build)
2. Import your child-specific CSS at `src/App.js` or its child components


## Running

After all theses steps, you can build the child-app `npm run build` and serve it `serve -C -p 5000 build`.

## Loading remote child from Root app (NGINX)

That part is focused on nginx but can be adapted for other webservers. The tricky part here is about loading external assets your child app could have.

The problem is that when your child app is load as a AMD module in the root app as following:
```js
singleSpa.declareChildApplication('myChild', () => SystemJS.import('http://11.22.33.44:5000/main.js'), pathPrefix('/child1'));
```
the assets pointed by your child app where in another host and won't be found.

To solve this we will create a proxy in your nginx configuration file, so all request with a pre-defined prefix will be redirect to the appropriated host.

```
...
    server {
        listen 80;
        #serve index.html for 'missing' resources (trick to work with router)
        error_page 404 /index.html;

        location /my-child-app/ {
            # forward requests to child-app server
            proxy_pass http://11.22.33.44:5000/; 
        }

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }`
        
    }
...
```

> notice that the proxy location `my-child-app` must match the `publicPath` configured in your child-app's webpack configuration. 

Finally, you can change your root app to load the child using the proxy:

```js
singleSpa.declareChildApplication('myChild', () => SystemJS.import('/my-child-app/main.js'), pathPrefix('/child1'));
```
