---
layout: sketchnote
title: "Testing Pyramid"
date: 2021-06-28 00:00:00
thumbnail: /assets/images/sketchnotes/2021-07-27-encapsulate-what-varies/encapsulate-what-varies-thumbnail.jpg
---

![{{page.title}}](/assets/images/sketchnotes/2021-07-27-encapsulate-what-varies/encapsulate-what-varies.jpg)

Encapsulate What Varies, or 'Encapsulate What Changes' is the technique of reducing the impact of frequently changing code by encapsulating it. The encapsulated code can then change independently to code that relies on it.

Whilst this principle relates specifically to code that frequently changes there are many benefits to encapsulating code, unrelated to how often it changes. Such as:

* Encapsulated code can be reasoned about in isolation from the code
* Preventing tight coupling to the implementation of the encapsulated code e.g. if you encapsulate the interaction of a downstream system, then replacing that downstream system becomes easier
* Encapsulating constants can indicate to other developers that they're related and provides control over if/how they're changed

## Related Reading

- [Encapsulate What Varies](https://alexkondov.com/encapsulate-what-varies/) - Succinct article with good examples
- [Wikipedia page on encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming))