---
layout: post
title:  "Dastardly duplication"
date:   2018-04-6 13:10:55 +0000
categories: javascript
author: nspragg
---

One of the most fundamental principles in software development is eliminating duplication. This is often referred to as the [dry principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) and is an underlying notion of many design patterns. Understanding and appropriately applying this principle is a key aspect of creating and maintaining clean codebases. 

In the [moving on from callbacks](https://refactoringbyexample.com/2018/03/moving-on-from-callbacks/) post a `DataFetcher` component for a on-line media store was refactored to use `async/await`. During the refactor it was noted that the fetching module contained functions that were very similar. Reviewing the [source](https://github.com/refactoring-by-example/dastardly-duplication/blob/master/lib/dataFetcher.js) evidences blatant duplication in the fetching and object conversion routines. Both of these will be refactored separately. 

### Remove duplication from the fetch functions:

Repeated functions making HTTP requests:
```js
async function getBooks() {
    const res = await request.get(metadataHost + '/books');
    if (res.statusCode >= 400) {
        throw new Error(`Error: ${res.statusCode}`);
    }
    return res.body;
} 

async function getDvds() {
    const res = await request.get(metadataHost + '/dvds');
    if (res.statusCode >= 400) {
        throw new Error(`Error: ${res.statusCode}`);
    }
    return res.body;
}

// plus another four...
```

Duplication like this is often a consequence of laziness; classic copy and paste programming. Indeed, shortcuts may be taken to save time when under pressure but in practice, this leads to code smells which waste time as the code is difficult to maintain and fragile. To quote Orrin Woodward, "There are many shortcuts to failure, but there are no shortcuts to true success". 

Duplication in the simple fetching functions may seem harmless but problems can so escalate; what if new endpoints need to be added or retry behaviour? As an example, here is some retry logic:

```js
async function getDvds() {
    let retries = 3;
    for (let i = 0; i < retries; i++) {
        try {
            const res = await request.get(metadataHost + '/dvds');
            if (res.statusCode >= 400) {
                throw new Error(`Error: ${res.statusCode}`);
            }
            return res.body;
        } catch (err) {
            if (i === retries - 1) {
                throw err;
            }
        }
    }
}
```

Common sense dictates adding more complex logic to each function is a poor choice and not to mention, in this case, faulty; the above code attempts retries for HTTP 400's (Bad Request), which is probably not desirable. This the bug would have been unnecessarily existed in six functions.  

The solution to avoiding this is simple; remove the duplication by creating a `get` function:
```js
const DEFAULT_RETRIES = 2;

async function get(url, opts) {
    let retries = opts && opts.retries || DEFAULT_RETRIES;

    if (retries === 0) return fetch(url);

    for (let i = 0; i < retries; i++) {
        try {
            const res = await fetch(url);
            return res.body;
        } catch (err) {
            if (i === retries - 1) {
                throw err;
            }
        }
    }
}
```

All the fetching routines were identical except for the url, which is now passed as a parameter. An `opts` object is also passed in to allow for configurable retries. Using an object is more flexible and allows for feature enhancement. For example, adding retry delay or timeouts. 

Now, all fetching logic is defined in one place, reducing code bloat. 

### Remove duplication from the conversion functions:

The model conversion functions are also duplicated:

```js
function toBook(to) {
    const titles = getTitles({
        productType: 'book',
        bookTitle: to.title,
        kind: to.genre,
        author: to.author
    });
    return {
        id: to.id,
        type: 'book',
        title: titles.title,
        subtitle: titles.subtitle,
        kind: to.genre
    }
}

function toDvd(to) {
    const titles = getTitles({
        productType: 'dvd',
        title: to.title,
        kind: to.genre,
        director: to.director,
        year: new Date(to.releaseDate).getFullYear()
    });
    return {
        id: to.id,
        type: 'dvd',
        title: titles.title,
        subtitle: titles.subtitle,
        kind: to.genre
    }
}
```

On first glance refactoring these functions looks really simple, but some there are some subtle differences. Before attempting to refactor, it's imperative to understand the differences. Some analysis indicates:

 * each converter generates titles from the source data by calling `getTitles` See [original post](https://refactoringbyexample.com/2017/01/dealing-with-long-conditionals/) for more details
 * the keys from the source object are used to build the titles object. `genre` and `releaseDate` map to different names.
 * the key mapping above is the same for all converter functions except for `toBook`. This also uses a different `bookTitle` key. 
 * the values from the source object are used to build the titles object. `releaseDate` requires reformatting. 
 * each converter function returns an product object with the following properties: `id`, `type`, `title`, `subtitle`, and `kind` (optional) 

A first pass solution would be to write a single `convert` function that uses a `key map` and `value map` to perform the conversions:

conversion maps:
```js
// all conversions require this mapping
const defaultMapping = {
    genre: 'kind',
    releaseDate: 'year'
};

// additional conversion for books
const bookMapping = {
    defaultMapping,
    title: 'bookTitle'
};

// map of property converter functions (e.g formatting) obj values
const resolvers = {
    releaseDate: (to) => {
        return new Date(to.releaseDate).getFullYear()
    }
};
```

Create a single conversion function:
```js
function convert(type, to) {
    const { id, genre } = to;
    const { title, subtitle } = createTitles(type, to);
    const model = {
        id,
        type,
        title,
        subtitle,
    }
    if (genre) {
        model.kind = genre;
    }

    return model;
}
```

Create helper functions to construct the object for `getTitles` and perform the object lookups:
```js
function resolveKey(key, mapping) {
    const override = mapping[key];
    if (override) return override;
    return key;
}

function resolveValue(key, source) {
    const fn = resolvers[key];
    if (fn) return fn(source);
    return source[key];
}

function createTitles(type, to) {
    const productDetails = {
        productType: type
    };
    for (const key of Object.keys(to)) {
        const mapping = type === 'book' ? bookMapping : defaultMapping;
        let mappedKey = resolveKey(key, mapping);
        productDetails[mappedKey] = resolveValue(key, to);
    }
    return getTitles(productDetails);
}
```

By creating a `convert` function the conversion is defined once rather than repeating for every product type. The variable parts of the conversion have been extracted to objects (for lookup), which can easily be updated for change requests or additional products. 

Eliminating duplication has significantly cleaned the `dataFetcher` module. However, the refactoring of the fetching and conversion functions seem half baked. Should the fetching module have specific knowledge of HTTP requests or object conversion? Is this the right level of abstraction? The answers to this questions will be addressed in a sequent post. 

Thanks for reading.
