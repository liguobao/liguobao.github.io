---
title: Math Typesetting
description: Math typesetting using KaTeX
date: 2023-08-24 00:00:00+0000
math: true
---

Stack supports math typesetting via KaTeX. Enable per post with `math: true` in front matter, or site‑wide by setting `params.article.math = true` in `config.toml`.

Inline math: $\\varphi = \\dfrac{1+\\sqrt5}{2}= 1.6180339887…$

Block math:

$$
    \\varphi = 1+\\frac{1} {1+\\frac{1} {1+\\frac{1} {1+\\cdots} } }
$$

$$
    f(x) = \\int_{-\\infty}^\\infty\\hat f(\\xi)\\,e^{2 \\pi i \\xi x}\\,d\\xi
$$

