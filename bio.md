---
layout: page
title: About the author
description: A page which introduces Dave.
---

{% assign id = "school-context" %}

I'm **Dave Bemiller**, a full stack staff software engineer working remotely from Pennsylvania.
After graduating from Carnegie Mellon <noscript>in 2011</noscript> with a Physics degree,
I've <span id="{{id}}"></span><noscript>worked in</noscript> [education](https://www.ixl.com),
[finance](https://www.creditkarma.com), [advertising](https://www.xandr.com), and
[healthcare](https://carbonhealth.com).

My main academic interests are [law](https://popehat.substack.com/),
[finance](https://www.amazon.com/Adaptive-Markets-Financial-Evolution-Thought/dp/0691135142),
[psychology](https://www.amazon.com/Behave-Biology-Humans-Best-Worst/dp/009957506X),
[design](https://www.coursera.org/specializations/ui-ux-design), and
of course [software engineering](https://www.amazon.com/dp/B08RMSHYGG). My other hobbies are
hiking, snowboarding, and video games, depending on the weather and time of year.

My wife and I live with three photogenic fluffs:

<section class="tiled-images">
{% include image_tile.html imageurl="/images/duke-of-fluffington.jpg" title="Fluffy cat" caption="The Duke of Fluffington" %}
{% include image_tile.html imageurl="/images/spider-cat.jpg" title="Climbing cat" caption="Spider Cat" %}
{% include image_tile.html imageurl="/images/seductive-fee.jpg" title="Fat cat" caption="The Most Interesting Cat in the World" %}
</section>

<script async type="text/javascript">
(function() {
    const now = new Date();
    const thisYear = now.getFullYear();
    const yearsSince2011 = thisYear - 2011;
    document.getElementById("{{id}}").innerText = "spent " + yearsSince2011 + " years engineering for";
})();
</script>