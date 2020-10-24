---
layout: post
title: "Some Random Stuff on JavaScript"
categories: [javascript, node.js, webpack, react]
comments: true
---
{% raw %}

## Inject Environment Variables into Web Pages at Build Time

Environment variables defined in `.env` can be accessed when webpack is generating web pages. After webpack finishes creating all those static files, the concept of environment variable does not exist anymore when we open a purely static web page in a browser. That means, at runtime, without support from a back end, we cannot get those environment variables that we used at build time. For a purely static web page, if we still want to have access to those environment variables at runtime in the browser, a circumvent is to inject those values into the web page at build time. If you are using [Create React App](https://create-react-app.dev/), there is a simple way to achieve that according to its [documentation](https://create-react-app.dev/docs/adding-custom-environment-variables/:#referencing-environment-variables-in-the-html):

```html
<title>%REACT_APP_WEBSITE_NAME%</title>
```

I am using webpack instead of Create React App. In that case, [HtmlWebpackPlugin](https://github.com/jantimon/html-webpack-plugin) is the solution. Actually, there is a Github repo called [jaketrent/html-webpack-template](https://github.com/jaketrent/html-webpack-template) that has already showed the trick.

In the config file of webpack, we add options for HtmlWebpackPlugin as follows:

```js
plugins: [
    ...
    new HtmlWebpackPlugin({
        template: "./src/template.ejs",
        ...
        // inject environment variables into pages at build time
        window: {
            env: {
                client_id: process.env.CLIENT_ID,
                client_secret: process.env.CLIENT_SECRET,
                redirect_uri: process.env.REDIRECT_URL,
                scopes: process.env.SCOPES,
                charts: charts
            }
        }
        ...
    }),
    ...
]
```

Then, in the template (note that we use suffix `.ejs` instead of `.html` for the template so that webpack can pick the correct loader), we add snippet in the `<head>`:

```html
<head>
    ...
    <!-- inject environment variables into pages at build time -->
    <script type="text/javascript">
        <% for (key in htmlWebpackPlugin.options.window) { %>
        window['<%= key %>'] = <%= JSON.stringify(htmlWebpackPlugin.options.window[key]) %>;
        <% } %>
    </script>
    ...
</head>
```
Basically, we inject all environment variables into object `window.env`, which we have access to in the browser. In [jaketrent/html-webpack-template](https://github.com/jaketrent/html-webpack-template), there are also many other notable tricks, like injecting Google Analytics info into the web page using HtmlWebpackPlugin.

## Add/Omit a Property of a JavaScript Object Conditionally in One Line

When creating an object in JavaScript, if whether a property should be in this object is based on some condition, we can create this object utilizing spread syntax as follows:
```js
let name = null;
let person = {
    ...(name && {name: name})
    };
console.log(person); // {}

name = 'Peter';
person = {
    ...(name && {name: name})
    };
console.log(person); // { name: 'Peter' }
```
Whether object `person` has property `name` or not is based on the value of variable `name`. We can also switch between properties conditionally:

```js
let name = null;
let person = {
    ...(name ? {name: name} : {anonymous: true})
};
console.log(person); // { anonymous: true }

name = 'Peter';
person = {
    ...(name ? {name: name} : {anonymous: true})
};
console.log(person); // { name: 'Peter' }
```

{% endraw %}