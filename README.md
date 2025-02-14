# @fastify/view

![CI](https://github.com/fastify/point-of-view/workflows/CI/badge.svg)
[![NPM version](https://img.shields.io/npm/v/@fastify/view.svg?style=flat)](https://www.npmjs.com/package/@fastify/view)
[![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg?style=flat)](https://standardjs.com/)

Templates rendering plugin support for Fastify.

`@fastify/view` decorates the reply interface with the `view` method for managing view engines, which can be used to render templates responses.

Currently supports the following templates engines:

- [`ejs`](https://ejs.co/)
- [`nunjucks`](https://mozilla.github.io/nunjucks/)
- [`pug`](https://pugjs.org/api/getting-started.html)
- [`handlebars`](https://handlebarsjs.com/)
- [`mustache`](https://mustache.github.io/)
- [`art-template`](https://aui.github.io/art-template/)
- [`twig`](https://twig.symfony.com/)
- [`liquid`](https://github.com/harttle/liquidjs)
- [`doT`](https://github.com/olado/doT)
- [`eta`](https://eta.js.org)

In `production` mode, `@fastify/view` will heavily cache the templates file and functions, while in `development` will reload every time the template file and function.

_Note: For **Fastify v3 support**, please use point-of-view `5.x` (npm i point-of-view@5)._

_Note that at least Fastify `v2.0.0` is needed._

_Note: [`ejs-mate`](https://github.com/JacksonTian/ejs-mate) support [has been dropped](https://github.com/fastify/point-of-view/pull/157)._

_Note: [`marko`](https://markojs.com/) support has been dropped. Please use [`@marko/fastify`](https://github.com/marko-js/fastify) instead._

#### Benchmarks

The benchmark were run with the files in the `benchmark` folder with the `ejs` engine.
The data has been taken with: `autocannon -c 100 -d 5 -p 10 localhost:3000`

- Express: 8.8k req/sec
- **Fastify**: 15.6k req/sec

## Install

```
npm i @fastify/view
```

<a name="quickstart"></a>

## Quick start

`fastify.register` is used to register @fastify/view. By default, It will decorate the `reply` object with a `view` method that takes at least two arguments:

- the template to be rendered
- the data that should be available to the template during rendering

This example will render the template and provide a variable `text` to be used inside the template:

```js
const fastify = require("fastify")();

fastify.register(require("@fastify/view"), {
  engine: {
    ejs: require("ejs"),
  },
});

fastify.get("/", (req, reply) => {
  reply.view("/templates/index.ejs", { text: "text" });
});

fastify.listen({ port: 3000 }, (err) => {
  if (err) throw err;
  console.log(`server listening on ${fastify.server.address().port}`);
});
```

If your handler function is asynchronous, make sure to return the result - otherwise this will result in an `FST_ERR_PROMISE_NOT_FULFILLED` error:

```js
// This is an async function
fastify.get("/", async (req, reply) => {
  // We are awaiting a function result
  const t = await something();

  // Note the return statement
  return reply.view("/templates/index.ejs", { text: "text" });
});
```

## Configuration

`fastify.register(<engine>, <options>)` accepts an options object.

### Options

- `engine`: The template engine object - pass in the return value of `require('<engine>')`. This option is mandatory.
- `layout`: @fastify/view supports layouts for **EJS**, **Handlebars**, **Eta** and **doT**. This option lets you specify a global layout file to be used when rendering your templates. Settings like `root` or `viewExt` apply as for any other template file. Example: `./templates/layouts/main.hbs`
- `propertyName`: The property that should be used to decorate `reply` and `fastify` - E.g. `reply.view()` and `fastify.view()` where `"view"` is the property name. Default: `"view"`.
- `root`: The root path of your templates folder. The template name or path passed to the render function will be resolved relative to this path. Default: `"./"`.
- `includeViewExtension`: Setting this to `true` will automatically append the default extension for the used template engine **if omitted from the template name** . So instead of `template.hbs`, just `template` can be used. Default: `false`.
- `viewExt`: Let's you override the default extension for a given template engine. This has precedence over `includeViewExtension` and will lead to the same behavior, just with a custom extension. Default `""`. Example: `"handlebars"`.
- `defaultContext`: The template variables defined here will be available to all views. Variables provided on render have precedence and will **override** this if they have the same name. Default: `{}`. Example: `{ siteName: "MyAwesomeSite" }`.
- `maxCache`: In `production` mode, maximum number of templates file and functions caches. Default: `100`. Example: `{ maxCache: 100 }`.

Example:

```js
fastify.register(require("@fastify/view"), {
  engine: {
    handlebars: require("handlebars"),
  },
  root: path.join(__dirname, "views"), // Points to `./views` relative to the current file
  layout: "./templates/template", // Sets the layout to use to `./views/templates/layout.handlebars` relative to the current file.
  viewExt: "handlebars", // Sets the default extension to `.handlebars`
  propertyName: "render", // The template can now be rendered via `reply.render()` and `fastify.render()`
  defaultContext: {
    dev: process.env.NODE_ENV === "development", // Inside your templates, `dev` will be `true` if the expression evaluates to true
  },
  options: {}, // No options passed to handlebars
});
```

## Rendering the template into a variable

The `fastify` object is decorated the same way as `reply` and allows you to just render a view into a variable instead of sending the result back to the browser:

```js
// Promise based, using async/await
const html = await fastify.view("/templates/index.ejs", { text: "text" });

// Callback based
fastify.view("/templates/index.ejs", { text: "text" }, (err, html) => {
  // Handle error
  // Do something with `html`
});
```

## Registering multiple engines

Registering multiple engines with different configurations is supported. They are distinguished via their `propertyName`:

```js
fastify.register(require("@fastify/view"), {
  engine: { ejs: ejs },
  layout: "./templates/layout-mobile.ejs",
  propertyName: "mobile",
});

fastify.register(require("@fastify/view"), {
  engine: { ejs: ejs },
  layout: "./templates/layout-desktop.ejs",
  propertyName: "desktop",
});

fastify.get("/mobile", (req, reply) => {
  // Render using the `mobile` render function
  return reply.mobile("/templates/index.ejs", { text: "text" });
});

fastify.get("/desktop", (req, reply) => {
  // Render using the `desktop` render function
  return reply.desktop("/templates/index.ejs", { text: "text" });
});
```

## Providing a layout on render

@fastify/view supports layouts for **EJS**, **Handlebars**, **Eta** and **doT**.
These engines also support providing a layout on render.

**Please note:** Global layouts and providing layouts on render are mutually exclusive. They can not be mixed.

```js
fastify.get('/', (req, reply) => {
  reply.view('index-for-layout.ejs', data, { layout: 'layout.html' })
})
```

## Setting request-global variables
Sometimes, several templates should have access to the same request-specific variables. E.g. when setting the current username.

If you want to provide data, which will be depended on by a request and available in all views, you have to add property `locals` to `reply` object, like in the example below:

```js
fastify.addHook("preHandler", function (request, reply, done) {
  reply.locals = {
    text: getTextFromRequest(request), // it will be available in all views
  };

  done();
});
```

Properties from `reply.locals` will override those from `defaultContext`, but not from `data` parameter provided to `reply.view(template, data)` function.

## Minifying HTML on render

To utilize [`html-minifier`](https://www.npmjs.com/package/html-minifier) in the rendering process, you can add the option `useHtmlMinifier` with a reference to `html-minifier`,
and the optional `htmlMinifierOptions` option is used to specify the `html-minifier` options:

```js
// get a reference to html-minifier
const minifier = require('html-minifier')
// optionally defined the html-minifier options
const minifierOpts = {
  removeComments: true,
  removeCommentsFromCDATA: true,
  collapseWhitespace: true,
  collapseBooleanAttributes: true,
  removeAttributeQuotes: true,
  removeEmptyAttributes: true
}
// in template engine options configure the use of html-minifier
  options: {
    useHtmlMinifier: minifier,
    htmlMinifierOptions: minifierOpts
  }
```

To filter some paths from minification, you can add the option `pathsToExcludeHtmlMinifier` with list of paths
```js
// get a reference to html-minifier
const minifier = require('html-minifier')
// in options configure the use of html-minifier and set paths to exclude from minification
const options = {
  useHtmlMinifier: minifier,
  pathsToExcludeHtmlMinifier: ['/test']
}

fastify.register(require("@fastify/view"), {
  engine: {
    ejs: require('ejs')
  },
  options
});

// This path is excluded from minification
fastify.get("/test", (req, reply) => {
  reply.view("./template/index.ejs", { text: "text" });
});

```



## Engine-specific settings

<!---
// I don't think this is needed - see https://github.com/fastify/point-of-view/issues/280
## EJS

To use include files please extend your template options as follows:

```js
// get a reference to resolve
const resolve = require('node:path').resolve;
// other code ...
// in template engine options configure how to resolve templates folder
options: {
  filename: resolve("templates");
}
```

and in ejs template files (for example templates/index.ejs) use something like:

```html
<%- include('header.ejs') %>
```

with a path relative to the current page, or an absolute path. Please check this example [here](./templates/layout-with-includes.ejs)
--->

### Mustache

To use partials in mustache you will need to pass the names and paths in the options parameter:

```js
  options: {
    partials: {
      header: 'header.mustache',
      footer: 'footer.mustache'
    }
  }
```

```js
fastify.get('/', (req, reply) => {
  reply.view('./templates/index.mustache', data)
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.mustache', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      const render = mustache.render.bind(mustache, file)
      reply.view(render, data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.mustache', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### Handlebars

To use partials in handlebars you will need to pass the names and paths in the options parameter:

```js
  options: {
    partials: {
      header: 'header.hbs',
      footer: 'footer.hbs'
    }
  }
```

You can specify [compile options](https://handlebarsjs.com/api-reference/compilation.html#handlebars-compile-template-options) as well:

```js
  options: {
    compileOptions: {
      preventIndent: true
    }
  }
```

To access `defaultContext` and `reply.locals` as [`@data` variables](https://handlebarsjs.com/api-reference/data-variables.html):

```js
  options: {
    useDataVariables: true
  }
```

To use layouts in handlebars you will need to pass the `layout` parameter:

```js
fastify.register(require("@fastify/view"), {
  engine: {
    handlebars: require("handlebars"),
  },
  layout: "./templates/layout.hbs",
});

fastify.get("/", (req, reply) => {
  reply.view("./templates/index.hbs", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.hbs', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      const render = handlebars.compile(file)
      reply.view(render, data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.hbs', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### Nunjucks

You can load templates from multiple paths when using the nunjucks engine:

```js
fastify.register(require("@fastify/view"), {
  engine: {
    nunjucks: require("nunjucks"),
  },
  templates: [
    "node_modules/shared-components",
    "views",
  ],
});
```

To configure nunjucks environment after initialization, you can pass callback function to options:

```js
options: {
  onConfigure: (env) => {
    // do whatever you want on nunjucks env
  };
}
```

```js
fastify.get('/', (req, reply) => {
  reply.view('./templates/index.njk', data)
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.njk', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      const render = nunjucks.compile(file)
      reply.view(render, data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.njk', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### Liquid

To configure liquid you need to pass the engine instance as engine option:

```js
const { Liquid } = require("liquidjs");
const path = require('node:path');

const engine = new Liquid({
  root: path.join(__dirname, "templates"),
  extname: ".liquid",
});

fastify.register(require("@fastify/view"), {
  engine: {
    liquid: engine,
  },
});

fastify.get("/", (req, reply) => {
  reply.view("./templates/index.liquid", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.liquid', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      const render = engine.renderFile.bind(engine, './templates/index.liquid')
      reply.view(render, data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.liquid', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### doT

When using [doT](https://github.com/olado/doT) the plugin compiles all templates when the application starts, this way all `.def` files are loaded and
both `.jst` and `.dot` files are loaded as in-memory functions.
This behavior is recommended by the doT team [here](https://github.com/olado/doT#security-considerations).
To make it possible it is necessary to provide a `root` or `templates` option with the path to the template directory.

```js
fastify.register(require("@fastify/view"), {
  engine: {
    dot: require("dot"),
  },
  root: "templates",
  options: {
    destination: "dot-compiled", // path where compiled .jst files are placed (default = 'out')
  },
});

fastify.get("/", (req, reply) => {
  // this works both for .jst and .dot files
  reply.view("index", { text: "text" });
});
```

```js
const d = dot.process({ path: 'templates', destination: 'out' })
fastify.get('/', (req, reply) => {
  reply.view(d.index, data)
})
```

```js
fastify.get('/', (req, reply) => {
  reply.view({ raw: readFileSync('./templates/index.dot'), imports: { def: readFileSync('./templates/index.def') } }, data)
})
```

### eta

```js
const { Eta } = require('eta')
let eta = new Eta()
fastify.register(pointOfView, {
  engine: {
    eta
  },
  templates: 'templates'
})

fastify.get("/", (req, reply) => {
  reply.view("index.eta", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.eta', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view(eta.compile(file), data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.eta', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### ejs

```js
const ejs = require('ejs')
fastify.register(pointOfView, {
  engine: {
    ejs
  },
  templates: 'templates'
})

fastify.get("/", (req, reply) => {
  reply.view("index.ejs", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.ejs', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view(ejs.compile(file), data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.ejs', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### pug

```js
const pug = require('pug')
fastify.register(pointOfView, {
  engine: {
    pug
  }
})


fastify.get("/", (req, reply) => {
  reply.view("index.pug", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.pug', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view(pug.compile(file), data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.pug', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### twig

```js
const twig = require('twig')
fastify.register(pointOfView, {
  engine: {
    twig
  }
})


fastify.get("/", (req, reply) => {
  reply.view("index.twig", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.twig', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view(twig.twig({ data: file }), data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.twig', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

### art

```js
const art = require('art-template')
fastify.register(pointOfView, {
  engine: {
    'art-template': art
  }
})


fastify.get("/", (req, reply) => {
  reply.view("./index.art", { text: "text" });
});
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.art', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view(art.compile({ filename: path.join(__dirname, '..', 'templates', 'index.art') }), data)
    }
  })
})
```

```js
fastify.get('/', (req, reply) => {
  fs.readFile('./templates/index.art', 'utf8', (err, file) => {
    if (err) {
      reply.send(err)
    } else {
      reply.view({ raw: file }, data)
    }
  })
})
```

<!---
// This seems a bit random given that there was no mention of typescript before.
### Typing

Typing parameters from `reply.locals` in `typescript`:

```typescript
interface Locals {
  appVersion: string;
  isAuthorized: boolean;
  user?: {
    id: number;
    login: string;
  };
}

declare module "fastify" {
  interface FastifyReply {
    locals: Partial<Locals> | undefined;
  }
}

app.addHook("onRequest", (request, reply, done) => {
  if (!reply.locals) {
    reply.locals = {};
  }

  reply.locals.isAuthorized = true;
  reply.locals.user = {
    id: 1,
    login: "Admin",
  };
});

app.get("/data", (request, reply) => {
  if (!reply.locals) {
    reply.locals = {};
  }

  // reply.locals.appVersion = 1 // not a valid type
  reply.locals.appVersion = "4.14.0";
  reply.view<{ text: string }>("/index", { text: "Sample data" });
});
```
-->

## Miscellaneous

### Using @fastify/view as a dependency in a fastify-plugin

To require `@fastify/view` as a dependency to a [fastify-plugin](https://github.com/fastify/fastify-plugin), add the name `@fastify/view` to the dependencies array in the [plugin's opts](https://github.com/fastify/fastify-plugin#dependencies).

```js
fastify.register(myViewRendererPlugin, {
  dependencies: ["@fastify/view"],
});
```

### Forcing a cache-flush

To forcefully clear cache when in production mode, call the `view.clearCache()` function.

```js
fastify.view.clearCache();
```

<a name="note"></a>

## Note

By default views are served with the mime type 'text/html; charset=utf-8',
but you can specify a different value using the type function of reply, or by specifying the desired charset in the property 'charset' in the options object given to the plugin.

## Acknowledgements

This project is kindly sponsored by:

- [nearForm](https://nearform.com)
- [LetzDoIt](https://www.letzdoitapp.com/)

## License

Licensed under [MIT](./LICENSE).
