---
layout: post
title: "Effective Testing"
description: "What makes a good test suite, and why?"
synopsis: "Yet another hot take on what it takes to build fast, effective test suites"
date: 2024-03-02
tags: [software-testing]
---

Historically, the standard strategy for testing software was
[the test pyramid](https://martinfowler.com/bliki/TestPyramid.html). More recently, developers
[challenged this strategy](https://twitter.com/swyx/status/1261202288476971008), but it's unclear
whether their objections are [substantive or semantic](https://martinfowler.com/articles/2021-test-shapes.html).

This is my hot take on building an effective test suite.

<!--more-->

## What do we expect from our test suite in the first place?

The primary purpose of tests is **to confirm that the code is production-ready**. In other words:

- Every release-blocking bug produces a failing test
- Every failing test indicates a release-blocking bug

If every release-blocking bug produces a failing test, then the project's DORA
[change failure rate](https://cloud.google.com/blog/products/devops-sre/announcing-dora-2021-accelerate-state-of-devops-report) is 0%.

If tests fail when there's no release-blocking bug, then the suite creates _headwinds_ which slow down development.
Engineers need to spend time investigating and "fixing" those tests while they build out new features.

## How do we build an effective test suite?

This depends on the size and type of the software being built.
While there's no single answer, this process has worked pretty well for me.

### Identify the software's "user" and "surface area"

For published packages (e.g. [Lodash](https://lodash.com/), [Guava](https://github.com/google/guava), [Cats](https://typelevel.org/cats/)),
the "surface area" is _any public method_, and the "user" is _other code running inside the same process_.

For web servers, the "surface area" is _any public endpoint_ and the "user" is _other machines across the network_.

For a full web application, the "surface area" is _the user experience_ and the "user" is _a real human_.

### Imagine a perfect "end-to-end" test suite

To catch "bugs that impact users," the suite must _behave like a user does_.

For published packages, that means calling the public methods using your favorite unit test framework.

For web servers, it means spinning up a server process, database, etc. and making network calls with your
favorite unit test framework and HTTP client.

For full web applications, it means building a robot to use the mouse and keyboard.

### Make compromises for practical problems

Almost all "end-to-end" test suites for even simple projects will hit _some_ kind of practical problems.

#### Robots are science fiction

As of 2024, we do not yet have robots that can use the mouse and keyboard and interpret the program's
behavior like a human would.

The best we can do is use tools like Cypress, Selenium, Webdriver.io, etc.

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

- It complicates the test setup. Something needs to define the "set" of upstream services, run them, and wait for them all to be "ready" before executing tests
- These complications make the test startup much slower
- Network calls are much slower than test doubles, which slows down the suite
- If your code depends on _enough_ upstream services, it may not be possible to run them all on the same machine.

That's not to say it's _never_ a good idea to run upsream services locally--just that I don't find the juice worth the squeeze.

#### Filesystem Calls

The filesystem is global, mutable state on disk. This causes two issues:

- Tests which read and write to global, mutable state can't be run in parallel
- Reading and writing to disk is much slower than reading and writing to memory

If the testable application is small, it _may_ be ok for tests to read from disk directly.
Just beware that this is a major strategic decision! As the application grows, it will become
slower at a smaller size _and_ it will be harder to speed up by parallelizing the suite.

#### Caches and Databases

Caches and databases are global, mutable state which require a network call to interact with.
Combine the considerations from the two sections above, and amplify the effect because the
network is slower than disk.

#### Side effects

A test probably shouldn't bill an _actual_ credit card, or place a _real_ order for supplies from a third party company.

Operations like these _must_ be replaced by test doubles.

#### Configuration

By definition, configuration is [everything which is likely to vary between deploys](https://12factor.net/config).
This is especially challenging to test because Production will have _different values_ from local,
CI, staging, etc.

The trick here is to _separate_ tests which validate configuration from the rest of the suite, and run them
separately against a deployment _with production config values_
(e.g. a [canary deployment](https://cloud.google.com/deploy/docs/deployment-strategies/canary) before live traffic is shifted).

### Tradeoffs

#### Test doubles

Test doubles are, by definition, _not production code_. Tests which use them may fail
_if the double's defined behavior does not match production_.

This brushes up against to the core goal:

> Every failing test indicates a release-blocking bug

Suppose I make a PR which splits one large class into two. The code is cut/pasted,
and all callers are updated. Assume I did this properly and no bugs were introduced.

Now imagine what happens if a test defined a mock for that class. That test would fail on my _safe_ refactor,
which I now need to "fix" before merging. In other words, the double has made refactoring more expensive.
The more expensive a refactor is, the less likely I am to do it, which means lower quality code over time.

My philosophy is to _double sparingly, and with good reason_. When doubles are justified, try to organize the live code
so that you can double as small a function as possible. e.g. write a layer of "thin" classes or functions for outbound
network calls.

#### Performance

To prevent user-facing bugs from being deployed, effective tests need be run on every deployment.
A slow test suite will directly hurt your "Deployment Frequency" and "Lead Time for Changes" DORA metrics.

I know of a few ways to speed up a test suite:

- Run the test cases in parallel
- Write tests which are "less" end-to-end "less end-to-end" (e.g. replace Cypress with React Testing Library)
- Partition the test suite. On code/config change, run only the _subset_ of tests which could plausibly break
- Replace slow operations with faster test doubles

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
The biggest gains come from using doubles for network calls, which is already generally justified for the
other reasons described earlier.

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

I see two ways to mitigate this argument:

- Remove sources of flakiness, so the test suite only fails if _these_ changes broke something
- Make smaller PRs and merge them more frequently

"Smaller PRs" deserve a blog post on their own. One of their many benefits is that it makes the root cause
of failing tests much easier to track down, even if the tests themselves cover a very large amount of code.

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
