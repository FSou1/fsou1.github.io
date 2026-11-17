---
title: Friday Quiz or Emoji are weird in JavaScript
tags: JavaScript Emoji Length
description: "Why '👩‍👩‍👧‍👧'.length is 11 in JavaScript, and why different emoji report different string lengths."
---

Can you guess, what will be the output `'👩‍👩‍👧‍👧'.length // ?`?

It is `11`.

What about this one: `['☺', '😀', '1️⃣', '🇦🇨', '👱‍♂️', '👩‍❤️‍👨', '👩‍👩‍👧‍👧'].map((emoji) => emoji.length)`?

Well, here it goes: `[1, 2, 3, 4, 5, 8, 11]`.

My favorite two examples look like marriage and divorce:

```
['👨', '‍', '👩'].reduce((cur, next) => cur += next) // 👨‍👩

[...'👨‍👩‍👧']   // ['👨', '‍', '👩', '‍', '👧']
```

I've group every emoji by its length, so you don't have to [https://github.com/FSou1/EmojiGroupedByLength](https://github.com/FSou1/EmojiGroupedByLength).

Happy Friday.

More theory: [https://gomakethings.com/emoji-are-still-weird-but-modern-browser-methods-help/](https://gomakethings.com/emoji-are-still-weird-but-modern-browser-methods-help/)
