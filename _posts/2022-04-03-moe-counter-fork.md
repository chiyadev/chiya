---
layout: post
title: Moe-counter fork
tags: coding
---

I decided to fork the [Moe-counter][1] project and host it myself.
It is publicly available at [count.chiya.dev][2] and anyone is free to use it.

The original Moe-counter instance at [count.getloli.com][3] seems to be [very unreliable][4] because it is hosted on a free [repl.it][5] plan.
In comparison, my instance is hosted on a dedicated server running 24/7, so it should be more reliable.

Forking was necessary because the original project had hardcoded links and references to analytic scripts.
The domain was changed to `chiya.dev` and analytic scripts were removed.

I intend to keep this fork up-to-date by merging upstream changes periodically.

[1]: https://github.com/chiyadev/Moe-counter
[2]: https://count.chiya.dev
[3]: https://count.getloli.com/
[4]: https://github.com/journey-ad/Moe-counter/issues/15#issuecomment-889720686
[5]: https://replit.com/
