---
layout: post
title: "Test Suite Strategy"
description: "What makes a good test suite, and why?"
synopsis: "Yet another engineer's hot take on building fast, effective test suites"
date: 2024-03-02
tags: [software-testing]
---

Historically, the standard strategy for testing software was
[the test pyramid](https://martinfowler.com/bliki/TestPyramid.html)

More recently, developers [challenged this strategy](https://twitter.com/swyx/status/1261202288476971008),
but it's unclear whether their objections are [substantive or semantic](https://martinfowler.com/articles/2021-test-shapes.html).

This post is my hot take on what makes a good test strategy and why.

<!--more-->

First I'd like to ask:

## What do we expect from our test suite in the first place?

The primary purpose of tests is **to confirm that the code is production-ready**. Or put another way:

- Every release-blocking bug produces a failing test
- Every failing test indicates a release-blocking bug

"Change failure rate" is one of the four [DORA metrics](https://cloud.google.com/blog/products/devops-sre/announcing-dora-2021-accelerate-state-of-devops-report),
and the test suite is the most important tool for improving it. If every release-blocking bug produces a failing test, then
the change failure rate would be 0%.

The second goal as a measure of _maintenance cost_. Suppose I just want to move some code around,
rename some functions, or do other refactoring which _does not_ introduce bugs.

If refactoring causes tests to fail, I will need to update those before merging. That makes the reafactor _more expensive_
than if no tests were to fail. The more expensive a refactor is, the less likely I am to do it, and the lower quality the
code will become over time.

With that in mind:

## How do we build an effective test suite?

This depends on the size and type of the software being built.
So while there's no single answer, this is my process:

### Identify the software's "user" and "surface area"

For published packages (e.g. [Lodash](https://lodash.com/), [Guava](https://github.com/google/guava), [Cats](https://typelevel.org/cats/)),
the "surface area" is _any public method_, and the "user" is _other code running inside the same process_.

For web servers, the "surface area" is _any public endpoint_ and the "user" is _other machines across the network_.

For a full web application, the "surface area" is _the user experience_ and the "user" is _a real human_.

### Start by imagining a fully "end-to-end" test suite

To catch "bugs that impact users," the suite must _behave like a user does_.

For published packages, that means calling the public methods using your favorite unit test framework.

For web servers, it means spinning up a server process, database, etc. and making network calls with your
favorite unit test framework and HTTP client.

For full web applications, it means ~~building a robot to use the mouse and keyboard~~ using software like
Cypress or Selenium to imitate the human.

### Make compromises for practical problems

Almost all "end-to-end" test suites will hit _some_ kind of practical problems.
This section describes them, and discusses some workarounds.

#### Randomness

If the software's behavior is intentionally random, sometimes it may not be possible to find a "sweet spot"
where the test assertions _pass_ on any random value, but _fail_ in the event of release-blocking bugs.

To work around, replace the random number generator with a [test double](https://martinfowler.com/bliki/TestDouble.html).

#### System Time

Likewise, the system time will be different every time your test suite runs. If you can't find a "sweet spot"
for assertions, you'll need to use test doubles to fake the current time.

#### System timezone

Most CI runs in the cloud. By nature, the cloud could run your test suite on machines in any timezone.
In addition to appropriate assertions and test doubles, this can also be set programmatically in the
test suite's bootstrap code.

#### Network Calls

Since [the network is unreliable](https://architecturenotes.co/fallacies-of-distributed-systems/#the-network-is-reliable),
any tests which call upstream services will also be unreliable. Unreliable tests will fail when there's no release-blocking
bug, so we need a workaround.

There are two techniques here. The first, and my preferred, is to use test doubles like above. But it's worth pointing out
that another option is to _run the upstream service on the same machine_ and configure the code to call localhost during tests.

I tend to avoid running the upstream service(s) locally because:

1. The tools to define a "set" of upstream services, run them, and wait for them all to be "ready" adds a lot complexity to the test setup
2. Waiting for upstream services to come online adds much more time to the test startup
3. Network calls are much slower than test doubles (more on performance later!)
4. If your code depends on _enough_ upstream services, it may not be possible to run them all on the same machine.

#### Filesystem Calls

The filesystem is global, mutable state which lives on disk. This causes two issues:

1. Tests which read and write to the filesystem can't be run in parallel
2. Reading and writing to disk is much slower than reading and writing to memory.

If the testable application is small, it _may_ be ok for tests to read from disk directly.
If big, it's worth using test doubles so that you don't end up stuck with a slow test suite.

#### Caches and Databases

Caches and databases are global, mutable state which require a network call to interact with.
Combine the considerations from the two sections above.

A small test suite could run them locally and exercise the real code, _but_ beware that this is a
major strategic decision. As the application grows, the test suite will become slower more quickly
_and_ it will be harder to speed up by parallelizing.

#### Side effects

A test probably shouldn't bill an _actual_ credit card, or place a _real_ order for supplies from a third party company.

These types of operations _must_ be replaced by test doubles.

#### Configuration

By definition, configuration is [everything which is likely to vary between deploys](https://12factor.net/config).
This is especially challenging to test because Production will have _different values_ from local,
CI, staging, etc.

The trick here is to _separate_ tests which validate configuration from the rest of the suite, and run them
separately against a deployment _with production config values_
(e.g. a [canary deployment](https://cloud.google.com/deploy/docs/deployment-strategies/canary) before live traffic is shifted).

### Tradeoffs

#### Test doubles

Test doubles are, by definition, _not production code_. Tests which use them can fail
_if the double's defined behavior does not match production_.

Remember the core goal:

> Every failing test indicates a release-blocking bug

Suppose a test suite defines a double for some class, and we want to refactor that class by splitting it in half
and updating the callers. Even if the refactor is bug-free, the tests will still fail.

My philosophy is to _double sparingly, and with good reason_. When doubles are justified, try to double as small a function as possible.

As a general rule of thumb, I've found that doubles for

1. **Randomness & system time** are safe, but only necessary in rare cases, when the test assertions depend on them
2. **System timezone** don't cause problems, but slightly more fragile than setting it in the test bootstrap code
3. **Network, including caches & databases** are nearly always worth doubling, with the caveat that production code uses 'thin' classes or functions that don't do anything interesting besides make the network call. Some very small projects with upstream dependencies that are easy to run locally may not need to though.
4. **Filesystem** are outside my realm of experience. I just don't have experience with this kind of code.
5. **Configuration** are never worth it. Instead, run the test suite with intentionally chosen environment variables.

One common argument that I hear but _don't_ believe is that doubles "help make tests easier to write." In almost all
cases, this stems from having bad utility functions in the test code. Better to refactor the test code into better
utils so that the tests are easier to write than to start adding doubles.

#### Performance

Since the test suite's goal is to prevent user-facing bugs from being deployed, an effective test suite
needs be run on every deployment. A slow test suite will directly hurt your "Deployment Frequency"
and "Lead Time for Changes" DORA metrics.

I have seen a few ways to speed up a test suite:

1. Run the test cases in parallel
2. Write tests which are "less" end-to-end "less end-to-end" (e.g. replace Cypress with React Testing Library)
3. Partition the test suite. On code/config change, run only the _subset_ of tests which could plausibly break
4. Replace slow operations with faster test doubles

In my opinion, these are where the most intersting judgment calls get made.

**Paralellizing the test runs** makes a big improvement without sacrificing any test safety, but causes friction with any global,
mutable state. This is one of the reasons I tend to use doubles to around functions which call the database. Otherwise the infrastructure
will need to set up different databases for each worker.

**Writing "less end-to-end" tests** is situational. Most server-side applications do this to _some_ extent, by calling
the Controller functions directly, rather than making a network call. Occasionally bugs will slip through because
the HTTP framework isn't parsing requests the way the developer expects, but it's pretty rare. On the other hand,
frontend tools which "mimic" the DOM for unit test suites are _much_ less reliable than something like Cypress running in
a real browser.

**Running a subset of tests** is something that most organizations do _to some extent_. For example, suppose the client code's test suite uses doubles
for _all_ its network calls, because the team acknowledges that the network is unreliable. In that case there's _no reason_ to run the client code tests
on _server side_ code changes, because the client tests don't hit the server code anyway.

A similar tradeoff happens when a company switches from monolith to microservice. _If_ all the calling service's tests use doubles for the callee,
then there's no reason to run that suite when the callee's production code changes.

Of course, there's a catch. This layer of doubles splits the test suite in half, which does not guarantee correct behavior:

{% include image_tile.html imageurl="/images/integration-test-doors.gif" title="Two motion sensor gates which continually trigger each other to open and shut" %}

so this strategy only works well _if_ your engineers have a culture of treating backwards compatibility seriously. It also emphasizes the importance of the second principle!

> Every failing test indicates a release-blocking bug

If tests often fail alongside "safe" changes, engineers will develop a culture of "fixing tests" alongside their code. In the event
of a breaking change, those engineers will be less likely to realize that "no, seriously, _these_ tests are _actually important_" and
reconsider their changes.

Another perk here is that running subsets of tests means spending fewer CPU cycles to run the CI on each change. This
can save a lot of money on operating expenses.

**Replacing slow operations with doubles** _just_ for the sake of performance is not generally worth it.
The biggest gains will have already been achieved by using doubles for network calls, for all the reasons above.

## Counterarguments

Test strategies are contentious. That's why I described this post as a "hot take."

These are some counterarguments that I've heard occasionally, but am not convinced by.

### End-to-end suites are flaky

I think this is too vague to be helpful. I prefer to ask "_why_ is this end-to-end test flaky?"

I covered all the main causes I could remember, and listed workarounds, in the section above.
If/when I discover more, I'll update that section if I can think of a suitable workaround.

### End-to-end tests are hard to debug

The argument here is that end-to-end tests run "too much code," so when they fail it's very difficult to
find out why.

I see two actions which mitigate this argument:

1. Remove sources of flakiness, so the test suite only fails if _these_ changes broke something
2. Make smaller PRs and merge them more frequently

(2) deserves a blog post of its own... but one of its many benefits is that the root causes of failing tests
are much easier to track down.

## Summary

A well-built test suite accomplishes two goals:

- Every release-blocking bug produces a failing test
- Every failing test indicates a release-blocking bug

**Ideally** the test suite behaves like a user on production, but in practice _most software requires compromises_.
**Test doubles** can reduce flakiness when used strategically, but should be used _sparingly, and with good reason_. There
are some very pervasive _bad_ arguments to use test doubles, which are better managed by making smaller PRs and building better test utils to share code.
**Parallelization** will keep the test suite running fast as long as _global, mutable state_ is managed well.
**Partitioning the test suite** introduces some risks that bugs slip through, but it's also an easy way to save money
on CI bills and make the suite run faster.
