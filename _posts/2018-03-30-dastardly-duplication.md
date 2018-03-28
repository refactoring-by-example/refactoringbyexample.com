---
layout: post
title:  "Dastardly-duplication"
date:   2018-03-30 13:10:55 +0000
categories: javascript
author: nspragg
---

In the [moving on from callbacks](https://refactoringbyexample.com/2018/03/moving-on-from-callbacks/) post a `DataFetcher` component for a on-line media store was refactored to use `async/await`. During the refactor it was noted that the fetching module contained functions that were very similar. Reviewing the [source](https://github.com/refactoring-by-example/dastardly-duplication/blob/master/lib/dataFetcher.js) evidences blatant duplication:

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

Common sense dictates adding more complex logic to each function is a poor choice and not to mention, in this case, faulty;the above code attempts retries for HTTP 400's (Bad Request), which is probably not desirable. 

The solution is simple; remove the duplication:
```js
async function get(url) {
    const res = await request.get(url);
    if (res.statusCode >= 400) {
        throw new Error(`Error: ${res.statusCode}`);
    }
    return res.body;
}
```

All the fetching routines were identical except for the url, which is now passed as a parameter. Now, all fetching logic can be defined in one place, reducing code bloat. 

The model conversion functions are also duplicated, although they have some more suble differences:
```js
```

Thanks for reading.
