---
title: Applicative Functors
---

## Applying Applicatives

The name **applicative functor** is pleasantly descriptive given its functional origins. Functional programmers are notorious for coming up with names like `mappend` or `liftA4`, which seem perfectly natural when viewed in the math lab, but hold the clarity of an indecisive darth vader at the drive thru in any other context.

Anyhow, the name should spill the beans on what this interface gives us: *the ability to apply functors to each other*.

Now, why would a normal, rational person, such as yourself, want such a thing? What does it even *mean* to apply one functor to another?

To answer these questions, we'll start with a situation you may have already encountered in your functional travels. Let's say, hypothetically, that we have two functors (of the same type) and we'd like to call a function with both of their values as arguments. Something simple like adding the values of two `Container`s.

```js
// we can't do this because the numbers are bottled up.
add(Container.of(2), Container.of(3));
//NaN

// Let's use our trusty map
var container_of_add_2 = map(add, Container.of(2));
// Container(add(2))
```

We have ourselves a `Container` with a partially applied function inside. More specifically, we have a `Container(add(2))` and we'd like to apply its `add(2)` to the `3` in `Container(3)` to complete the call. In other words, we'd like to apply one functor to another.

Now, it just so happens that we already have the tools to accomplish this task. We can `chain` and then `map` the partially applied `add(2)` like so:

```js
Container.of(2).chain(function(two) {
  return Container.of(3).map(add(two));
});
```

The issue here is that we are stuck in the sequential world of monads wherein nothing may be evaluated until the previous monad has finished its business. We have ourselves two strong, independent values and I should think it unnecessary to delay the creation of `Container(3)` merely to satisfy the monad's sequential demands.

In fact, it would be lovely if we could succinctly apply one functor's contents to another's value without these needless functions and variables should we find ourselves in this pickle jar.


## Ships in bottles

<img src="images/ship_in_a_bottle.jpg" alt="http://hollycarden.deviantart.com" />

`ap` is a function that can apply the function contents of one functor to the value contents of another. Say that 5 times fast.

```js
Container.of(add(2)).ap(Container.of(3));
// Container(5)

// all together now
Container.of(2).map(add).ap(Container.of(3));
// Container(5)
```

There we are, nice and neat. Good news for `Container(3)` as it's been set free from the jail of the nested monadic function. It's worth mentioning again that `add`, in this case, gets partially applied during the first `map` so this only works when `add` is curried.

We can define `ap` like so:

```js
Container.prototype.ap = function(other_container) {
  return other_container.map(this.__value);
}
```

Remember, `this.__value` will be a function and we'll be accepting another functor so we need only `map` it. And with that we have our interface definition:


> An *applicative functor* is a pointed functor with an `ap` method

Note the dependence on **pointed**. The pointed interface is crucial here as we'll see throughout the following examples.

Now, I sense your skepticism (or perhaps confusion and horror), but keep an open mind; this `ap` character will prove useful. Before we get into it, let's explore a nice property.

```js
F.of(x).map(f) == F.of(f).ap(F.of(x))
```

In proper English, mapping `f` is equivalent to `ap`ing a functor of `f`. Or in properer English, we can place `x` into our container and `map(f)` OR we can lift both `f` and `x` into our container and `ap` them. This allows us to write in a left-to-right fashion:

```js
Maybe.of(add).ap(Maybe.of(2)).ap(Maybe.of(3));
// Maybe(5)

Task.of(add).ap(Task.of(2)).ap(Task.of(3));
// Task(5)
```

One might even recognise the vague shape of a normal function call if viewed mid squint. We'll look at the pointfree version later in the chapter, but for now, this is the preferred way to write such code. Using `of`, each value gets transported to the magical land of containers, this parallel universe where each application can be async or null or what have you and `ap` will apply functions within this fantastical place. It's like building a ship in a bottle.

Did you see there? We used `Task` in our example. This is a prime situation where applicative functors pull their weight. Let's look at a more in-depth example.

## Coordination Motivation

Say we're building a travel site and we'd like to retrieve both a list of tourist destinations and local events. Each of these are separate, stand-alone api calls.

```js
// Http.get :: String -> Task Error HTML

var renderPage = curry(function(destinations, events) { /* render page */  });

Task.of(renderPage).ap(Http.get('/destinations')).ap(Http.get('/events'))
// Task("<div>some page with dest and events</div>")
```

Both `Http` calls will happen instantly and `renderPage` will be called when both are resolved. Contrast this with the monadic version where one `Task` must finish before the next fires off. Since we don't need the destinations to retrieve events, we are free from sequential evaluation.

Again, because we're using partial application to achieve this result, we must ensure `renderPage` is curried or it will not wait for both `Tasks` to finish. Incidentally, if you've ever had to do such a thing manually, you'll appreciate the astonishing simplicity of this interface. This is the kind of beautiful code that takes us one step closer to the singularity.

Let's look at another example.

```js
// Helpers:
// ==============
//  $ :: String -> IO DOM
var $ = function(selector) {
  return new IO(function() { return document.querySelector(selector)});
}

//  getVal :: String -> IO String
var getVal = compose(map(_.prop('value')), $);

// Example:
// ===============
//  signIn :: String -> String -> Bool -> User
var signIn = curry(function(username, password, remember_me) { /* signing in */  })

IO.of(signIn).ap(getVal('#email')).ap(getVal('#password')).ap(IO.of(false));
// IO({id: 3, email: "gg@allin.com"})
```

`signIn` is a curried function of 3 arguments so we have to `ap` accordingly. With each `ap`, `signIn` receives one more argument until it is complete and runs. We can continue this pattern with as many arguments as necessary. Another thing to note is that two arguments end up naturally in `IO` whereas the last one needs a little help from `of` to lift it into `IO` since `ap` expects the function and all its arguments to be in the same type.

## Bro, do you even lift?

Let's examine a pointfree way to write these applicative calls. Since we know `map` is equal to `of/ap`, we can write generic functions that will `ap` as many times as we specify:

```js
var liftA2 = curry(function(f, functor1, functor2) {
  return functor1.map(f).ap(functor2);
});

var liftA3 = curry(function(f, functor1, functor2, functor3) {
  return functor1.map(f).ap(functor2).ap(functor3);
});

//liftA4, etc
```

`liftA2` is a strange name. It sounds like one of the finicky freight elevators in a rundown factory or a vanity plate for a cheap limo company. Once enlightened, however, it's self explanatory: lift these pieces into the applicative functor world.

When I first saw this 2-3-4 nonsense it struck me as ugly and unnecessary, after all, we can check arity of functions in JavaScript and build this up dynamically. However, it is often useful to partially apply `liftA(N)` itself, so it cannot vary in argument length.

Let's see this in use:

```js
// checkEmail :: User -> Either String Email
// checkName :: User -> Either String String

//  createUser :: Email -> String -> IO User
var createUser = curry(function(email, name) { /* creating... */ });

Either.of(createUser).ap(checkEmail(user)).ap(checkName(user));
// Left("invalid email")

liftA2(createUser, checkEmail(user), checkName(user));
// Left("invalid email")
```

Since `createUser` takes two arguments, we use the corresponding `liftA2`. The two statements are equivalent, but the `liftA2` version has no mention of `Either`. This makes it more generic and flexible since we are no longer married to a specific type.


Let's see the previous examples written this way:

```js
liftA2(add, Maybe.of(2), Maybe.of(3));
// Maybe(5)

liftA2(renderPage, Http.get('/destinations'), Http.get('/events'))
// Task("<div>some page with dest and events</div>")

liftA3(signIn, getVal('#email'), getVal('#password'), IO.of(false));
// IO({id: 3, email: "gg@allin.com"})
```


## Operators

In languages like haskell, scala, PureScript, and swift, where it is possible to create your own infix operators you may see syntax like this:

```hs
-- haskell
add <$> Right 2 <*> Right 3
```

```js
// JavaScript
map(add, Right.of(2)).ap(Right.of(3))
```

It's helpful to know that `<$>` is `map` (aka `fmap`) and `<*>` is just `ap`. This allows for a more natural function application style and can help remove some parenthesis.

# Free can openers
<img src="images/canopener.jpg" alt="http://breannabeckmeyer.com/drawing.html" />

We haven't spoken much about derived functions. Seeing as all of these interfaces are built off of each other and obey a set of laws, we can define some weaker interfaces in terms of the stronger ones.

For instance, we know that an applicative is first a functor, so if we have an applicative instance, surely we can define a functor for our type.

This kind of perfect computational harmony is possible because we're working within a mathematical framework. Mozart couldn't have done better even if he had torrented ableton as a child.

I mentioned earlier that `of/ap` is equivalent to `map`. We can use this knowledge to define `map` for free:

```js
// map derived from of/ap
X.prototype.map = function(f) {
  return this.constructor.of(f).ap(this);
}
```

Monads are at the top of the food chain, so to speak, so if we have `chain`, we get functor and applicative for free:

```js
// map derived from chain
X.prototype.map = function(f) {
  var m = this;
  return m.chain(function(a) {
    return m.constructor.of(f(a));
  });
}

// ap derived from chain/map
X.prototype.ap = function(other) {
  return this.chain(function(f) {
    return other.map(f);
  });
};
```

If we can define a monad, we can define both the applicative and functor interfaces. This is quite remarkable as we get all of these can openers for free. We can even examine a type and automate this process.

It should be pointed out that part of `ap`'s appeal is the ability to run things concurrently so defining it via `chain` is missing out on that optimization. Despite that, it's good to have an immediate working interface while one works out the best possible implementation.

Why not just use monads and be done with it, you ask? It's good practice to work with the level of power you need, no more, no less. This keeps cognitive load to a minimum by ruling out possible functionality. For this reason, it's good to favor applicatives over monads.

Monads have the unique ability to sequence computation, assign variables, and halt further execution all thanks to the downward nesting structure. When one sees applicatives in use, they needn't concern themselves with any of that business.

Now, on to the legalities...

# Laws

Like the other mathematical constructs we've explored, applicative functors hold some useful properties for us to rely on in our daily code. First off, you should know that applicatives are "closed under composition", meaning `ap` will never change container types on us(yet another reason to favor over monads). That's not to say we cannot have multiple different effects - we can stack our types knowing that they will remain the same during the entirety of our application.

To demonstrate:

```js
  var tOfM = compose(Task.of, Maybe.of);

  liftA2(liftA2(_.concat), tOfM('Rainy Days and Mondays'), tOfM(' always get me down'));
  // Task(Maybe('Rainy Days and Mondays always get me down'))
```

See, no need to worry about different types getting in the mix.

Time to look at our favorite categorical law: *identity*:

### Identity

```js
// identity
A.of(id).ap(v) == v
```

Right, so applying `id` all from within a functor shouldn't alter the value in `v`. For example:

```js
var v = Identity.of("Pillow Pets");
Identity.of(id).ap(v) == v
```
`Identity.of(id)` makes me chuckle at its futility. Anyway, what's interesting is that, as we've already established, `of/ap` is the same as `map` so this law follows directly from functor identity: `map(id) == id`.

The beauty in using these laws is that, like a militant kindergarten gym coach, they force all of our interfaces to play well together.

### Homomorphism

```js
// homomorphism
A.of(f).ap(A.of(x)) == A.of(f(x))
```

A *homomorphism* is just a structure preserving map. In fact, a functor is just a *homomorphism* between categories as it preserves the original category's structure under the mapping.



We're really just stuffing our normal functions and values into a container and running the computation in there so it should come as no surprise that we will end up with the same result if we apply the whole thing inside the container (left side of the equation) or apply it outside, then place it in there (right side).

A quick example:

```js
Either.of(_.toUpper).ap(Either.of("oreos")) == Either.of(_.toUpper("oreos"))
```

### Interchange

The *interchange* states that it doesn't matter if we choose to lift our function into the left or right side of `ap`.

```js
// interchange
v.ap(A.of(x)) == A.of(function(f) { return f(x) }).ap(v)
```

Here is an example:

```js
var v = Task.of(_.reverse);
var x = 'Sparklehorse';

v.ap(Task.of(x)) == Task.of(function(f) { return f(x) }).ap(v)
```

### Composition

And finally composition which is just a way to check that our standard function composition holds when applying inside of containers.

```js
// composition
A.of(compose).ap(u).ap(v).ap(w) == u.ap(v.ap(w));
```

```js
var u = IO.of(_.toUpper);
var v = IO.of(_.concat("& beyond"));
var w = IO.of("blood bath ");

IO.of(_.compose).ap(u).ap(v).ap(w) == u.ap(v.ap(w))
```

## In Summary

A good use case for applicatives is when one has multiple functor arguments. They give us the ability to apply functions to arguments all within the functor world. Though we could already do so with monads, we should prefer applicative functors when we aren't in need of monadic specific functionality.

We're almost finished with container APIs. We've learned how to `map`, `chain`, and now `ap` functions. In the next chapter, we'll learn how to work better with multiple functors and disassemble them in a principled way.



## Exercises

```js
require('./support');
var Task = require('data.task');
var _ = require('ramda');

// fib browser for test
var localStorage = {};



// Exercise 1
// ==========
// Write a function that adds two possibly null numbers together using Maybe and ap().

//  ex1 :: Number -> Number -> Maybe Number
var ex1 = function(x, y) {
  // write me
};


// Exercise 2
// ==========
// Now write a function that takes 2 Maybe's and adds them. Use liftA2 instead of ap().

//  ex2 :: Maybe Number -> Maybe Number -> Maybe Number
var ex2 = undefined;



// Exercise 3
// ==========
// Run both getPost(n) and getComments(n) then render the page with both. (The n arg is arbitrary.)
var makeComments = _.reduce(function(acc, c) { return acc+"<li>"+c+"</li>" }, "");
var render = _.curry(function(p, cs) { return "<div>"+p.title+"</div>"+makeComments(cs); });

//  ex3 :: Task Error HTML
var ex3 = undefined;



// Exercise 4
// ==========
// Write an IO that gets both player1 and player2 from the cache and starts the game.
localStorage.player1 = "toby";
localStorage.player2 = "sally";

var getCache = function(x) {
  return new IO(function() { return localStorage[x]; });
};
var game = _.curry(function(p1, p2) { return p1 + ' vs ' + p2; });

//  ex4 :: IO String
var ex4 = undefined;





// TEST HELPERS
// =====================

function getPost(i) {
  return new Task(function (rej, res) {
    setTimeout(function () { res({ id: i, title: 'Love them futures' }); }, 300);
  });
}

function getComments(i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res(["This book should be illegal", "Monads are like space burritos"]);
    }, 300);
  });
}
```
