---
layout: post
title: "Mocking JavaScript a little - test doubles with sinon.js, mocha and chai"
author: Jan Molak
date: 2014-07-22 22:10:57 +0100
published: true
comments: true
categories:
- test doubles
- tdd
- mocha
- chai
- sinon
---
How do you write a test in which you cannot (or chose not to) use real dependencies of whatever is that you're testing?
What are your options in JavaScript? And what does a test doubles library give you that plain-old-js can't?
Interested to find out? Read on!

<!--more-->

## System under test
Let's assume we have a situation similar to the one described by Uncle Bob Martin in "The Little Mocker"[^1].

We have a `System`:

```javascript src/system.js
'use strict';

/**
 * @constructor
 * @param {Authoriser} authoriser
 */
function System(authoriser) {

    /** @private */
    var login_count = 0;

    /** @public */
    this.login = function(username, password) {
        if (authoriser.authorise(username, password)) {
            ++login_count;
        }
    };

    /** @public */
    this.loginCount: function() {
        return login_count;
    };
}
```

that delegates the process of logging users in to the `Authoriser`, which implementation might look more or less like this:

```javascript src/authoriser.js
'use strict';

/** @constructor */
function Authoriser() { }

/**
 * @public
 *
 * @param {String} username
 * @param {String} password
 *
 * @returns {Boolean}
 */
Authoriser.prototype.authorise = function(username, password) {

    // connects to LDAP or some database, verifies user credentials
    // returns true if the username/password pair is valid, false otherwise
    // ...
};
```

## A dummy

How can we prove that a newly created `System` has no logged in users?
And do we even need to construct a real `Authoriser` object for that?

Since we know that calling `system.loginCount` does not even touch the `authoriser` object,
we can replace the `authoriser` with **a dummy**:

```javascript spec/system.spec.js
describe('A newly created System', function() {

    var dummy_authoriser = { };

    it('has no logged in users', function() {
        var system = new System(dummy_authoriser);

        expect(system.loginCount()).to.equal(0);
    });
});
```

And that's all there is to a dummy! You pass it into something when you have to provide an argument,
but you know that it will never be used.

## A stub

Let's now suppose that you want to test a part of your `System` that requires you to be logged in.

Of course you might say that you could *just log in*, but you've already tested that login works,
and since the process of logging in takes time, why do it twice? Also, if there's a bug in login your test
would fail too! That's an unnecessary coupling that can be easily avoided with **a stub**:

```javascript spec/system.spec.js
describe('A System with a logged in user', function() {

    var accepting_authoriser = { authorise: function() { return true; } };

    it('has a loginCount of 1', function() {
        var system = new System(accepting_authoriser);

        system.login('bob', 'SecretPassword');

        expect(system.loginCount()).to.equal(1);
    });
});
```
So what does this stubbed `accepting_authoriser` do? It returns true, therefore accepting any username/password pairs.

If you now wanted to test another part of your system that handles unauthorised users you could create a `rejecting_authoriser` like this:
```javascript
var rejecting_authoriser = { authorise: function() { return false; } };
```

So that's a stub, it simply returns a value and has no other logic in it.

{% blockquote Martin Fowler, Test Double %}
Stubs provide canned answers to calls made during the test, usually not responding at all to anything outside what's programmed in for the test.
{% endblockquote %}

Beautiful! So nice and simple, isn't it? Just one line of code! Isn't duck typing brilliant?
Imagine how much more code you'd have to write if you were using Java or C#!
You'd probably have to create an `Authoriser` interface in the first place,
then make sure both the real implementation and your stub `implement` that `interface`, then ...

Oh yeah, the `interface`... We don't have this construct in JavaScript, do we?

The trouble with using plain-old-js for hand-rolling your test stubs
is that you won't get a meaningful error when the interface of the stubbed-out object changes
and your test stubs are no longer correct. In worst case scenario you might not even get an error at all!

What could be the consequences of this, you may ask?
Well, I've seen huge test suites pass even though the application they were supposed to test was fundamentally broken
as half the code didn't even exist anymore... Trust me, you don't want to be there[^6] :-)

Right, so this situation is not cool, but can we do better than that? Yes we can! Enter sinon.js[^2]!

How would the `accepting_authoriser` look implemented with sinon then?

```javascript spec/system.spec.js
describe('A System with a logged in user', function() {

    var accepting_authoriser = sinon.createStubInstance(Authoriser);
    accepting_authoriser.authorise.returns(true);

    it('has a loginCount of 1', function() {
        var system = new System(accepting_authoriser);

        system.login('bob', 'SecretPassword');

        expect(system.loginCount()).to.equal(1);
    });
});
```

And what makes it different? Calling `sinon.createStubInstance(Authoriser)` creates a stub object
that allows you to stub out only those methods that exist on the `prototype` of the original `Authoriser`.

Now that's important because it means that should the `#authorise` method happily decide to change its
name to say `#isAuthorised` one day, you'd get a `TypeError` *in your test*,
exactly where you attempt to stub out the no longer existing `#authorise`. Cool, right?

## A spy

Do you remember when I said in the beginning that `System`
delegates the process of logging users in to the `Authoriser`?
How can we test whether or not this interaction really takes place?
What we'd need to do is spy on the caller to see inside the workings of the algorithm we're testing -
and that's precisely what **spies** are for[^3]:

```javascript spec/system.spec.js
describe('A System', function() {
    var accepting_authoriser_spy = {

        authorise_was_called: false,

        authorise: function() {
            this.authorise_was_called = true;

            return true;
        }
    }

    afterEach(function() {
        accepting_authoriser_spy.authorise_was_called = false
    };

    it('delegates logging users in to the authoriser', function() {
        var system = new System(accepting_authoriser_spy);

        system.login('bob', 'SecretPassword');

        expect(accepting_authoriser_spy.authorise_was_called).to.equal(true);
    });
});
```

You inject a spy object in exactly the same way you'd inject a stub, but then at the end of your test
you check if the interactions recorded by your spy are the ones you expected it to see.

{% blockquote Martin Fowler, Test Double %}
Spies are stubs that also record some information based on how they were called.
{% endblockquote %}

Another thing to note is that
because a spy maintains state (`authorise_was_called` flag in this example), it needs to be reset between the tests
to avoid one test case affecting another - this is done in the `afterEach` block.

Of course our hand-rolled implementation suffers from the problems I described earlier when we talked
about hand-rolling stubs. Let's improve our previous implementation with sinon:

```javascript spec/system.spec.js
describe('A System', function() {
    var accepting_authoriser = sinon.createStubInstance(Authoriser);

    accepting_authoriser.authorise.returns(true);

    afterEach(function() {
        accepting_authoriser.authorise.restore();
    });

    it('delegates logging users in to the authoriser', function() {
        var system = new System(accepting_authoriser);

        system.login('bob', 'SecretPassword');

        expect(accepting_authoriser.authorise).to.have.been.called;
    });
});
```

Similarly to the stub example, I'm also using `sinon.createStubInstance` here.

There's one significant difference between our hand-rolled spy implementation and the one above though:
**sinon spy** itself is not the main object you inject, it's a **wrapper around object's method**.
That's why I inject the `accepting_authoriser` but perform the assertion on `accepting_authoriser.authorise` spy and
use `accepting_authoriser.authorise.restore()` to reset it.

Additionally, the fact that sinon spy is a wrapper means that if you wanted to spy
on the real `Authoriser's` method you could do it like so:
```javascript
var authoriser    = new Authoriser(),
    system        = new System(authoriser),
    authorise_spy = sinon.spy(authoriser, 'authorise');

system.login('bob', 'SecretPassword');

expect(authorise_spy).to.have.been.called;
```

## A mock

{% blockquote Robert Martin (Uncle Bob), The Little Mocker %}
A mock is not so interested in the return values of functions.
It's more interested in what function were called, with what arguments, when, and how often.
{% endblockquote %}

Another thing that makes a mock[^4] different from a stub is its `verify` method,
which groups all the assertions and performs them at the end of a test, verifying our `System's` **behaviour**:

```javascript spec/system.spec.js
describe('A System', function() {
    var accepting_authoriser_mock = {

        authorise_was_called: false,

        authorise: function() {
            this.authorise_was_called  = true;
            return true;
        },

        verify: function() {
            expect(this.authorise_was_called).to.equal(true);
        }
    };

    it('delegates logging users in to the authoriser', function() {
        var system = new System(accepting_authoriser_mock);

        system.login('bob', 'SecretPassword');

        accepting_authoriser_mock.verify(); // assertions moved to #verify
    });
});
```

Now that we understand how mocks work and we know how to build them ourselves,
let's see how sinon can make things easier. There's an essential distinction we need to make
before diving into sinon mocks though:

{% blockquote Christian Johansen, "Test Spies&#44; Stubs and Mocks - Part 1.5" %}
In Sinon, the basic unit of faking is functions, always functions.
So a "stub object" in Java-type literature translates to a "stub function/method" in Sinon/JavaScript land.
{% endblockquote %}

... and same goes for sinon mocks. OK, but what does this mean to you? Let's have a look at differences in the implementation first:

```javascript spec/system.spec.js
describe('A System', function() {
    var authoriser = new Authoriser(),
        mock       = sinon.mock(authoriser);

    afterEach(function() {
        mock.restore();
    });

    it('delegates logging users in to the authoriser', function() {
        // prepare
        var system = new System(authoriser);

        // expect
        mock.expects('authorise').once().returns(true);

        // act
        system.login('bob', 'SecretPassword');

        // assert
        mock.verify();
    });
});
```

An important thing to note here is that sinon mock is created as a wrapper around
an instance of the **real** `Authoriser`, but instead of the mock we're injecting the already mentioned **real** instance
into the `System`.

This strategy might be problematic if instantiation of your depended-on object (`Authoriser` in our example)
is expensive; If that's the case you might want to avoid calling the constructor directly by using `Object.create`[^5]:

```javascript
var authoriser = Object.create(Authoriser.prototype),
    mock       = sinon.mock(authoriser);
```

Note: At the time of writing sinon.js (version 1.10.3) does not provide any equivalent of `createStubInstance`
for mocks.

## A fake

{% blockquote Robert Martin (Uncle Bob), The Little Mocker %}
Fake has business behavior. You can drive a fake to behave in different ways by giving it different data.
{% endblockquote %}

Suppose we wanted to define several personas to whom our `System` responded differently.
Let's say "bob" knows his password and "alice" forgot it; My personal recommendation
would be to use two separate stubs, but let's talk about fakes as some might prefer to use them instead:

```javascript spec/system.spec.js
describe('A System', function() {
    var fake_authoriser = {
            authorise: function(username, password) {
                return (username === 'bob' && password === 'SecretPassword');
            }
        },

        system = new System(fake_authoriser);

    it('allows Bob in', function() {
        system.login('bob', 'SecretPassword');

        expect(system.loginCount()).to.equal(1);
    });

    it('does not allow Alice in', function() {
        system.login('alice', 'password');

        expect(system.loginCount()).to.equal(0);
    });
});
```

... and you could also achieve the same result with sinon:

```javascript spec/system.spec.js
describe('A System', function () {
    var fake_authoriser = sinon.createStubInstance(Authoriser);
        fake_authoriser.authorise
            .returns(false)
            .withArgs('bob', 'SecretPassword').returns(true),

        system = new System(fake_authoriser);

    it('allows Bob in', function () {
        system.login('bob', 'SecretPassword');

        expect(system.loginCount()).to.equal(1);
    });

    it('does not allow Alice in', function () {
        system.login('alice', 'password');

        expect(system.loginCount()).to.equal(0);
    });
});
```
Be very careful with fakes as they can quickly become **extremely** complicated.
If you can use stubs instead - please do. If you can't then perhaps
it just highlighted that the thing you're trying to test is too complex and needs breaking down?

## Tools

Techniques described in this article can be applied to both front-end and back-end (node.js) development,
same applies to tools I used in the above examples, namely:

* [mocha](http://visionmedia.github.io/mocha/) - a popular JavaScript test framework
* [sinon.js](http://sinonjs.org/) - test doubles library
* [chai.js](http://chaijs.com/) and its [bdd-style](http://chaijs.com/api/bdd/) assertion library
* [sinon-chai](https://github.com/domenic/sinon-chai) - sinon.js assertions for chai

## Final thoughts and a word of warning

Bear in mind that the more you spy and mock, the tighter you couple your tests to the implementation of your system.
This leads to fragile tests that may break for reasons that shouldn't break them.

Test doubles should be used with care and applied only when they're the best tool for the task at hand.
Just like with any other tool in your development tool belt: don't use a screwdriver to knock in a nail just
because you got a new screwdriver and want to try it out ;-)

[^1]: [The Little Mocker](http://blog.8thlight.com/uncle-bob/2014/05/14/TheLittleMocker.html) blog post by Uncle Bob Martin. As you might have already noticed, Uncle Bob's post has inspired this article greatly!
[^2]: [Sinon.js](http://sinonjs.org/) was written by Christian Johansen, author of [this excellent book](http://www.amazon.co.uk/gp/product/0321683919/ref=as_li_ss_tl?ie=UTF8&camp=1634&creative=19450&creativeASIN=0321683919&linkCode=as2&tag=smartcode-21)
[^3]: To better understand what spies and other test doubles are and how to use them I highly recommend reading [this book](http://www.amazon.co.uk/gp/product/0131495054/ref=as_li_ss_tl?ie=UTF8&camp=1634&creative=19450&creativeASIN=0131495054&linkCode=as2&tag=smartcode-21)
[^4]: The concept of "mock objects" was originally introduced by Tim Mackinnon, Steve Freeman and Philip Craig in their paper [Endo-Testing: Unit Testing with Mock Objects](http://www.ccs.neu.edu/research/demeter/related-work/extreme-programming/MockObjectsFinal.PDF). Steve Freeman is a co-author of [the GOOSE book](http://www.amazon.co.uk/gp/product/0321503627/ref=as_li_ss_tl?ie=UTF8&camp=1634&creative=19450&creativeASIN=0321503627&linkCode=as2&tag=smartcode-21). Again, highly recommended.
[^5]: Object.create is well documented on [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
[^6]: If you already *are* in this situation [get in touch](/contact), I can help you to get out of it :-) OK, no more cheeky marketing. Get back to the article, quick, quick: