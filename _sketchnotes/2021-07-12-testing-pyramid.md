---
layout: sketchnote
title: "Testing Pyramid"
date: 2021-06-28 00:00:00
thumbnail: /assets/images/sketchnotes/2021-07-12-testing-pyramid/testing-pyramid-thumbnail.jpg
image: /assets/images/sketchnotes/2021-07-12-testing-pyramid/testing-pyramid.jpg
---

The Testing Pyramid, devised by [Mike Cohn](https://twitter.com/mikewcohn), is a guide for the types of automated tests
to favour in a project's test suite. It points at the unreliability, slow and costly nature of end-to-end tests by
placing them at the top smaller section of the pyramid and the quick, cheap and isolated unit tests at the bottom, largest
section.

However, remember the Testing Pyramid is only a guide, as the [Practical Test Pyramid article](https://martinfowler.com/articles/practical-test-pyramid.html) perfectly sums up:

> Still, due to its simplicity the essence of the test pyramid serves as a good rule of thumb when it comes to establishing your own test suite. Your best bet is to remember two things from Cohn's original test pyramid:
>
>    1. Write tests with different granularity
>    2. The more high-level you get the fewer tests you should have
