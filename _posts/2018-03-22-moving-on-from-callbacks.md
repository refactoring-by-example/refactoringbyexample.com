---
layout: post
title:  "Moving on from callbacks"
date:   2018-03-22 09:48:55 +0000
categories: javascript
author: nspragg
---
Software typically changes over to time to meet new requirements, patch faults and address feeback from the users. Programming lanuages are no different. As developers it's imperative to keep our skills at the cutting edge and where appropriate, apply skills on the software we're writing and maintaining. By doing this we can capitalise on the benefits of the languages' evolution. At the time of writing a notable example was asynchronous in NodeJs with the introduction of [async functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) and the [await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await) operator. 

Following on from refactoring an online media store ([Dealing with long conditiontionals](https://refactoringbyexample.com/2017/01/dealing-with-long-conditionals/)), this refactor demonstrates migrating from callbacks to async/await.   

The online media store uses an offline component, `DataFetcher`, to fetch and construct product models, which it writes to a product database. 

For example: module usage 

```js
 dataFetcher.fetch((err) => {
   if (err) {
     console.error(err)
   }
   // no err indicates success
 });
```

For example: the `.fetch` method that does most of the work

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
                product['price'] = stocks[i].price;
                product['quantity'] = stocks[i].quantity;
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

The `.fetch` does the following:
 * Sequentially executes an array functions, passing data from one to another. i.e `waterfall`
 * Makes calls to multiple endpoints in `parallel` and aggregates the results into an results object. Each key mapping to the results of an API call 
 * Converts raw data into the products model objects (eg dvd) and removes any blaclisted items.
 * Fetches stock metadata from the products using an asynchronous `map` 
 * Merges stock data with the product and writes `each` product model to the product database

 Wow that's alot of responsibility for a single function! Lets break down each step and use async/await where appropriate. 

### Parallel requests
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

The first step is to convert the functions to async functions by preceding the function name with `async` and `promisifying` request:

```js

const request = require('request').defaults({ json: true });
request.get = utils.promisify(request.get);

async function getBooks() {
    return request.get(metadataHost + '/books');
}
```

Next, await the response and add error handling logic:

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

The blacklist was originally included in the `.parallel` for convenience. Using async/await, this is not nessessary and  can be called separately. Similarly, `Promise.all` is used to sychronise the parallel requests:

```js
let blacklist = getBlacklist();
let productSourceData = getProductData();
[blacklist, productSourceData] = await Promise.all([blacklist, productSourceData]);
```

The logic to create the product model and filter blacklisted items is synchronous and can be reused:

```js
const products = filterByBlacklist(createProducts(productSourceData), blacklist);
```

Next the stock data can be fetched by mapping over the products:
```js
async function getStockData(products) {
    return Promise.all(products.map((product) => getStocks(product.id)));
}
```

`getStockData` will request the stock data in parallel returning a promise, which if successful, will fulfil to an array of stock responses. The resulting code should look intuitive and didn't require any third party libraries.

Finally, the product and stock information can be merged. Makes sense to extract this logic:
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

A refactored `.fetch` in it's entirely would look like this:

```js
module.exports.fetch = async () => {
    let blacklist = getBlacklist();
    let productSourceData = getProductData();
    [blacklist, productSourceData] = await Promise.all([blacklist, productSourceData]);

    const products = filterByBlacklist(createProducts(productSourceData), blacklist);
    const stocks = await getStockData(products);

    return await Promise.all(merge(products, stocks).map(dao.save));
};
```

By utilising a combination of more recent language features (primarily async/await) and function extraction, the implementation of `.fetch` is noticably more clear, concise and doesn't require external dependancies. 

Thanks for reading.
