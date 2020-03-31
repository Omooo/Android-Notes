---
Long
---

为什么使用 Long 时，大家推荐多使用 valueOf 方法，少使用 parseLong 方法？

因为 Long 本身有缓存机制，缓存了 -128 到 127 范围内的 Long，valueOf 方法会从缓存中去拿值，如果命中缓存，会减少资源的开销，parseLong 方法就没有这个机制。

