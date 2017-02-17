---
layout: post
title:  "Dealing with long conditionals"
date:   2017-01-20 09:48:55 +0000
categories: javascript
---
Long conditionals typically evolve as requirements grow. It’s natural, and in some cases, a seemingly a simple task to add another condition to an existing list of conditions. This is particular tempting to do when under pressure and deadlines are closing in. However, a “quick win” is not always the best solution. Following this approach can often lead to more complex code, adversely affecting readability and maintainability. This is likely to result in more brittle software.

In addition, each time there is a new requirement (potentially a new condition) existing code will need to change. This violates the open/close principle.  

The code below highlights the structure of a long conditional:

```js
if (conditionA) {
 console.log('Execute code for condition A');
} else if (conditionB) {
 console.log('Execute code for condition B');
} else if (conditionC) {
 console.log('Execute code for condition C');
} else if (conditionD) {
 console.log('Execute code for condition D');
} else {
 console.log('Execute default');
}
```

Numerous books and blogs demonstrate ways of refactoring small, generic code snippets, but how can this be achieved in a practical context? What is the thought process?

Lets use realistic example of incrementally refactoring a small library for an on-line media store. The library is responsible for generating titles and subtitles for product pages. The library exports a single function, which accepts an object containing product information, such the product of type and author. The function returns another object containing the generated title and subtitle.

For example:

```js
const titles = getTitles({
  productType: 'non-fictional-book',
  bookTitle : 'book title',
  author : 'author',
  year: '1990'
});

console.log(titles.title);
console.log(titles.subtitle);
```
Although not mandatory, it's worth taking a few minutes to review the library. The source code for this library can be found at:
https://github.com/nspragg/online-store

Before making any changes it's imperative to run the tests and ensure all tests are passing. I've ensured this by developing this project using "Test Driven Development" (TDD). Having a comprehensive suite of tests allows refactoring to be performed with greater confidence.

Here is an example of the tests:

```js
describe('Titles', () => {
  describe('when the type is fictional book', () => {
    let titles;
    const bookTitle = 'Charles Dickens';
    const author = 'A Tale of Two Cities';
    const kind = 'fiction';

    beforeEach(() => {
      titles = getTitles({
        productType: BOOK,
        kind,
        bookTitle,
        author
      });
    });

    it('returns the book title as the title', () => {
      assert.equal(titles.title, bookTitle);
    });

    it('returns the author as the subtitle', () => {
      assert.equal(titles.subtitle, author);
    });
  });
});
```

So, lets take a look at the original version of the titles module:

```js
'use strict';

module.exports = (data) => {
  const titles = {};
  const productType = data.productType;
  const kind = data.kind;

  if (productType === 'book' && kind === 'non-fiction') {
    titles.title = data.bookTitle;
    titles.subtitle = `${data.author} (${data.year})`;

  } else if (productType === 'book' && kind === 'travel-guide') {
    titles.title = `${data.publisher}: ${data.city}`;
    titles.subtitle = data.year;

  } else if (productType === 'blu-ray' && kind === 'film') {
    titles.title = `${data.title} (${data.year})`;
    titles.subtitle = data.director;

  } else if (productType === 'blu-ray' && kind === 'box-set') {
    titles.title = `${data.showName} (Season ${data.seasonNumber})`;
    titles.subtitle = data.year;

  } else if (productType === 'vinyl-record') {
    titles.title = data.albumName;
    titles.subtitle = data.artistName;

  } else {
    titles.title = data.bookTitle;
    titles.subtitle = data.author;
  }

  return titles;
};
```

The first observation is that there are several hard coded string literals. These would benefit from only being defined once, as string constants. This may seem like a trivial refactor, but it's often worth starting with a small, simple change. The Chinese philosopher Lao stated, "a journey of a thousand miles begins with a single step".   

**With the string constants, the code now looks like:**

```js
'use strict';

const BOOK = 'book';
const BLU_RAY = 'blu-ray';
const VINYL = 'vinyl-record';

const NON_FICTIONAL_BOOK = 'non-fiction';
const TRAVEL_GUIDE = 'travel-guide';
const FILM = 'film';
const BOXSET = 'box-set';

module.exports = (data) => {
  const titles = {};
  const productType = data.productType;
  const kind = data.kind;

  if (productType === BOOK && kind === NON_FICTIONAL_BOOK) {
    titles.title = data.bookTitle;
    titles.subtitle = `${data.author} (${data.year})`;
  } else if (productType === BOOK && kind === TRAVEL_GUIDE) {
    titles.title = `${data.publisher}: ${data.city}`;
    titles.subtitle = data.year;
  } else if (productType === BLU_RAY && kind === FILM) {
    titles.title = `${data.title} (${data.year})`;
    titles.subtitle = data.director;
  } else if (productType === BLU_RAY && kind === BOXSET) {
    titles.title = `${data.showName} (Season ${data.seasonNumber})`;
    titles.subtitle = data.year;
  } else if (productType === VINYL) {
    titles.title = data.albumName;
    titles.subtitle = data.artistName;
  } else {
    titles.title = data.bookTitle;
    titles.subtitle = data.author;
  }

  return titles;
};
```

Conditionals can often be simplified and more clearly expressed when given a descriptive name. This is typically achieved by extracting the condition and assigning it to a variable or creating a wrapper function (encapsulating the conditional). Either would work in this case. Lets opt for the former.

**Assign conditionals to variables:**

```js
'use strict';

const BOOK = 'book';
const BLU_RAY = 'blu-ray';
const VINYL = 'vinyl-record';

const NON_FICTIONAL_BOOK = 'non-fiction';
const TRAVEL_GUIDE = 'travel-guide';
const FILM = 'film';
const BOXSET = 'box-set';

module.exports = (data) => {
  const titles = {};
  const productType = data.productType;
  const kind = data.kind;

  const isNonFictionalBook = productType === BOOK && kind === NON_FICTIONAL_BOOK;
  const isTravelGuide = productType === BOOK && kind === TRAVEL_GUIDE;
  const isFilm = productType === BLU_RAY && kind === FILM;
  const isBoxset = productType === BLU_RAY && kind === BOXSET;
  const isVinyl = productType === VINYL;

  if (isNonFictionalBook) {
    titles.title = data.bookTitle;
    titles.subtitle = `${data.author} (${data.year})`;
  } else if (isTravelGuide) {
    titles.title = `${data.publisher}: ${data.city}`;
    titles.subtitle = data.year;
  } else if (isFilm) {
    titles.title = `${data.title} (${data.year})`;
    titles.subtitle = data.director;
  } else if (isBoxset) {
    titles.title = `${data.showName} (Season ${data.seasonNumber})`;
    titles.subtitle = data.year;
  } else if (isVinyl) {
    titles.title = data.albumName;
    titles.subtitle = data.artistName;
  } else {
    titles.title = data.bookTitle;
    titles.subtitle = data.author;
  }

  return titles;
};
```

Now that each conditional is easy to read, it's worth focusing attention on the behaviour of each block. It's quite apparent that block creates the title/subtitle and assigns the titles to a shared titles object. Although the titles object is declared and defined using the const keyword, the contents of the object is mutable. By prohibiting mutable state within this function, there is less chance of inadvertently modifying the object.

**Refactor each block to return a titles object inline:**

```js
'use strict';

const BOOK = 'book';
const BLU_RAY = 'blu-ray';
const VINYL = 'vinyl-record';

const NON_FICTIONAL_BOOK = 'non-fiction';
const TRAVEL_GUIDE = 'travel-guide';
const FILM = 'film';
const BOXSET = 'box-set';

module.exports = (data) => {
  const productType = data.productType;
  const kind = data.kind;

  const isNonFictionalBook = productType === BOOK && kind === NON_FICTIONAL_BOOK;
  const isTravelGuide = productType === BOOK && kind === TRAVEL_GUIDE;
  const isFilm = productType === BLU_RAY && kind === FILM;
  const isBoxset = productType === BLU_RAY && kind === BOXSET;
  const isVinyl = productType === VINYL;

  if (isNonFictionalBook) {
    return {
      title: data.bookTitle,
      subtitle: `${data.author} (${data.year})`
    };
  } else if (isTravelGuide) {
    return {
      title: `${data.publisher}: ${data.city}`,
      subtitle: data.year
    };
  } else if (isFilm) {
    return {
      title: `${data.title} (${data.year})`,
      subtitle: data.director
    };
  } else if (isBoxset) {
    return {
      title: `${data.showName} (Season ${data.seasonNumber})`,
      subtitle: data.year
    };
  } else if (isVinyl) {
    return {
      title: data.albumName,
      subtitle: data.artistName
    };
  }

  return {
    title: data.bookTitle,
    subtitle: data.author
  };
};
```

The previous refactor has made each block responsible for building a titles object. In other words, each block has knowledge of title state and behaviour (how titles are built). This is better represented as a titles abstraction.

For example, a Film class:

```js
class Film {
  constructor(data) {
    this._title = data.title;
    this._year = data.year;
    this._director = data.director;
  }

  getTitle() {
    return `${this._title} (${this._year})`;
  }

  getSubtitle() {
    return this._director;
  }
}
```

**Implement all titles abtractions:**

```js
module.exports = (data) => {
 const productType = data.productType;
  const kind = data.kind;

  const isNonFictionalBook = productType === BOOK && kind === NON_FICTIONAL_BOOK;
  const isTravelGuide = productType === BOOK && kind === TRAVEL_GUIDE;
  const isFilm = productType === BLU_RAY && kind === FILM;
  const isBoxset = productType === BLU_RAY && kind === BOXSET;
  const isVinyl = productType === VINYL;

  if (isNonFictionalBook) {
    return new NonFictionalBook(data);

  } else if (isTravelGuide) {
    return new TravelGuide(data);

  } else if (isFilm) {
    return new Film(data);

  } else if (isBoxset) {
    return new Boxset(data);

  } else if (isVinyl) {
    return new Vinyl(data);
  }

  return new FictionalBook(data);
};
```

The titles logic has been improved but the most pertinent refactor is outstanding. That is, this code breaks the open/close principle. Every time there is a new requirement the existing code requirements modification. For example, a new product type 'pc game' was required, a new PCGame class would need to be implemented *and* if/else's would require modification. i.e new condition.    

What's the fix? A pragmatic solution would be to make each title abstraction responsible for knowing if it's a match on the current data. This could be implemented by encapsulating the conditional in a static (class) method. That way, the caller doesn't have to instantiate the title class unless it's a match. For example, an `.isMatch(data)` static

This is also a good opportunity to extract the logic that creates the titles abstraction to a factory function.

**Using a static isMatch method and a factory:**

```js
function getProduct(data) {
  if (NonFictionalBook.isMatch(data)) {
    return new NonFictionalBook(data);

  } else if (TravelGuide.isMatch(data)) {
    return new TravelGuide(data);

  } else if (Film.isMatch(data)) {
    return new Film(data);

  } else if (Boxset.isMatch(data)) {
    return new Boxset(data);

  } else if (Vinyl.isMatch(data)) {
    return new Vinyl(data);
  }

  return new FictionalBook(data);
}

module.exports = (data) => {
  const product = getProduct(data);

  return {
    title: product.getTitle(),
    subtitle: product.getSubtitle()
  };
};
```

The last refactor is only an intermediary step, as we still have a conditional list for the `isMatch` calls. The conditional could be removed by iterating over a list of title abstractions (constructor references) until we have a match. The matching abstraction can then be instantiated and returned by `getProduct`. The default can be returned if a match is not found.    

**Removing the conditional:**

```js
const PRODUCTS = [
  NonFictionalBook,
  TravelGuide,
  BluRayFilm,
  BluRayBoxSet,
  VinylRecord
];

function getProduct(data) {
  const matches = PRODUCTS.filter(product => product.isMatch(data));
  const Product = matches[0] || FictionalBook;

  return new Product(data);
}

module.exports = (data) => {
  const product = getProduct(data);

  return {
    title: product.getTitle(),
    subtitle: product.getSubtitle()
  };
};
```

For brevity the `getProduct` function was retained in the same file. In practice it would be worth considering extracting this to a separate file, completing decoupling the product factory from the logic that invokes it.

In my opinion, the previous refactoring has made some palpable improvements to the titles logic. An effective way to demonstrate that is to add a new requirement in the form of a DVD product type. Previously this would have involved modifying existing code by adding another if statement. Fortunately, after refactoring, there is no long conditional to change, just the additional of a new DVD abstraction.

**Add DVD product type:**

```js
const PRODUCTS = [
  NonFictionalBook,
  TravelGuide,
  BluRayFilm,
  BluRayBoxSet,
  VinylRecord,
  DigitVideoDisc
];

// rest of getProduct remains unchanged
```

You may have observed that, although minimal, the products array requires modification when adding or removing product types. Dynamically loading the product types would eradicate this completely. This may involve writing marginally more complicated code, but should only have to be defined once.  

**Replace the hardcoded products array with dynamically loaded types:**

```js
const Filehound = require('filehound');

const PRODUCTS = Filehound.create()
  .path(path.join(__dirname, './products'))
  .not()
  .match('*fictionalBook*') // exclude the default product
  .findSync()
  .map((product) => require(product));
```

In summary, the refactoring steps performed have simplified and decoupled the code. Hopefully you agree that the titles object more extensible and easier on the eye. Then again, beauty is in the eye of the beholder.

Thanks for reading.
