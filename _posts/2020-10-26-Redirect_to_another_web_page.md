---
title: Redirect to Another Web Page
tags: JavaScript HTML HowTo
---

When I was thinking about this post, I thought it would be very short and straightforward. However, it looks like there are at least 3 different ways with pros and significant pitfalls.

In general, there are 3 ways you can make a redirect.

## HTML meta

```
<meta http-equiv="refresh" content="5; url = https://bit.ly/3okeOK4" />
```

## JavaScript

```
<script type="text/javascript">
    window.location.href = "https://bit.ly/3okeOK4"
</script>
```

## Server-side redirects

```
# Apache
Redirect 301 / http://www.new-website.com
RedirectMatch 301 /blog(.*) http://www.new-website.com$1
Redirect 301 /page.html http://www.old-website/new-page.html
```

So if you'd like to know their differences and decide which one to use and when then I hope you enjoy this video.

<iframe width="560" height="315" src="https://www.youtube.com/embed/iI9fb-nKatY" frameborder="0" class="center-image" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Source code and references:

- [HTML redirect example](https://github.com/FSou1/SeasonedDeveloper/tree/main/html/redirect-to-a-page)
- [JavaScript redirect examples](https://github.com/FSou1/SeasonedDeveloper/tree/main/javascript/redirect-to-a-page)
- [301 & 302 redirect examples](https://github.com/FSou1/SeasonedDeveloper/tree/main/other/redirect-to-a-page)
- [How to Redirect a Web Page](https://css-tricks.com/redirect-web-page/)
- [http-equiv="refresh"](https://html.spec.whatwg.org/multipage/semantics.html#attr-meta-http-equiv-refresh)
