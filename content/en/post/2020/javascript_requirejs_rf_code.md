---
layout: post
title: Writing Modular Code with RequireJS
category: javascript
date: 2016-10-22 00:00:00
tags:
- javascript
- requirejs
---

# Preface

Early on, JS went straight into script tags. As projects grew, we split into .js files and ordered `<script>` tags to model dependencies — brittle and blocking.

RequireJS (AMD) addresses both: it loads modules asynchronously and manages dependencies explicitly.

## Include RequireJS

```html
<script src="js/require.js" defer async="true"></script>
```

Use `data-main` to specify the entry module:

```html
<script src="js/require.js" data-main="js/home"></script>
```

## Configure

In `home.js`:

```js
require.config({
  baseUrl: '/DomainJS/',
  paths: {
    jquery: 'lib/jquery-1.11.3.min',
    AMUI: 'lib/amazeui.2.7.1.min',
    'jquery.range': 'lib/jquery.range',
    es5: 'lib/es5',
    mapController: 'mapController',
    addToolbar: 'addToolbar'
  },
  shim: {
    addToolbar: { deps: ['jquery'] },
    'jquery.range': { deps: ['jquery'] }
  }
});
```

## Load modules

```js
require(['domready!', 'jquery', 'AMUI', 'mapController', 'city', 'commuteGo'],
function (doc, $, AMUI, mapController, city, commuteGo) {
  city.initAllCityInfo();
  mapController.init();

  $("input[name='locationType']").on('click', mapController.locationMethodOnChange);
  $("input[name='vehicle']").on('click', commuteGo.go);

  // ... omitted ajax and UI code ...
});
```

The first parameter lists dependencies; the callback runs after all load successfully.

## Write AMD modules

No deps:

```js
// helper.js
define(function () {
  var getQueryString = function (name) {
    var reg = new RegExp('(^|&)' + name + '=([^&]*)(&|$)');
    var r = window.location.search.substr(1).match(reg);
    if (r != null) return unescape(r[2]);
    return null;
  };
  return { getQueryString: getQueryString };
});
```

With deps:

```js
// marker.js
define(['mapSignleton', 'city', 'transfer'], function (mapSignleton, city, transfer) {
  // ...
});
```

For more on config and optimization, see:

- https://segmentfault.com/a/1190000002401665
- https://segmentfault.com/a/1190000002403806

