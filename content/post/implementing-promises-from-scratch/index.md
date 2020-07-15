---
date: 2020-07-05T11:58:08-03:00
description: "Creating a javascript promise from scratch to understand what it is and why we need them"
featured_image: "/images/implementing-promise-from-scratch/promise-logo.png"
og_image: "/images/implementing-promise-from-scratch/promise-logo.png"
tags: ["Javascript", "Promises", "Node", "TDD"]
title: "Making a promise to understand javascript promises"
summary: "In this post we are going to implement a complete promise from scratch using TDD. We are going to understand why promises, what they are and how they work internally."
author: Joao Paulo Lindgren
---

TLDR;

- In the beginning of this post, I try to give a small context, of why promises were introduced in the javascript and [what they are](#promises-to-the-rescue)
- Next, I propose to build a promise from scratch to understanding more deeply how it works. If you want to go directly to there click [here](#implementing-the-bastard-promise)
- If you want simply to see the code, or clone it, take a look at the
  [github repo](https://github.com/jplindgren/promise-from-scratch "Promise from scratch repository")

In the earlier days when the gods created the javascript, they created as a non-blocking language relying heavily on callbacks to run the background I/O operations. Using just one thread, javascript running on browser/node can call a Web API or a SO thread passing a function (callback) that should be called when the job is done, or when an error happens.

![Event Loop](/images/implementing-promise-from-scratch/event-loop.png)

Callbacks work fine and make node very responsive since it does not block its only thread.
So why we need promises?
Well, the great problem with callbacks is when complexity increases and we need to have nested callbacks. You probably already seen this piece of code

![Callback Hell](/images/implementing-promise-from-scratch/callbackhell.png)

Of course, the image is a joke, but in real life, if you have multiple nested callbacks things can get pretty wild.

## Promises to the rescue

To solve this problem, promises were created.
Following the joke of the last picture, this is how that code would look like with promises.

```javascript {linenos=inline}
const promisifyLoad = (disk) =>
    new Promise(resolve => floppy.load(disk, data => resolve(data)));
const promisifyPrompty = (message) =>
    new Promise(resolve => floppy.prompt(message, () => resolve()));
}

promisifyLoad('disk1')
    .then(promisifyPrompty('Please insert disk 2'))
    .then(promisifyLoad('disk2'))
    .then(promisifyPrompty('Please insert disk 3'))
    .then(promisifyLoad('disk3'))
    .then(promisifyPrompty('Please insert disk 4'))
    .then(promisifyLoad('disk5'))
    .then(() => {
        // if node.js would have existed in 1995 with promises!
    })
```

In essence, a promise is a proxy to a future value that you still do not have.
It is just it, a javascript object with certain properties to follow. No black magic behind it.
Some key behaviors of this "object" promise we can check in the specification.

http://www.ecma-international.org/ecma-262/6.0/#sec-promise-constructor

To get a better understanding of how a promise works, we are going to implement a promise object that I´ll call BastardPromise using the key aspects of a javascript promise.

- A promise should be created with an argument called **executor**. This argument must be a function and cannot be undefined.
- A new promise should have a "pending" state, should have an empty queue of fulfilled and rejected reactions, and should call the executor argument passing the its resolve, reject function.
- It can only be fulfilled or rejected and can be resolved only once.
- The methods then, catch, and finally should return a new promise.
- `Then` method should receive an onFulFill and onReject arguments and return a new promise. If the previous promise is fulfilled it should resolve the value. If it is pending, it should enqueue it to be resolved later.
- `Catch` method is just a wrapper for `then(_, onRejected).`
- `Finally` should return a new promise and must be called regardless of whether the promise is fulfilled or rejected.

I am assuming you are familiar with es6 syntax, how to create a new node project and how to create/run tests.

## Implementing the bastard promise

We are going to use Jest to do our tests, but because we are reimplementing something that already exists, you can use a lib called [Scientist](https://github.com/trello/scientist) for node to guide us. This is a cool lib ported from ruby that let you match an old behavior with a new behavior, it is really good for refactorings for example. In our case, we could, for example, create tests to check if the behaviors of the native javascript Promise matches our BastardPromise behavior.

First of all, create an empty class called `BastardPromise` and export it.

```javascript {linenos=inline}
class BastardPromise {}

module.exports = BastardPromise;
```

Next, create a test to define our constructor behavior.

```javascript {linenos=inline}
const BastardPromise = require("./bastard-promises");

describe("Bastard Promises", () => {
  describe("Initialization", () => {
    it("should be created in a pending state", () => {
      const promise = new BastardPromise(() => {});
      expect(promise.state).toBe("pending");
    });

    it("should be created with empty chained arrays", () => {
      const promise = new BastardPromise(() => {});
      expect(promise.fulFillReactionChainQueue).toEqual([]);
      expect(promise.rejectReactionChainQueue).toEqual([]);
    });

    it("should be created with undefned value", () => {
      const promise = new BastardPromise(() => {});
      expect(promise.innerValue).toBeUndefined();
    });

    it("should throw exception if executor is not a function", () => {
      expect(() => new BastardPromise({ data: "hi" })).toThrow(TypeError);
    });
  });
});
```

Now, we are going to implement our first piece of code in the `BastardPromise`.

```javascript {linenos=inline, hl_lines=["3-7"]}
class BastardPromise {
  constructor(executor) {
    if (typeof executor !== "function") throw new TypeError(`Promise resolver ${executor} is not a function`);

    this.state = "pending";
    this.fulFillReactionChainQueue = [];
    this.rejectReactionChainQueue = [];
    this.innerValue = undefined;
  }
}
```

- Our constructor receives a function as an argument and checks if it is a function. Later we are calling this function in our constructor.
- The state is set to "pending".
- Set the fulFillReactionChainQueue/rejectReactionChainQueue to empty. This queue will serve to hold references for resolve, reject values of promises that are not resolved yet.
- Set promise value to undefined.

By the time, we should have all of our tests passing. If you are new to node, you can run jest -watchAll to the tests run automatically, or in the VSCODE you can install jest test explorer to also run automatically your tests.

![tests green](/images/implementing-promise-from-scratch/tests_green_1.png)

To effectively resolve our promise value, we will need to implement a resolve and reject methods. Every time we create a promise, we need to pass a function (the executor argument). As soon as our promise is created it will trigger this function passing as arguments its resolve and reject function. The executor function should call resolve (to fulfill the promise), reject (to reject the promise), or return another promise.
The BastardPromise `resolve` method will be responsible to check if the promise is not pending, set its state to fulfilled, set its value, and call all the fulFillChain queue to resolve pending promises in the chain.

The `reject` method is similar to the resolve, except that we set the state as rejected. Also, we are going to trigger the pending reject chains.

Finally, with these two methods, we can call the executor function in the constructor passing both the resolve and the reject methods as arguments.

```javascript {linenos=inline, hl_lines=[10, "13-20", "22-29"]}
class BastardPromise {
  constructor(executor) {
    if (typeof executor !== "function") throw new TypeError(`Promise resolver ${executor} is not a function`);

    this.state = "pending";
    this.fulFillReactionChainQueue = [];
    this.rejectReactionChainQueue = [];
    this.innerValue = undefined;

    executor(this.resolve, this.reject);
  }

  resolve = (value) => {
    if (this.state !== "pending") return;

    this.state = "fulfilled";
    this.innerValue = value;

    for (const onFulFilled of this.fulFillReactionChainQueue) onFulFilled(this.innerValue);
  };

  reject = (error) => {
    if (this.state !== "pending" return;

    this.state = "rejected";
    this.innerValue = error;

    for (const onRejected of this.rejectReactionChainQueue) onRejected(this.innerValue);
  };
}
```

To avoid check the state against magic strings every time, we can create a `promise-state.js` to encapsulate the states.

```javascript {linenos=inline}
const state = {
  PENDING: "pending",
  FULFILLED: "fulfilled",
  REJECTED: "rejected",
};

module.exports = state;
```

After that, we can check and set the state using `state.PENDING` for instance.

Now we are able to create tests to check if our promise will be resolved/rejected correctly.

```javascript {linenos=inline}
it("should be able to be resolved", () => {
  const value = "success";
  const resolvedPromise = new BastardPromise((resolve) => resolve(value));
  expect(resolvedPromise.state).toBe(state.FULFILLED);
  expect(resolvedPromise.innerValue).toBe(value);
});

it("should be able to be rejected", () => {
  const error = new Error("promise rejected");
  const rejectedPromise = new BastardPromise((_, reject) => reject(error));
  expect(rejectedPromise.state).toBe(state.REJECTED);
  expect(rejectedPromise.innerValue).toBe(error);
});
```

Once again, our tests should be all green.

![tests green 2](/images/implementing-promise-from-scratch/tests_green_2.png)

## Adding thennable methods chain

We are ready for the next step. One of the key aspects of a promise is the capability of being chained and resolve other chains inside it.
The main method responsible for that behavior is the `then` method. Although is not hard, it is probably the hardest party to gasp in our promise object.
Let's write the tests for some behaviors.

```javascript {linenos=inline}
describe("chainning behavior", () => {
  describe("when fulFilled", () => {
    it("should throw exception if then onFulFill is not a function", () => {
      expect(() => new BastardPromise((resolve) => resolve("data")).then(null)).toThrow(
        new TypeError("onFulFill is not a function")
      );
    });

    it("should resolved value be passed to then", (done) => {
      return new BastardPromise((resolve) => resolve({ data: 777 })).then(({ data }) => {
        expect(data).toBe(777);
        done();
      });
    });

    it("should thennable methods be chained", (done) => {
      return new BastardPromise((resolve) => resolve({ data: 777 }))
        .then(({ data }) => {
          expect(data).toBe(777);
          return data + 223;
        })
        .then((res) => {
          expect(res).toBe(1000);
          done();
        });
    });

    it("should async resolved value be passed to then", (done) => {
      return new BastardPromise((resolve) => setTimeout(() => resolve({ data: 777 }), 5)).then(({ data }) => {
        expect(data).toBe(777);
        done();
      });
    });
  });
});
```

Of course, our tests should be broken right now. With these tests we defined some key behaviors of our `then` method. The first one is explicit from the specification. The onFulFill argument should be a function. The second key behavior is the capability of chain multiple "thens", resolving values in each of them.
The last test is important to check the ability to deal with async values. Instead of a function that resolves a value right way, we use a setTimeout to simulate an API call for example, and call resolve only on the callback.

Let's dive into the code to check how we accomplish all these things.

```javascript {linenos=inline}
then = (onFulFill) => {
  if (typeof onFulFill !== "function") throw TypeError("onFulFill is not a function");
  return new BastardPromise((resolve) => {
    const onFulFilled = (value) => resolve(onFulFill(value));

    if (this.state == state.FULFILLED) onFulFilled(this.innerValue);
    else this.fulFillReactionChainQueue.push(onFulFilled);
  });
};
```

- The first thing will be to check if the argument received (onFulFill) is a function.
- Second, we create a new promise passing an executor function that receives a resolver.
- Inside this executor function, we created a function that resolves the onFulFill argument from the `then` method.
- This function can take two paths. If the current promise is resolved `this.state == state.FULFILLED`, we resolve the onFulFill passing the current promise value synchronously.
- If the current promise is still pending, means that it is being resolved, so we add to the current fulFillReactionChainQueue to be resolved lately asynchronously.

If you are not experienced with javascript, the tricky part here is the notion that we are in a promise A, creating the promise B. And, although the executor is being called in the constructor of the promise B, the `this` is a closure referencing promise A. So, in the end, we check the state and add in the queue of the Promise A.
On the other hand, the `resolve` function is from promise B (passed as an argument to the executor), so when we enqueue the onFulFilled in the promise A, we are in fact enqueueing the resolve of the promise B. If sounds confusing, I encourage you to debug the code to get a better understand, you can even create unique ids to the promises to help you.

If everything is green move to the catch part.
The tests and the implementation.

```javascript {linenos=inline}
const getErrorObject = (message = "Promise has been rejected") => new Error(message);
describe("when rejected", () => {
  it("should catch", (done) => {
    const actualError = getErrorObject();
    new BastardPromise((resolve, reject) => reject(actualError)).catch((err) => {
      expect(err.message).toBe(actualError.message);
      done();
    });
  });

  it("should catch and then", (done) => {
    const actualError = getErrorObject();
    new BastardPromise((_, reject) => reject(actualError))
      .catch((err) => {
        expect(err.message).toBe(actualError.message);
        return { data: "someData" };
      })
      .then(({ data }) => {
        expect(data).toBe("someData");
        done();
      });
  });
});
```

Some basics scenarios of the catch are capturing the exception threw in previous promises, and "renew" the promise chain. That last part means that we can put a then after a catch, and in case the catch returns a value, it would be passed to the next then.

The catch implementation is something like this.

```javascript {linenos=inline}
catch = (onReject) => {
    return new BastardPromise((resolve, reject) => {
        const onRejected = (value) => resolve(onReject(value));

        if (this.state == state.REJECTED) onRejected(this.innerValue);
        else this.rejectReactionChainQueue.push(onRejected);
    });
};
```

The `catch` method is similar to the `then` method, but look if the state is rejected to decide if it will resolve now or later the onReject argument.
Notice, that we pass the result of the onReject call to the `resolve` method, and not to the reject method. This is because the properties of the catch is to "renew" the promise. So the value is resolved and the promise is set to fulfilled again.

![tests green 3](/images/implementing-promise-from-scratch/tests_green_3.png)

## Take advantage of tests to fix bug and refactoring

Hurray! We have a basic promise feature! We have made great progress until here, but if you have a keen eye, you´ll notice several things.
First, there are some bugs. For example, if we have a rejected promise, followed by a then, and a catch our promise is getting stuck. Another bug will happen if we return another promise inside of a then for example.
Another bug that we may find, is when our `onFulFill` or the `onReject` arguments passed to the `then`/`catch` method throws an exception, in that case, the correct behavior would be to reject the promise instead of let the code throw the exception.

Also, I´ll take the opportunity to make some refactorings. For example, our `then` method to follow the specification can receive not only the onFulFill as argument, but also an onReject argument. If we implement then like this, we can turn our catch into just a wrapper for `then(_, onReject)`.
Let's get to work! I hope that you already got the feeling of how things work internally, so I´ll speed up things a little bit from now on.

Adding the tests for the bugs that we found.

```javascript {linenos=inline}
describe("when fulFilled", () => {
  ... already existent tests

  it("should support chain of promises on which outside promises are returned", (done) => {
    const outsidePromise = new Promise((resolve) => setTimeout(() => resolve({ file: "photo.jpg" }), 10));

    return new BastardPromise((resolve) => {
      setTimeout(() => resolve({ data: "promise1" }), 10);
    })
      .then((response) => {
        expect(response.data).toBe("promise1");
        return outsidePromise;
      })
      .then((response) => {
        expect(response.file).toBe("photo.jpg");
        done();
      });
  });
});
describe("when rejected", () => {
  ... already existent tests

  it("should allow catch after a then with resolved promise", (done) => {
    const errorMessage = "Promise has been rejected";
    const thenFn = jest.fn();

    return new BastardPromise((resolve, reject) => {
      setTimeout(() => reject(new Error(errorMessage)), 10);
    })
      .then(thenFn)
      .catch((error) => {
        expect(error.message).toBe(errorMessage);
        expect(thenFn).toHaveBeenCalledTimes(0);
        done();
      });
  });

  it("should allow catch after a then with resolved promise", (done) => {
    const errorMessage = "Promise has been rejected";
    const thenFn = jest.fn();

    return new BastardPromise((resolve, reject) => {
      setTimeout(() => reject(new Error(errorMessage)), 10);
    })
      .then(thenFn)
      .catch((error) => {
        expect(error.message).toBe(errorMessage);
        expect(thenFn).toHaveBeenCalledTimes(0);
        done();
      });
  });

  it("should catch if returned inner promise is rejected", (done) => {
    const errorMessage = "an error ocorred";
    const anotherPromise = new BastardPromise((resolve, reject) => reject(new Error(errorMessage)));
    return new BastardPromise((resolve) => {
      resolve("promise1");
    })
      .then((response) => {
        expect(response).toBe("promise1");
        return anotherPromise;
      })
      .catch(({ message }) => {
        expect(message).toBe(errorMessage);
        done();
      });
  });

  it("should catch on then reject callback", (done) => {
    const thenResolveFn = jest.fn();
    const renewedData = { year: 1984 };
    new Promise((resolve, reject) => reject("just an error"))
      .then(thenResolveFn, (err) => {
        expect(err).toBe("just an error");
        return renewedData;
      })
      .then((res) => {
        expect(thenResolveFn).toHaveBeenCalledTimes(0), done();
        expect(res).toBe(renewedData);
        done();
      });
  });
});
```

Our revamped `BastardPromise` would be something like:

```javascript {linenos=inline, hl_lines=[7, "16-19", 24, 33, "40-45", 49, "51-69"]}
const state = require("./promise-states");
class BastardPromise {
  constructor(executor) {
    if (typeof executor !== "function") throw new TypeError(`Promise resolver ${executor} is not a function`);

    this.state = state.PENDING;
    this.chainReactionQueue = [];
    this.innerValue = undefined;

    executor(this.resolve, this.reject);
  }

  resolve = (value) => {
    if (this.state !== state.PENDING) return;

    if (value != null && typeof value.then === "function") {
      const result = value.then(this.resolve.bind(this), this.reject.bind(this));
      return result;
    }

    this.state = state.FULFILLED;
    this.innerValue = value;

    for (const { onFulFilled } of this.chainReactionQueue) onFulFilled(this.innerValue);
  };

  reject = (error) => {
    if (this.state !== state.PENDING) return;

    this.state = state.REJECTED;
    this.innerValue = error;

    for (const { onRejected } of this.chainReactionQueue) onRejected(this.innerValue);
  };

  then = (onFulFill, onReject) => {
    if (typeof onFulFill !== "function") throw TypeError("onFulFill is not a function");

    return new BastardPromise((resolve, reject) => {
      const onRejected = this.getOnRejectedAction(resolve, reject, onReject);
      const onFulFilled = this.getOnFulFilledAction(resolve, reject, onFulFill);

      if (this.state == state.FULFILLED) onFulFilled(this.innerValue);
      else if (this.state == state.REJECTED) onRejected(this.innerValue);
      else this.chainReactionQueue.push({ onFulFilled, onRejected });
    });
  };

  catch = (onReject) => this.then(() => {}, onReject);

  resolveOrFallback(resolve, reject) {
    try {
      resolve();
    } catch (err) {
      reject(err);
    }
  }

  getOnRejectedAction = (resolve, reject, onReject) => {
    return (value) => {
      this.resolveOrFallback(() => (onReject != null ? resolve(onReject(value)) : reject(value)), reject);
    };
  };

  getOnFulFilledAction = (resolve, reject, onFulFill) => {
    return (value) => {
      this.resolveOrFallback(() => resolve(onFulFill(value)), reject);
    };
  };
}

module.exports = BastardPromise;
```

- In the constructor, we changed the two arrays, one for fulfilled and others for rejected reactions to only one. This is not strictly necessary, but I think the code is cleaner and with less information to confuse the reader.
- To fix the test that we return a promise inside the `then`, we changed the `resolve` method to identify if the value is a promise (we check for the then method), in case true, we resolve that promise.
- The `then` method concentrates most of the changes. Now it allows to the caller send two functions as arguments. One when is fulfilled and the other when the promise is rejected. Now we are in compliance with the specification and fix the bug when a promise is rejected followed by a then and catch (test case **should allow catch after a then with resolved promise**).
- We encapsulate the callback methods `onFulFilled`, `onRejected` in the `then` method to be executed inside a try/catch. In case an exception is thrown, we reject the promise on the catch.
- Because then is allowed to receive a onReject, our catch now is only a wrapper for a `then(() => {}, onReject)` call.

In the end, we should have all our tests passing.

![tests green 4](/images/implementing-promise-from-scratch/tests_green_4.png)

That's it, our promise is basically completed. In the next section as a bonus, we are going to implement static ´all´ and ´race´ methods because they are very common and important in a lot of use cases.

As an overview, promises are just proxy objects that work with callbacks to let you chain future values.

If you are wondering what is the difference to Async/Await, it is basically nothing. Async/Await is just a syntax sugar over the promises to let you write async code in a synchronous way.
In fact, the async function return a promise. Almost every case you can use a case, you can change for async/await, except in the cases of running then in parallel where is usual to use `promise.all` and `promise.race` like we are going to see above.

Promises

```javascript {linenos=inline}
return asyncFunction()
  .then((result) => f1(result))
  .then((result2) => f2(result2));
```

async/await

```javascript {linenos=inline}
const result = await asyncFunction();
const result2 = await f1(result);
return await f2(result2);
```

## [Bonus] Implementing all and race methods

A very common use case is when you have multiple promises and want to wait to all o them complete to do some work, or you need to respond only to the fastest resolved one.
To accomplish that, we need to implement these two static methods. As we did before, we will start writing the tests.

```javascript {linenos=inline}
describe("static methods", () => {
  describe("promise all", () => {
    it("should wait untill all promise are resolved and pass array of results to then", (done) => {
      const p1 = new BastardPromise((resolve) => resolve({ data: "syncData" }));
      const p2 = new BastardPromise((resolve) => setTimeout(resolve({ data: "async" }), 100));
      return BastardPromise.all([p1, p2]).then((results) => {
        expect(results).toContainEqual({ data: "syncData" });
        expect(results).toContainEqual({ data: "async" });
        done();
      });
    });

    it("should reject if one promise fail", (done) => {
      const thenFn = jest.fn();
      const p1 = new BastardPromise((_, reject) => setTimeout(() => reject("error"), 10));
      const p2 = new BastardPromise((resolve) => resolve("async"), null, "p2");
      return BastardPromise.all([p1, p2])
        .then(thenFn)
        .catch((err) => {
          expect(err).toBe("error");
          expect(thenFn).toHaveBeenCalledTimes(0);
          done();
        });
    });
  }); //all

  describe("promise race", () => {
    it("should return first promise to resolve", (done) => {
      const p1 = new BastardPromise((resolve) => resolve({ data: "syncData" }));
      const p2 = new BastardPromise((resolve) => setTimeout(resolve({ data: "async" }), 10));
      return BastardPromise.race([p1, p2]).then((result) => {
        expect(result).toEqual({ data: "syncData" });
        done();
      });
    });

    it("should fulfill if the first promise is resolved despite other errors", (done) => {
      const resolvedObject = { data: "resolvedWithSuccess" };
      const promise1 = new BastardPromise((resolve) => setTimeout(() => resolve(resolvedObject), 5));
      const promise2 = new BastardPromise((resolve, reject) => setTimeout(() => reject({ data: "async2" }), 20));

      return BastardPromise.race([promise1, promise2]).then((result) => {
        expect(result).toEqual(resolvedObject);
        done();
      });
    });

    it("should reject if the first promise is rejected", (done) => {
      const actualError = getErrorObject();
      const promise1 = new Promise((resolve) => setTimeout(() => resolve("resolve"), 20));
      const promise2 = new Promise((_, reject) => setTimeout(() => reject(actualError), 5));

      return Promise.race([promise1, promise2]).catch((error) => {
        expect(error.message).toBe(actualError.message);
        done();
      });
    });
  }); // race
}); // static methods
```

- The `all` method must return a new promise and wait until all the promises passed as an array argument be fulfilled to pass to resolve the next then.
- The `all` method returned promise should be rejected if just one promise passed as argument is rejected.
- The `race` method must return a new promise, that will be resolved with the value fo the first resolved promise passed as an argument to it.
- The `race` method returned promise should be rejected if the first promised to resolve of the array passed as argument is rejected.

The `all` and `race` implementation.

```javascript {linenos=inline}
static all = (promises) => {
    let results = [];
    const reducedPromise = promises.reduce(
      (prev, curr) => prev.then(() => curr).then((p) => results.push(p)),
      new BastardPromise((resolve) => resolve(null))
    );
    return reducedPromise.then(() => results);
};

static race = (promises) =>
    new Promise((res, rej) => {
        promises.forEach((p) => p.then(res).catch(rej));
    });
```

## [Bonus 2] Implementing finally method!

I completely forgot about the `finally` method! Altough it was not present in since ecmascript 6 like the other methods it is a easy one to implement and a very nice feature to have in our promise!
The `finally` is also a chainable method that returns a new promise and is always called no matter the promise is fulfilled or rejected. Also, the `finally` method does not receive a value from the previous promise.

```javascript {linenos=inline}
it("should then and call finally afterall", (done) => {
  new BastardPromise((resolve) => setTimeout(() => resolve({ data: 777 }), 5))
    .then(({ data }) => expect(data).toBe(777))
    .finally((res) => {
      expect(res).toBeUndefined();
      done();
    });
});

it("should catch and call finally afterall", (done) => {
  new BastardPromise((resolve, reject) => reject("just an error"))
    .catch((err) => expect(err).toBe("just an error"))
    .finally((res) => {
      expect(res).toBeUndefined();
      done();
    });
});
```

```javascript {linenos=inline}
finally = (onFinally) => {
    return new BastardPromise((resolve) => {
      const onFinalized = () => resolve(onFinally());
      if (this.state == state.FULFILLED) onFinalized();
      else this.finallyChainReactionQueue.push(onFinalized);
    });
  };
```

Do not forget to initialize the finallyChainReactionQueue as an empty array in the constructor.

[You can access the complete code here](https://github.com/jplindgren/promise-from-scratch)

## References

Very nice video series from Waldemar Neto talking about promises. Unfortunately or fortunately depending on what language do you speak is recorded in Portuguese.

https://www.youtube.com/watch?v=CcL2WZRvROQ

Very nice article also showing how to implement a promise.

https://thecodebarbarian.com/write-your-own-node-js-promise-library-from-scratch.html

Nice article talking about Tasks, microtasks, queues, and schedules. If you read this article, you will understand a difference from our promise to the native promise. The native promise executes its function putting into the microtask queue even if it is synchronous.
https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
