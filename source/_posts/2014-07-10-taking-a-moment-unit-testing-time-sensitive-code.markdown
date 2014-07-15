---
layout: post
title: "Taking a Moment - Unit Testing Time-Sensitive Code"
date: 2014-07-10 21:59:05 +0100
published: false
comments: true
categories:
- extreme javascript
- tdd
- moment
---

Let's assume you have a piece of time-sensitive JavaScript code, where the result
of its execution depends on the time when it is executed:

```javascript src/greeter.js
'use strict';

function Greeter() {}

Greeter.prototype.greet = function() {
    var now = new Date().getHours();

    if (5 <= now && now <= 12) {
        return 'Good morning';
    } else {
        return 'Hello';
    }
};
```
How would you test it?
<!--more-->

```javascript spec/greeter.spec.js
describe('Greeter', function() {
    it('says "Good morning" between 5 AM and noon', function() {
        var greeter = new Greeter();

        // todo: control the time ...

        expect(greeter.greet()).to.equal('Good morning');
    });
})
```

Ideally our `Greeter` object should not be relying on the system time directly,

## Inject the date


describe('Greeter', function() {
    it('says "Good morning" between 5 AM and noon', function() {
        var greeter = new Greeter();

        // todo: control the time ...

        expect(greeter.greet()).to.equal('Good morning');
    });
})

## Replace Date with Moment






OK, but what if you can't change the code? Use sinon.

