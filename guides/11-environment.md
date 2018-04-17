## 11-Environment

Environment configuration allows us to include or not include certain things in the build given certain
configuration options. For example, we probably want to not include live reload for production builds,
for this we need to have different environments. When building, we can provide an environment flag option.
So lets go ahead and configure things to support this.

```sh
yarn add --dev broccoli-env@^0.0.1
```

Note, the `broccoli-stew` package also comes with an `env` utility, with a slightly different API, but it doesn't expose
the environment name, and I want to be able to print it to the console, so I'm using the `broccoli-env` package instead.

```js
const Funnel = require("broccoli-funnel");
const Merge = require("broccoli-merge-trees");
const EsLint = require("broccoli-lint-eslint");
const SassLint = require("broccoli-sass-lint");
const CompileSass = require("broccoli-sass-source-maps");
const Rollup = require("broccoli-rollup");
const LiveReload = require('broccoli-livereload');
const babel = require("rollup-plugin-babel");
const nodeResolve = require('rollup-plugin-node-resolve');
const commonjs = require('rollup-plugin-commonjs');
const env = require('broccoli-env').getEnv() || 'development';
const isProduction = env === 'production';

// Status
console.log('Environment: ' + env);

const appRoot = "app";

// Copy HTML file from app root to destination
const html = new Funnel(appRoot, {
  files: ["index.html"],
  annotation: "Index file",
});

// Lint js files
let js = new EsLint(appRoot, {
  persist: true
});

// Compile JS through rollup
js = new Rollup(js, {
  inputFiles: ["**/*.js"],
  annotation: "JS Transformation",
  rollup: {
    input: "app.js",
    output: {
      file: "assets/app.js",
      format: "iife",
      sourcemap: !isProduction,
    },
    plugins: [
      nodeResolve({
        jsnext: true,
        browser: true,
      }),
      commonjs({
        include: 'node_modules/**',
      }),
      babel({
        exclude: "node_modules/**",
      }),
    ],
  }
});

// Lint css files
let css = new SassLint(appRoot + '/styles', {
  disableTestGenerator: true,
});

// Copy CSS file into assets
css = new CompileSass(
  [css],
  "app.scss",
  "assets/app.css",
  {
    sourceMap: !isProduction,
    sourceMapContents: true,
    annotation: "Sass files"
  }
);

// Copy public files into destination
const public = new Funnel("public", {
  annotation: "Public files",
});

// Remove the existing module.exports and replace with:
let tree = new Merge([html, js, css, public], {annotation: "Final output"});

// Include live reaload server
if (!isProduction) {
  tree = new LiveReload(tree, {
    target: 'index.html',
  });
}

module.exports = tree;
```

What we've done here is wrapped the `LiveReload` section in an `env === "development"`, this ensures the `LiveReload`
tree is not included in the build when making a production build.

In order to pass in a different environment, simply add `BROCCOLI_ENV=production` before the build command, e.g.
`BROCCOLI_ENV=production broccoli build dist`. To make this simpler, lets add a new run command in `package.json`:

```json
{
  "scripts": {
    "clean": "rm -rf dist",
    "build": "yarn clean && broccoli build dist",
    "build-prod": "yarn clean && BROCCOLI_ENV=production broccoli build dist",
    "serve": "broccoli serve",
    "debug-build": "yarn clean && node $NODE_DEBUG_OPTION $(which broccoli) build dist",
    "debug-serve": "node $NODE_DEBUG_OPTION $(which broccoli) serve"
  }
}
```

Now, running `yarn build-prod` will build in "production" mode.

Completed Branch: [examples/11-environment](https://github.com/oligriffiths/broccolijs-tutorial/tree/examples/11-environment)

Next: [12-minify](/docs/12-minify.md)
