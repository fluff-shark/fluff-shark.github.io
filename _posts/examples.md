---
layout: post
title: "Some title"
description: "Some description."
date: 2019-01-01
feature_image: images/author.jpg 
tags: [post, , tags, here]
published: false
---

Some useful markdown/html snippets from the template posts for future reference:

Anything above the comment below will show up on the main page.

<!--more-->

People need to click "continue reading" to see content on this line and below.

A full-sized picture looks like this:

{% include image_full.html imageurl="/images/author.jpg" title="Some title" caption="Caption appears here" %}

Quotes with citations look like:

>“Returning home is the most difficult part of long-distance hiking; You have grown outside the puzzle and your piece no longer fits.”
><cite>― Cindy Ross</cite>

[An inline-style link with title](https://www.google.com "Google's Homepage")

```css
#header h1 { 
  color: #fff;
  margin-bottom: 1.5em; 
}
```

```javascript
// Simple map
var map;
function initMap() {
  map = new google.maps.Map(document.getElementById('map'), {
    center: {lat: -34.397, lng: 150.644},
    zoom: 8
  });
}
```

```json
{
  "menu": {
    "id": "file",
    "value": "File",
    "popup": {
      "menuitem": [
        {"value": "New", "onclick": "CreateNewDoc()"}
      ]
    }
  }
}
```

```yml
sass:
  input_file: sass/main.scss.njk
  output_file: assets/css/main.css
```

Inline external video:

<iframe src="https://player.vimeo.com/video/153339497?byline=0" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
