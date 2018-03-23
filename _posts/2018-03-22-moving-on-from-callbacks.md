---
layout: post
title:  "Moving on from callbacks"
date:   2018-03-22 09:48:55 +0000
categories: javascript
author: nspragg
---
Software typically changes over to time to meet new requirements, patch faults and address feeback from users. Programming lanuages are no different. As developers it's imperative to keep our skills at the cutting edge and where appropriate, apply skills on the software we're writing and maintaining. By doing this we can capitalise on the benefits of the languages' evolution. At the time of writing a notable example was asynchronous programming in NodeJs with the introduction of [async functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) and the [await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await) operator. 

Following on from refactoring an online media store ([Dealing with long conditiontionals](https://refactoringbyexample.com/2017/01/dealing-with-long-conditionals/)), this refactor demonstrates migrating from callbacks to async/await. 

The online media store uses an offline component, `DataFetcher` module, to fetch and construct product models, which it writes to a product database. The source code for the `DataFetcher` can be found [here](https://github.com/refactoring-by-example/using-async-functions)

The DataFetcher module is used like this:

```js
 dataFetcher.fetch((err) => {
   if (err) {
     console.error(err)
   }
   // no err indicates the database has been updated successfully
   // with the latest products. 
 });
```

The `.fetch` method does most of the work:

```js
module.exports.fetch = (cb) => {
     async.waterfall([
        (done) => {
            async.parallel({
                book: getBooks,
                dvd: getDvds,
                bluray: getBluerays,
                vinyl: getVinyls,
                blacklist: getBlacklist
            }, done);
        },
        (productSourceData, done) => {
            blacklist = productSourceData.blacklist;
            productSourceData = _.omit(productSourceData, 'blacklist');

            const products = filterByBlacklist(convertProducts(productSourceData), blacklist);
            done(null, products);
        },
        (products, done) => {
            async.map(products, getStocks, (err, stocks) => {
                if (err) return done(err);
                done(null, products, stocks);
            });
        },
        (products, stocks, done) => {
            for (const [i, product] of products.entries()) {
                product.price = stocks[i].price;
                product.quantity = stocks[i].quantity;
            }
            done(null, products);
        }

    ], (err, products) => {
        if (err) return cb(err);
        async.each(products, dao.save, cb);
    });
}
```

Refactoring code like this can be tricky because it contains serveral async `patterns`. To simplify the refactor, it's useful to break down the logic into steps and identify any patterns. 

The `.fetch` using an `async.waterfall`, sequentially executes the following steps (as an array of function references):
 * Requests raw product data in `parallel` (from different API end points) and aggregates the results into an object. Each key maps to an API response
 * Creates the product model objects (eg dvd) from the fetched raw data and removes any blacklisted (banned) items.
 * Fetches stock (inventory) metadata for the products using an asynchronous `map` 
 * Merges stock data with the product models and writes `each` product model to the product database

 Wow that's a lot of responsibility for a single function! Lets break down each step and use async/await where appropriate. 

## Parallel requests
```js
async.parallel({
        book: getBooks,
        dvd: getDvds,
        bluray: getBluerays,
        vinyl: getVinyls,
        blacklist: getBlacklist
}, (err, results) => {
      // process results
});
```

`.parallel` accepts an object which maps a string to a fetching function. The callback either yields with an error (if any requests fail) or an object containing the same keys but mapped to the results of each function invocation. The fetching functions are **very** similar. Here is an example: 

```js
function getBooks(cb) {
    request.get(metadataHost + '/books', (err, res, body) => {
        if (err) return cb(err);
        if (res.statusCode >= 400) {
            return cb(new Error(`Error: ${res.statusCode}`));
        }
        cb(null, body);
    });
}
```

It's worth noting that in practice the first step should be to refactor the corresponding test. However, for reasons of brevity, we'll focus on the main library. 

Let's start by converting the fetching functions to async functions by preceding the function name with `async` and `promisifying` request. Let's refactor `getBooks` as an example:

```js

const request = require('request').defaults({ json: true });
request.get = utils.promisify(request.get);

async function getBooks() {
    return request.get(metadataHost + '/books');
}
```

Next, `await` the response and add http response code error handling:

```js
async function getBooks() {
    const res = await request.get(metadataHost + '/books');
    if (res.statusCode >= 400) {
      return new Error(`Error: ${res.statusCode}`);
    }
    return res;
}
```

Since request now returns a promise, `await` can be used be suspend the function execution until the promise return by `.get` is fulfilled. If successful, the function will resume execution and assign the fulfilment value to `res`. On failure, an error will be thrown. Several points are worth noting; first, the return type of an async function is an promise. In this case, res will be returned to the caller wrapped in a promise. Secondly, `await` can only be invoked from within an `async` function. Thirdly, when awaiting an async function, errors can be caught using a [try catch](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch). This greatly enhances readability and is likely to be natural to many developers.  

Once all the supporting functions have been converted to async functions, the next stage is to execute the requests in parallel and aggregate the responses in a object. Rather than modifying this logic in the `.fetch`, we'll extract it to it's own function:

```js
async function getProductData() {
    const pending = {
        book: getBooks(),
        dvd: getDvds(),
        bluray: getBlurays(),
        vinyl: getVinyls()
    };

    await Promise.all(Object.values(pending));

    const productSourceData = Promise.resolve({})
    return Object.keys(pending).reduce(async (acc, key) => {
        acc = await acc;
        acc[key] = await pending[key];
        return acc;
    }, productSourceData);
```

`getProductData` creates an object of string (productType) to `Promise`. [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) is used to wait for all the requests to fulful or will throw an error. After successful completion, the response body (fullfillment value) is assigned to the corresponding key using `reduce`. Apart from the `async` function declaration, it's important to pass the accumulator as a promise since the return value of an async function is a promise.  

The blacklist was originally included in the `.parallel` call for convenience. By using `await` and `Promise.all`, this is not neccessary are the calls can be made independently. This is a cleaner solution: 

```js
const [blacklist, productSourceData] = await Promise.all([getBlacklist(), getProductData()]);
```

## Create products and apply blacklist filter
The logic to create the product model and filter blacklisted items is synchronous and can be reused:

```js
const products = filterByBlacklist(createProducts(productSourceData), blacklist);
```

## Fetch stock data for the products
```js
(products, done) => {
    async.map(products, (product, cb) => {
                getStocks(product.id, cb);
    }, (err, stocks) => {
    if (err) return done(err);
    done(null, products, stocks);
    });
}
```
The above asynchonously maps over the products fetching the stock data via the product id. This would benefit from being extracted to a named function:

```js
async function getStockData(products) {
    return Promise.all(products.map((product) => getStocks(product.id)));
}
```

`getStockData` will request the stock data in parallel returning a promise, which if successful, will fulfil to an array of stock responses. The resulting code should look intuitive and didn't require any third party libraries.

## Merge product/stock data and update database
```js
 (products, stocks, done) => {
            for (const [i, product] of products.entries()) {
                product.price = stocks[i].price;
                product.quantity = stocks[i].quantity;
            }
            done(null, products);
        }
```

Mering product and stock data is useable but is more clearly defined as a named function:

```js
function merge(products, stocks) {
    for (const [i, product] of products.entries()) {
        product.price = stocks[i].price;
        product.quantity = stocks[i].quantity;
    }
    return products;
}
```  

As the return value of `merge` is an array, we can use the `.forEach` to write to the products database:
```js
return merge(products, stocks).forEach(dao.save);
```

Seems like a convenient one liner! Having said that, it has a serious limitation. What if one or more calls to `dao.save` fail? Most likely, it would result in an unhandled rejected promises, even if the caller uses a try catch:

```js
try {
    merge(products, stocks).forEach(dao.save);
} catch (err) {
    // never gets in catch block
}
```

A combination of `await`, `Promise.all` and `map` can be used to migitate this:

```js
return await Promise.all(merge(products, stocks).map(dao.save));
```

A refactored [`.fetch`](https://github.com/refactoring-by-example/using-async-functions/blob/async-await/lib/dataFetcher.js#L180) in it's entirely would look like this:

```js
module.exports.fetch = async () => {
    const [blacklist, productSourceData] = await Promise.all([getBlacklist(), getProductData()]);

    const products = createProducts(productSourceData);
    const filteredProducts = filterByBlacklist(products, blacklist);
    const stocks = await getStockData(filteredProducts);

    return await Promise.all(merge(products, stocks).map(dao.save));
};
```

This variant of the `.fetch` is noticability more clear and concise. This has largely been achieved by extracting code into named (async) functions and waiting on asynchronous operations, where necessary, using `await`. 

Error handling has been improved as try/catch blocks can be consistently used for synchronous and asynchronous logic. This is arguably more inititive than the error handling convention used with callbacks. 

The use of third party a library for control flow like `waterfall` and other asynchronous patterns are redundant as these can be easily implemented using native `Javascript`. However, some patterns, particularly ones limiting concurrency are more envolved. For more complicated behaviour it may be worth evalating an existing `Promise` libray such as `Bluebird`. 

Thanks for reading.
