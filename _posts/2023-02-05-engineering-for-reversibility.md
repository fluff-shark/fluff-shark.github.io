---
layout: post
title: "Engineering for Reversibility"
description: "Applying Jeff Bezos' decision making advice to software engineering"
synopsis: "Jeff Bezos divided decisions into two categories. Most decisions are reversible, and should be made by high judgment individuals or small groups. A few decisions are irreversible, and should be made carefully with great deliberation. In software engineering, reversibility is a spectrum. This post describes how decisions can be made more reversible with the right processes, tools, and culture."
date: 2023-02-05
tags: [software-engineering]
---

In a [letter to Amazon's shareholders](https://www.sec.gov/Archives/edgar/data/1018724/000119312516530910/d168744dex991.htm),
Jeff Bezos wrote:

> Some decisions are consequential and irreversible or nearly irreversible – one-way doors
> – and these decisions must be made methodically, carefully, slowly, with great deliberation
> and consultation. If you walk through and don’t like what you see on the other side, you
> can’t get back to where you were before. We can call these Type 1 decisions. But most decisions
> aren’t like that – they are changeable, reversible – they’re two-way doors. If you’ve made a
> suboptimal Type 2 decision, you don’t have to live with the consequences for that long.
> You can reopen the door and go back through. Type 2 decisions can and should be made quickly
> by high judgment individuals or small groups.

This put two questions on my mind:

1. Which software engineering decisions are reversible or irreversible?
2. How can we make our decisions _more reversible_?

<!--more-->

### Changing Server Implementation

In principle, changing code that runs on servers is a clear Type 2 decision.
It takes _two clicks_ for me to update this site, and a more specialized git
UI could easily reduce that to one.

In practice, there's almost always _some_ kind of friction to deployments.
The two most common examples are pull requests and code reviews.

Organizations can encourage Type 2 thinking by:

- **Investing in continuous deployment**: The faster and more automated the deployments, the
more quickly code can be reverted or patched if necessary.
- **Minimizing deployment friction**: Some friction, like code reviews, add enough value to
be worth the tradeoff. But friction should be added sparingly, and with good reason.
The more friction, the more engineers will start to see code changes as a Type 1 decision.

### Changing Client Implementation

Client code is more difficult to change than server code, for a few reasons:

- Callers may have cached your server's responses, and now need the TTL to expire.
- Users can leave their browser tab open for a long time, so they won't fetch your new code.

App code has even more concerns:

- Google/Apple take _days_ to vet your new app code, even when things go smoothly.
- Users may choose not to update their apps, once the new version is released.

Client code updates _can be_ Type 1, but we can minimize the risk by:

- **Using [feature flags](https://www.atlassian.com/continuous-delivery/principles/feature-flags)**:
Feature flags can be used to roll back instantly if new client code causes problems in production.
- **Setting Cache-Control wisely**: Be deliberate about your
[Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) header choices.
If a TTL on a file is long enough, then updating that file becomes a Type 1 decision.
- **Supporting live app updates**: For Android apps, support
[in-app updates](https://developer.android.com/guide/playcore/in-app-updates/kotlin-java). For iOS,
support [forced app updates](https://betterprogramming.pub/force-update-your-apps-74de57523650).
These can upset users, so be careful not to overuse them, but they're a lifesaver if you have a
critical bug fix or emergency.

### Changing Data

Moving data or updating its format _without downtime_ involves:

1. Updating "read" code to pull from the new format, with fallback logic to the old one.
2. Updating "write" code to target the new format rather than the old one.
3. Running an ETL to migrate the existing data.
4. Deleting the "read" code's fallback logic.
5. Dropping all the old data.

The difficulty of this process depends on:

- How many places in the code directly read/write the data
- How many objects need to be migrated.

Some best practices here are:

- **Consolidate the read/write code as much as possible**: The fewer places that the code touches
the data source, the easier (1) and (2) will be.
- **If you expect lots of data, treat the model as a Type 1 decision**: The ETL in (3) has a
best case runtime of `O(numObjects)`. If there will be many objects, transforms will become
prohibitively expensive.

### Changing Contracts

A "contract" is _any agreement between your code and those who call it_. This includes:

- Function arguments and return values
- Service API requests and responses
- Documentation or tooling which either describes:
    - What your code requires from its callers
    - How your code will behave when called

Updating a contract involves:

1. Adding a new function or endpoint to implement the new behavior
2. Updating all callers to use the new function or endpoint
3. Deleting the old function or endpoint.

The difficulty of this process depends on:

- How many callers will there be?
- How much control do you have over those callers?

{% include image_tile.html imageurl="/images/reversible-api-graph.jpg" title="Graph showing reversible contracts" %}

Type 2 contracts live in the upper left quadrant. Type 1 contracts live in the lower right.
The upper right is somewhere in between, depending on the size of your organization.

Some best practices here are:

- **Minimize access levels**: `private` contracts are firmly a Type 2 decision. `public` ones
may not be, depending on the context. It's very easy to change a `private` one to a `public`,
but updating or removing a `public` may be very expensive.
- **Enlist beta testers** for Type 1 contracts. Their feedback will be valuable, the beta period
lets you iterate before making a full Type 1 commitment.

### Conclusion

As a software engineer, the key lesson I took from Bezos' advice is this:

> Reversibility is a spectrum. Decisions can be made _more reversible_ with the
> right tools, processes, and culture. Adopting these will empower teams and individuals
> to move faster and act more with more autonomy.
