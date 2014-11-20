# Async Generator Proposal (ES7)

Async Generators are currently proposed for ES7 and are at the strawman phase. This proposal builds on the [async function](https://github.com/lukehoban/ecmascript-asyncawait) proposal.

JavaScript programs are single-threaded and therefore must streadfastly avoid blocking on IO operations. Today web developers must deal with a steadily increasing number of push stream APIs:

* Server sent events
* Web sockets
* DOM events

Developers should be able to easily consume these push data sources, as well as compose them together to build complex concurrent programs.

ES6 introduced generator functions for producing data via iteration, and a new for...of loop for consuming data via iteration.

```JavaScript
// data producer
function* nums() {
  yield 1;
  yield 2;
  yield 3;
}

// data consumer
function printData() {
  for(var x of nums()) {
    console.log(x);
  }
}
```

These features are ideal for progressively consuming data stored in collections or lazily-produced by computations. However they are not well-suited to consuming asynchronous streams of information, because Iteration is synchronous. 

The async generator proposal attempts to solve this problem by adding symmetrical support for Observation to ES7. It would introduce asynchronous generator functions for producing data via _observation_, and a new for..._on_ loop for consuming data via observation.

```JavaScript
// data producer
async function*() {
  yield 1;
  yield 2;
  yield 3;
}

// data consumer
async function printData() {
  for(var x on nums()) {
    console.log(x);
  }
}
```

The for..._on_ loop would allow any of the web's many push data streams to be consumed using the simple and familiar loop syntax. Here's an example that returns the first stock price that differs by a certain delta.

```JavaScript
async function getPriceSpikes(stockSymbol, int maxDelta) {
  var delta,
    oldPrice,
    price;
    
  for(var price on toObservable(new WebSocket("ws://www.fakedomain.com/stockstream/" + stockSymbol))) {
    if (oldPrice == null) {
      oldPrice = price;
    }
    else {
      delta = Math.abs(price - oldPrice);
      oldPrice = price;
      if (delta > maxDelta) {
        return {price, oldPrice};
      }
    }
  }
}

// get the first price that differs from previous spike by $5.00
getPriceSpikes("JNJ", 5.00).then(priceDelta => console.log("PRICE SPIKE:", priceDelta));
```

## Introducing Async Generators

An ES6 generator function differs from a normal function, in that it returns multiple values:

```JavaScript
function *nums() {
  yield 1;
  yield 2;
}

for(var num of nums) {
  console.log(num);
}
```

An [async function](https://github.com/lukehoban/ecmascript-asyncawait) (currently proposed for ES7) differs from a normal function, in that it _pushes_ its return value asynchronously via a Promise. A value is _pushed_ if it is delivered in the argument position, rather than in the return position.

```JavaScript
function getStockPrice(name) {
  return getPrice(getSymbol(name));
}

try {
  //  data delivered in return position
  var price = getStockPrice("JNJ");
  console.log(price);
}
catch(e) {
  console.error(e);
}

// async version of getStockPrice function
function async getStockPriceAsync(name) {
  return await getPriceAsync(await getSymbolAsync(name));
}

getStockPriceAsync("JNJ").
  then(
    // data delievered in argument position (push)
    price => console.log(price),
    error => console.error(error));
```

We can view these language features in a table like so:

|               | Sync          | Async         |
| ------------- |:-------------:|:-------------:|
| function      | T             | Promise<T>    |
| function*     | Iterator<T>   |      ???      |

An obvious question presents itself: _"What does an async generator function return?"_

```JavaScript
async function *getStockPrices(stockName, currency) {
  for(var price on getPrices(await getStockSymbol(stockName))) {
    yield convert(price, currency);
  }
}

// What type is prices?
var prices = getStockPrices("JNJ", "CAN");
```

If a generator function modifies a function and causes it to return multiple values and the async modifier causes functions to push their values, _an asynchronous generator function must push multiple values_. What data type fits this description?

## Introducing Observable

ES6 introduces the Generator interface, which is a combination of two different interfaces:

1. Iterator
2. Observer

The Iterator is a data source that can return a value, an error (via throw), or a final value (value where IterationResult::done).

```JavaScript
interface Iterator {
  IterationResult next();
}

type IterationResult = {done: boolean, value: any}

interface Iterable {
  Iterator iterator();
}
```

The Observer is a data _sink_ which can be pushed a value, an error (via throw()), or a final value (return()):

```JavaScript
interface Observer {
  void next(value);
  void return(returnValue);
  void throw(error);
}
```

These two data types mixed together forms a Generator:

```JavaScript
interface Generator {
  IterationResult next(value);
  IterationResult return(returnValue);
  IterationResult throw(error);
}
```

Iteration and Observation both enable a consumer to progressively retrieve 0...N values from a producer. _The only difference between Iteration and Observation is the party in control._ In iteration, the party in control is the consumer because it initiates the requests for the value and the producer must synchronously respond. 

In this example a consumer requests an Iterator from an Array, and progressively requests the next value until the stream is exhausted.

```JavaScript
function printNums(arr) {
  // requesting an iterator from the Array, which is an Iterable
  var iterator = arr[@@iterator](),
    pair;
  // consumer (this function)
  while(!(pair = iterator.next()).done) {
    console.log(pair.value);
  }
}
```

This code relies on the fact that in ES6 all collections implement the Iterable interface.

```JavaScript
interface Iterable {
  Iterator @@iterator()
}
```

ES6 also added special support for...of syntax, the program above can be rewritten like this:

```JavaScript
function printNums(arr) {
  for(var value of arr) {
    console.log(value);
  }
}
```

ES6 added great support for Iteration, but currently there is no equivalent of the Iterable type for Observation. How would we design such a type? By taking the dual of the Iterable type.

```JavaScript
interface Iterable {
  Generator @@iterator()
}
```

The dual of a type is derived by swapping the argument and return types, and taking the dual of each argument. The dual of a Generator is a generator, because it is symmetrical. The generator can both accept and return the same three messages:

1. data
2. error
3. final value

Therefore all that is left to do is swap the arguments and return type of the Iterator's iterator method and then we have an Observable.

```JavaScript
interface Observable {
  void @@observer(Generator observer)
}
```

An Observable accepts an Observer and pushes it 0...N values and optionally terminates by either pushes an error or a return value. This data type is what you get when you compose together the async and * function modifiers. 

In ES7, any collection that is Iterable can also Observable. Here is an implementation for Array.

```
Array.prototype[@@observer] = function(observer) {
  for(var x of this) {
    observer.next(v);
  }
  observer.return();
};
```

Async generators 
Async generators can be transpiled into Async functions. A transpiler is in the works.

The following code...

```JavaScript
async function* getStockPrices(stockNames, nameServiceUrl) {
    var stockPriceServiceUrl = await getStockPriceServiceUrl();

    // observable.buffer() -> AsyncObservable that supports backpressure by buffering
    for(var name on stockNames.buffer()) {
        // accessing arguments array instead of named paramater to demonstrate necessary renaming
        var price = await getPrice(await getSymbol(name, arguments[1]), stockPriceServiceUrl),
            topStories = [];

        for(var topStory on getStories(symbol).take(10)) {
            topStories.push(topStory);

            if (topStory.length === 2000) {
                break;
            }
        }

        // grab the last value in getStories - technically it's actually the return value, not the last next() value.
        var firstEverStory = await* getStories();

        // grab all similar stock prices and return them in the stream immediately
        // short-hand for: for(var x on obs) { yield x }
        yield* getStockPrices(getSimilarStocks(symbol), nameServiceUrl);

        // This is just here to demonstrate that you shouldn't replace yields inside a function
        // scope that was present in the unexpanded source. Note that this is a regular 
        // generator function, not an async one.
        var noop = function*() {
            yield somePromise;
        };

        yield {name: name, price: price, topStories: topStories, firstEverStory: firstEverStory };
    }
}
```

...can be transpiled into the [async/await](https://github.com/lukehoban/ecmascript-asyncawait) feature proposed for ES7:

```JavaScript
function getStockPrices(stockNames, nameServiceUrl) {
    var $args = Array.prototype.slice(arguments);

    return new Observable(function forEach($observer) {
        var $done,
            $decoratedObserver = 
                decorate(
                    $observer, 
                    // code when return or throw is called on observer
                    function() { $done = true});

        // inline invoke async function. This is like using a promise as a scheduler. Necessary
        // because ES6 doesn't expose microtask API
        (async function() {
            var stockPriceServiceUrl,
                name,
                $v0,
                price,
                topStories,
                topStory,
                firstEverStory,
                noop;

            // might've returned before microtask runs. This first check must run before any other
            // code. Not that the first await inline in the variable declaration has been moved down
            // beneath this line. 
            if ($done) { return; }

            stockPriceServiceUrl = await getStockPriceServiceUrl();
            if ($done) { return; }

            // for...on becomes forEach.
            // The function passed to observable.forEach becomes a next() function  the observer.
            // If the next method returns a Promise, the Observable must wait until the Promise is
            // resolved before pushing more values. This is how backpressure works.
            // If there is any await expression (or another for on) in the body of the for on, the 
            // function passed to forEach becomes an async function. Async functions return promises
            // so backpressure will be applied.
            await stockNames.buffer().forEach(async function($name) {
                // At the top of every forEach next() function, we must check if the async
                // function is short-circtuited via observer.return(). 
                if ($done) { this.return(); return; }

                name = $name;

                $v0 = await getSymbol(name, $args[1]);
                // Unsubscription might have happened after _every_ await, so expressions need to be 
                // broken into multiple statements so that we can check for done = true after each 
                // await, and return if done = true.             
                if ($done) { this.return(); return; }

                price = await getPrice($v0, stockPriceServiceUrl);
                if ($done) { this.return(); return; }

                topStories = [];

                // body of for...on contains no awaits, so function is not async
                await getStories.forEach(function($topStory) {
                    // check for unsubscription
                    if ($done) { this.return(); return; }
                    topStory = $topStory;
                    topStories.push(topStory);

                    if (topStory.length === 2000) {
                        // break turns into a return() and return
                        this.return();
                        return;
                    }
                });
                if ($done) { this.return(); return; }

                // await* blah just expands to await blah.returnValue()
                firstEverStory = await getStories().returnValue();
                if ($done) { this.return(); return; }

                // yield* obs -> for(var x on obs) { yield x } -> 
                // await obs.forEach(function(x) { if ($done) { this.return(); return; } $decoratatedObserver.next(x); })
                await getStockPrices(getSimilarStocks(symbol), nameServiceUrl).forEach(function($v1) {
                    // check for unsubscription
                    if ($done) { this.return(); return; }

                    $decoratedObserver.next($v1);
                });
                if ($done) { this.return(); return; }

                // Note that this yield is not replaced.
                noop = function*() {
                    yield somePromise;
                };

                // Every yield statement becomes a $decoratedObserver.next()
                $decoratedObserver.next({name: name, price: price, topStories: topStories, firstEverStory: firstEverStory });
            })
        }()).
            then(
                function(value) { 
                    if (!$done) {
                        decoratedObserver.return(value); 
                    }
                },
                function(error) { 
                    if (!done) {
                        return decoratedObserver.throw(error);
                    }
                });

        return decoratedObserver;
    });
}
```

