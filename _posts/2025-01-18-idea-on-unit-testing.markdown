---
layout: post
comments: false
title:  "Unit testing and beyond"
excerpt: "Unit testing is an underrated skill. It's not optional, it's an integral component of quality software. Unit tests are developer's accountability. But don't limit to just unit tests. Aim for developer driven integration testing."
date:   2025-01-18 10:00:00
---

Unit testing is an underrated skill.

Even in 2025 I find "software sponsors" in denial that unit testing and integration testing adds value to the product. So, they don't want developers to spend time on writing tests, because it's the tester's job. Eventually we have unstable and fragile code bases that very few people understand and remember.

Here is the thing,

A developer needs to write unit tests and integration tests to provide evidence that the delivered code works, based on as many scenarios thought of. Then these tests themselves become documentation. But don't limit to just unit tests. Developer driven integration testing is what brings real quality! With modern tools like [testcontainers](https://testcontainers.com/), it becomes even easier and more relevant to have integration tests that works on real database, cache and message queue instances and verifies SQL, caching, messaging, events, etc., that we would have perhaps mocked in a pre-Docker era.

This saves a hell-lot-of-time for developers also, you make a new change, add tests, run tests, as long as existing tests and new tests are Ok, you push the change. As simple as that. The developer doesn't have to go around verifying other scenarios ensuring the new change has not broken anything. Add them in the pipelines, you have test coverage reports... This is how modern teams work! Creates and maintains the teams' confidence on their deliverables.

You need dedicated testers as well; they are the last check point before the product is released. They can be super productive by using automation tests. Their job is not to do manual regression testing and maintain excel sheets every time we are up for a release.