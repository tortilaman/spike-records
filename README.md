# Spike Records

[![npm](http://img.shields.io/npm/v/spike-records.svg?style=flat)](https://badge.fury.io/js/spike-records) [![tests](http://img.shields.io/travis/static-dev/spike-records/master.svg?style=flat)](https://travis-ci.org/static-dev/spike-records) [![dependencies](http://img.shields.io/david/static-dev/spike-records.svg?style=flat)](https://david-dm.org/static-dev/spike-records) [![coverage](http://img.shields.io/coveralls/static-dev/spike-records.svg?style=flat)](https://coveralls.io/github/static-dev/spike-records?branch=master)

remote data -> static templates

## Why should you care?

Static is the best, but sometimes you need to fetch data from a remote source which makes things not so static. Spike Records is a little webpack plugin intended for use with [spike](https://github.com/static-dev/spike) which allows you to make data pulled from a file or url available in your view templates.

It can pull data from the following places:

- A javascript object
- A file containing a javascript object or JSON
- A URL that returns JSON
- A [GraphQL](http://graphql.org) endpoint

## Installation

Install into your project with `npm i spike-records -S`.

Then load it up as a plugin in `app.js` like this:

```javascript
const Records = require('spike-records')
const standard = require('reshape-standard')
const locals = {}

module.exports = {
  reshape: standard({ locals: () => locals }),
  plugins: [new Records({
    addDataTo: locals,
    test: { file: 'data.json' }
  })]
}
```

## Usage

The primary use case for spike-records is to inject local variables into your html templates, although technically it can be used for anything. In the example above, we use the [reshape-standard](https://github.com/reshape/standard) plugin pack to add variables (among other functionality) to your html. Spike's default template also uses `reshape-standard`.

In order to use the results from spike-records, you must pass it an object, which it will put the resolved data on, using the `addDataTo` key. This plugin runs early in spike's compile process, so by the time templates are being compiled, the object will have all the data necessary on it. If you are using the data with other plugins, ensure that spike-records is the first plugin in the array.

I know this is an unusual pattern for a javascript library, but the way it works is quite effective in this particular system, and affords a lot of flexibility and power.

The records plugin accepts an object, and each key in the object (other than `addDataTo`) should contain another object as it's value, with either a `file`, `url`, `data`, or `graphql` key. For example:

```js
const Records = require('spike-records');
let promiseFn = require('./promiseModule'); //for promises
const menuObj = {pepperoniPizza: 10.50, cheesePizza: 8.50, cinaStix: 5.00}
const locals = {}

module.exports = {
  plugins: [new Records({
    addDataTo: locals,
    one: { file: 'data.json' },
    two: { url: 'http://api.carrotcreative.com/staff' },
    three: { data: { foo: 'bar' } },
    four: { data: promiseFn(menuObj) },
    five: {
      graphql: {
        url: 'http://localhost:1234',
        query: 'query { allPosts { title } }',
        variables: 'xxx', // optional
        headers: { authorization: 'Bearer xxx' } // optional
      }
    }
  })]
}
```

Whatever data source you provide, it will be resolved and added to your view templates as `[key]`. So for example, if you were trying to access `three` in your templates, you could access it with `three.foo`, and it would return `'bar'`, as such:

```jade
p {{ three.foo }}
```

Now let's get into some more details for each of the data types.

### File

`file` accepts a file path, either absolute or relative to your [spike](https://github.com/static-dev/spike) project's root. So for the example above, it would resolve to `/path/to/project/data.json`.

### Url

`url` accepts either a string or an object. If provided with a string, it will make a request to the provided url and parse the result as JSON, then return it as a local. If you need to modify this behavior, you can pass in an object instead. The object is passed through directly to [rest.js](https://github.com/cujojs/rest), their docs for acceptable values for this object [can be found here](https://github.com/cujojs/rest/blob/master/docs/interfaces.md#common-request-properties). These options allow modification of the method, headers, params, request entity, etc. and should cover any additional needs.

### Data

The most straightforward of the options, this will just pass the data right through to the locals. Also if you provide an A+ compliant promise for a value, it will be resolved and passed in to the template.

#### Promises

If you're new to promises, this [Google Developers](https://developers.google.com/web/fundamentals/getting-started/primers/promises) article should get you up to speed. The following example is similar to that article. Since we're using Webpack let's keep app.js clean by requiring our promise function from a module we'll create named `promiseModule`. You have the power of Node.js, Webpack, and Spike available to you in this module, but here's a simple example.

##### promiseModule.js
Here's our module, which exports one function that returns a promise that will either be resolved or rejected. This is the module that was required above.
```js
module.exports = function(obj) {
  return new Promise((resolve, reject) => {
    //As an example, let's create a new pizza menu showing their type and price.
    let pizzas = new Array;
    for(var prop in obj) {
      if(${prop}.includes('Pizza')) {
        pizzas.push({
          type: ${prop}.replace("Pizza", ""),
          price: "$".concat(${obj[prop]})
        });
      }
    }
    //Pizza Menu
    if(pizzas.length > 0) {
      resolve(pizzas);
    } else {
      reject(Error("There's no pizza :("));
    }
  });
};
```
##### app.js
This particular promises example requires three changes to app.js. As seen above:
1. The promise function was required with `let promiseFn = require('./promiseModule');`.
2. `menuObj` was defined before it was passed to promiseFn().
3. `promiseFn(menuObj)` was provided as the data key's value.

##### index.sgr
Accessing the json is simple in a loop:
```js
each(loop='pizza of pizzas')
    p {{pizza.type}}, {{pizza.price}}
```


## Additional Options

Alongside any of the data sources above, there are a few additional options you can provide in order to further manipulate the output.

### Transform

If you want to transform the data from your source in any way before injecting it as a local, you can use this option. For example:

```js
const Records = require('spike-records')
const locals = {}

module.exports = {
  plugins: [new Records({
    addDataTo: locals,
    blog: {
      url: 'http://blog.com/api/posts',
      transform: (data) => { return data.response.posts }
    }
  })]
}
```

### Template

Using the template option allows you to write objects returned from records to single page templates. For example, if you are trying to render a blog as static, you might want each `post` returned from the API to be rendered as a single page by itself.

The `template` option is an object with `path` and `output` keys. `path` is an absolute or relative path to a template to be used to render each item, and `output` is a function with the currently iterated item as a parameter, which should return a string representing a path relative to the project root where the single view should be rendered. For example:

```js
const Records = require('spike-records')
const locals = {}

module.exports = {
  plugins: [new Records({
    addDataTo: locals,
    blog: {
      url: 'http://blog.com/api/posts',
      template: {
        path: 'templates/single.sgr',
        output: (post) => { return `posts/${post.slug}.html` }
      }
    }
  })]
}
```

Note that for this feature to work correctly, the data returned from your data source must be an array. If it's not, the plugin will throw an error. If you need to transform the data before it is rendered into templates, you can do so using a `transform` function, as such:

```js
const Records = require('spike-records')
const locals = {}

module.exports = {
  plugins: [new Records({
    addDataTo: locals,
    blog: {
      url: 'http://blog.com/api/posts',
      template: {
        transform: (data) => { return data.response.posts },
        path: 'templates/single.sml',
        output: (post) => { return `posts/${post.slug}.html` }
      }
    }
  })]
}
```

If you use a `transform` function outside of the `template` block, this will still work. The difference is that a `transform` inside the `template` block will only use the transformed data for rendering single templates, whereas the normal `transform` option will alter that data that is injected into your view templates as locals, as well as the single templates.

Inside your template, a local called `item` will be injected, which contains the contents of the item for which the template has been rendered. It will also contain all the other locals injected by spike-records and otherwise, fully transformed by any `transform` functions provided.

## License & Contributing

- Details on the license [can be found here](LICENSE.md)
- Details on running tests and contributing [can be found here](contributing.md)
