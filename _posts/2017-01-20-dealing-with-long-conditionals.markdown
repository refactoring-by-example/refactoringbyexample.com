---
layout: post
title:  "Dealing with long conditionals"
date:   2017-01-20 09:48:55 +0000
categories: javascript
---
Long conditionals typically evolve as requirements grow. It’s natural, and in some cases, a seemingly a simple task to add another condition to an existing list of conditions. This is particular tempting to do when under pressure and deadlines are closing in. However, a “quick win” is not always the best solution. Following this approach can often lead to more complex code, adversely affecting readability and maintainability. This is likely to result in more brittle software.

In addition, each time there is a new requirement (potentially a new condition) existing code will need to change.  This violates the open/close principle.  

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
