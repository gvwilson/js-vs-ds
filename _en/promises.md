---
permalink: "/en/promises/"
title: "Promises"
questions:
- "How does JavaScript implement delayed computation?"
- "Is there an easier way to handle delayed computation?"
- "How can a program wait for many promises to complete, or for one promise in a set to complete?"
- "Is there an even easier way to manage all of this?"
keypoints:
- "JavaScript keeps an execution queue for delayed computations."
- "Use promises to manage delayed computation instead of raw callbacks."
- "Use a callback with two arguments to handle successful completion (resolve) and unsuccessful completion (reject) of a promise."
- "Use `then` to express the next step after successful completion and `catch` to handle errors."
- "Use `Promise.all` to wait for all promises in a list to complete and `Promise.race` to wait for the first promise in a set to complete."
- "Use `await` to wait for the result of a computation."
- "Mark functions that can be waited on with `async`."
---

-   Callbacks quickly become complicated because of:
    -   Nesting: a delayed calculation that needs the result of a delayed calculation that needs...
    -   Error handling: who notices and takes care of errors?
    -   This is often a problem in real life too
-   Example: turn a bunch of CSV files into HTML pages
    -   Inputs: the name of a directory that contains one or more CSV files and the name of an output directory
    -   Result: output directory is replaced by one that has one HTML page per CSV file *and* an index page
-   Can do this with synchronous operations, but that's not the JavaScript Way [tm]
    -   By which we mean, doing it that way doesn't introduce you to tools we're going to need later
-   Try doing it with callbacks
    -   Can't create the output directory until the existing one has been emptied
    -   Can't generate the HTML pages until the output directory has been re-created
    -   Can't generate the index page until the CSV files have been processed
-   Instead of callbacks, use [promises](#g:promise)
-   And then use `async` and `await` to make things even easier
-   Three mechanisms because people learn as they go along, but the simple high-level ideas often don't make sense unless you understand how they work
    -   This is also often a problem in real life

## The Execution Queue {#s:promises-queue}

-   FIXME: explain execution queue <https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/>
-   Most functions execute in order

```js
[1000, 1500, 500].forEach(t => {
  console.log(t)
})
```
{: title=promises/not-callbacks-alone.js}
```text
1000
1500
500
```

-   A handful of built-in functions delay execution
    -   `setTimeout` is probably the most widely used

```js
[1000, 1500, 500].forEach(t => {
  console.log(`about to setTimeout for ${t}`)
  setTimeout(() => {console.log(`inside timer handler for ${t}`)}, t)
})
```
{: title=promises/callbacks-with-timeouts.js}
```text
about to setTimeout for 1000
about to setTimeout for 1500
about to setTimeout for 500
inside timer handler for 500
inside timer handler for 1000
inside timer handler for 1500
```

-   Setting a timeout of zero has the effect of deferring execution without delay

```js
[1000, 1500, 500].forEach(t => {
  console.log(`about to setTimeout for ${t}`)
  setTimeout(() => {console.log(`inside timer handler for ${t}`)}, 0)
})
```
{: title=promises/callbacks-with-zero-timeouts.js}
```text
about to setTimeout for 1000
about to setTimeout for 1500
about to setTimeout for 500
inside timer handler for 1000
inside timer handler for 1500
inside timer handler for 500
```

-   We can use this to build a generic non-blocking function

```js
const nonBlocking = (callback) => {
  setTimeout(callback, 0)
}

[1000, 1500, 500].forEach(t => {
  console.log(`about to do nonBlocking for ${t}`)
  nonBlocking(() => console.log(`inside callback for ${t}`))
})
```
{: title=promises/non-blocking.js}
```text
about to do nonBlocking for 1000
about to do nonBlocking for 1500
about to do nonBlocking for 500
inside callback for 1000
inside callback for 1500
inside callback for 500
```

-   Why bother?
    -   Because we may want to give something else a chance to run
-   Node provides `setImmediate` to do this for us
    -   And also `process.nextTick`, which doesn't do quite the same thing

```js
[1000, 1500, 500].forEach(t => {
  console.log(`about to do setImmediate for ${t}`)
  setImmediate(() => console.log(`inside immediate handler for ${t}`))
})
```
{: title=promises/set-immediate.js}
```text
about to do setImmediate for 1000
about to do setImmediate for 1500
about to do setImmediate for 500
inside immediate handler for 1000
inside immediate handler for 1500
inside immediate handler for 500
```

-   Can see from first `setTimeout` example above that execution can happen in a different order than called
    -   Function call with `t = 500` happens first, despite being last iteration of loop
-   This asynchronous execution is helpful but confusing at first

## Promises {#s:promises-promises}

-   Consider situation where reading a file
-   Accessing local storage takes a (relatively) long time
    -   Worse when reading data over the web
-   JS doesn't wait for file to load before continuing execution
-   Moves onto other tasks and comes back later
-   With promises we can queue up code to execute once a task is finished
-   E.g., finding size of a local file with `fs-extra.stat`

```js
const fs = require('fs-extra')
fs.stat('moby-dick.txt').then((stats) => console.log(stats.size))
```

-   `fs-extra.stat` produces some statistics about the file
-   The argument to `then` is a function that is called after `fs-extra.stat` has finished profiling the file
-   `fs-extra.stat` returns a Promise object
-   To understand them better, let's create our own Promise to fetch a datafile

```js
var prom = new Promise((resolve, reject) => {
    fetch("https://api.nasa.gov/neo/rest/v1/feed?api_key=DEMO_KEY&start_date=2018-08-20")
    .then((response) => {
        if (response.status === 200) {
            resolve("fetched page successfully")
        }
    })
}).then((message) => console.log(message))
// fetched page successfully
```

-   Construct a new Promise object, providing a function
-   That function takes two arguments: resolve and reject
-   Call `resolve` inside the function body, to determine the value returned if everything completed successfully
-   All promises have a `then` method that takes this value returned by resolve as an input argument
-   Use `reject` to define returned value when promise fails
-   And `catch` to process that returned value (usually an `Error` object)

```js
var prom = new Promise((resolve, reject) => {
    fetch("https://api.nasa.gov/neo/rest/v1/feed?api_key=DEMO_KEY&start_date=20-08-2018")
    .then((response) => {
        if (response.status === 200) {
            resolve("fetched page successfully")
        }
        else {
            reject(Error("there was a problem. got HTTP status code ${response.status}"))
        }
    })
}).then((message) => console.log(message))
.catch((error) => console.log(error.message))
```

-   Can also provide two functions to `then`, where second argument will process reject value
-   Going back to `fs-exta.stat` example above, what if we want to process multiple files e.g. calculate total size?
-   Might think to write a loop

```js
const fs = require("fs-extra")

var total_size = 0
var files = ["jane-eyre.txt", "moby-dick.txt", "life-of-frederick-douglass.txt"]
for (let fileName of files) {
    fs.stat(fileName).then((stats) => {
        total_size += stats.size
    })
}
console.log(total_size)
```

-   Danger: `fs.stat` in each iteration is executed asynchronously
-   Might try chaining Promises together, to ensure that each executes only after the last resolved

```js
const fs = require("fs-extra")

var total_size = 0
new Promise((resolve, reject) => {
    fs.stat("jane-eyre.txt").then((jeStats) => {
        fs.stat("moby-dick.txt").then((mdStats) => {
            fs.stat("life-of-frederick-douglass.txt").then((fdStats) => {
                resolve(jeStats.size + mdStats.size + fdStats.size)
            })
        })
    })
}).then((total) => console.log(total))
```

-   But this is a lot of nesting, doesn't scale, and (potentially) a lot of unnecessary waiting around
-   The answer is `Promise.all`
-   Returns an array of results from completed promises _after all have resolved_
    -   Order of results corresponds to that of promises in input array

```js
const fs = require('fs-extra')

var total_size = 0
var files = ["jane-eyre.txt", "moby-dick.txt", "life-of-frederick-douglass.txt"]
Promise.all(files.map(f => fs.stat(f))).then(stats => stats.reduce((t,s) => {return t + s.size}, 0)).then(console.log)
```

-   There is also `Promise.race`, which takes an array of promises and returns the result of the first one to complete

## Using Promises {#s:promises-usage}

-   Count the number of lines in a set of files over a certain size
-   Step 1: find input files

```js
const fs = require('fs-extra')
const glob = require('glob-promise') // a way to find things in the filesystem using wildcards. returns a promise

const srcDir = process.argv[2]

glob(`${srcDir}/**/*.txt`)
  .then(files => console.log('glob', files))
  .catch(error => console.error(error))
```
{: title=promises/step-01.js}
```
$ node step-01.js .
glob [ './common-sense.txt',
  './jane-eyre.txt',
  './life-of-frederick-douglass.txt',
  './moby-dick.txt',
  './sense-and-sensibility.txt',
  './time-machine.txt' ]
```

-   Step 2: get the status of each file
-   This doesn't work because `fs.stat` is delayed

```js
...imports and arguments as before...

glob(`${srcDir}/**/*.txt`)
  .then(files => files.map(f => fs.stat(f)))
  .then(files => console.log('glob + files.map/stat', files))
  .catch(error => console.error(error))
```
{: title=promises/step-02.js}
```
$ node step-02.js .
glob + files.map/stat [ Promise { <pending> },
  Promise { <pending> },
  Promise { <pending> },
  Promise { <pending> },
  Promise { <pending> },
  Promise { <pending> } ]
```

-   Step 3: use `Promise.all` to wait for all stat promises to resolve

```js
...imports and arguments as before...

glob(`${srcDir}/**/*.txt`)
  .then(files => Promise.all(files.map(f => fs.stat(f))))
  .then(files => console.log('glob + Promise.all(files.map/stat)', files))
  .catch(error => console.error(error))
```
{: title=promises/step-03.js}
```
$ node step-03.js
glob + Promise.all(files.map/stat) [ Stats {
    dev: 16777220,
    mode: 33188,
    ...more information... },
    ...five more Stats objects...
]
```

-   Step 4: we need to remember the name of the file we stat'd, so we need to write our own function that returns a pair

```js
...imports and arguments as before...

const statPair = (filename) => {
  return new Promise((resolve, reject) => {
    fs.stat(filename)
      .then(stats => resolve({filename, stats}))
      .catch(error => reject(error))
  })
}

glob(`${srcDir}/**/*.txt`)
  .then(files => Promise.all(files.map(f => statPair(f))))
  .then(files => console.log('glob + Promise.all(files.map/statPair)', files))
  .catch(error => console.error(error))
```
{: title=promises/step-04.js}
```
$ node step-04.js .
glob + Promise.all(files.map/statPair) [ { filename: './common-sense.txt',
    stats:
     Stats {
       dev: 16777220,
       mode: 33188,
       ...more information... }
     },
     ...five more (filename, Stats) pairs...
]
```

-   Step 5: make sure we're only working with files >100000 characters

```js
...imports and arguments as before...

glob(`${srcDir}/**/*.txt`)
  .then(files => Promise.all(files.map(f => statPair(f))))
  .then(files => files.filter(pair => pair.stats.size > 100000))
  .then(files => Promise.all(files.map(f => fs.readFile(f.filename, 'utf8'))))
  .then(contents => console.log('...readFile', contents.map(c => c.length)))
  .catch(error => console.error(error))
```
{: title=promises/step-05.js}
```
$ node step-05.js .
...readFile [ 148134, 1070331, 248369, 1276201, 706124, 204492 ]
```

-   Step 6: split into lines and count

```js
...imports and arguments as before...

const countLines = (text) => {
  return text.split('\n').length
}

glob(`${srcDir}/**/*.txt`)
  .then(files => Promise.all(files.map(f => statPair(f))))
  .then(files => files.filter(pair => pair.stats > 100000))
  .then(files => Promise.all(files.map(f => fs.readFile(f.filename, 'utf8'))))
  .then(contents => contents.map(c => countLines(c)))
  .then(lengths => console.log('lengths', lengths))
  .catch(error => console.error(error))
```
{: title=promises/step-06.js}
```
$ node step-06.js .
lengths [ 2654, 21063, 4105, 22334, 13028, 3584 ]
```
## `async` and `await` {#s:promises-async-await}

-   Review: work with the output of a promise with `.then`
-   Output of `.then` is another promise
-   So we can end up with long chains
-   We can use `async` and `await` to avoid this problem
-   We can write asynchronous functions very similarly to the synchronous ones that we're used to
    -   E.g., for the `statPair` function that we wrote earlier

```js
const fs = require('fs-extra')

const statPairAsync = async (filename) => {
    var stats = await fs.stat(filename)
    return {filename, stats}
}

statPairAsync("moby-dick.txt").then((white_whale) => console.log(white_whale.stats))
```

-   `async` functions still return a Promise
-   But we can then chain those with other `async` functions using `await`
-   `await` collects the result returned by a resolved promise, we can use `.catch` to handle any errors thrown
-   Let's convert the complete example from the previous section

```js
const fs = require('fs-extra')
const glob = require('glob-promise')

const statPairAsync = async (filename) => {
    var stats = await fs.stat(filename)
    return {filename, stats}
}

const countLines = (text) => {
  return text.split('\n').length
}

const processFiles = async (globpath) => {
    var filenames = await glob(globpath)
    var pairs = await Promise.all(filenames.map(f => statPairAsync(f)))
    var filtered = pairs.filter(pair => pair.stats.size > 100000)
    var contents = await Promise.all(filtered.map(f => fs.readFile(f.filename, 'utf8')))
    var lengths = contents.map(c => countLines(c))
    console.log(lengths)
}

const srcDir = process.argv[2]

processFiles(`${srcDir}/**/*.txt`)
  .catch(e => console.log(e.message))
```

-   Using `async` and `await` avoid need for long `then` chains, less nested
-   Can only use `await` with `async` functions - syntax error if used elsewhere

## Exercises {#s:promises-exercises}

### A Stay of Execution

Insert `console.log('This is a sharp Medicine, but it is a Physician for all diseases and miseries.')`
in the appropriate place in the code block below so that the output reads

```
Waiting...
This is a sharp Medicine, but it is a Physician for all diseases and miseries.
Waiting...
Finished.
```

```js
const holdingMessage = () => {
    console.log('Waiting...')
}

const swingAxe = () => {
    setTimeout(() => {
        holdingMessage()
        console.log('Finished.')
    }, 100)
    holdingMessage()
}

swingAxe()
```

### A Synchronous or Asynchronous?

Which of these functions would you expect to be asynchronous? How can you tell?
Does it matter? And, if so, what is a good strategy to find out for sure if a
function is asynchronous?

1. `findNearestTown(coords)`: given a set of coordinates (`coords`) in Brazil,
   looks up and returns the name of the the nearest settlement with an estimated population greater than 5000.
   Throws an error if `coords` fall outside Brazil.
2. `calculateSphereVolume(r)`: calculates and returns the volume of a sphere with radius `r`.
3. `calculateRoute(A,B)`: returns all possible routes by air between airports `A` and `B`,
   including direct routes and those with no more than 2 transfers.

### Handling Exceptions

What (if any) output would you expect to see in the console when the code below
is executed?

```js
const checkForBlanks = (inputValue) => {
    return new Promise((resolve, reject) => {
        if (inputValue === '') {
            reject(Error("Blank values are not allowed"))
        } else {
            resolve(inputValue)
        }
    })
}

new Promise((resolve, reject) => {
    setTimeout(() => {
        reject(Error('Timed out!'))
    }, 1000)
    resolve('')
}).then(
    output => checkForBlanks(output), error => console.log(error.message)).then(
        checkedOutput => console.log(checkedOutput)).catch(
            error => console.log(error.message))
```

a) `Timed out!`
b) blank output
c) `Blank values are not allowed`
d) a new `Promise` object

### Empty Promises

Fill in the blanks (`___`)in the code block below so that, when executed, the function
returns an `Array[7, 8, 2, 6, 5]`.

```js
const makePromise = (someInteger) => {
    return ___ Promise((resolve, reject) => {
        setTimeout(___(someInteger), someInteger*1000)
    })
}
Promise.___([makePromise(7), makePromise(___), makePromise(2), makePromise(6), makePromise(5)]).then(
    numbers => ___(numbers))
```

Now adapt the function so that it returns only `2`? (_Hint: you can achieve this
by changing only a single one of the blank fields._)

### `async`, Therefore I Am

Using `async` and `await`, convert the function below into an asynchronous function
with the same behaviour and output. Do you find your solution easier to read and
follow than the original version? Do you think that that is only because you
wrote this version?

{% include links.md %}
