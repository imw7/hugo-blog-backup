---
title: Hugo使用指南
tags: [Hugo, 常用组件和技巧]
categories: [Hugo]
abbrlink: hugo_guide
date: 2022-07-01T21:23:51+08:00
toc: false
---

本文介绍的是在 `Hugo` 的使用过程中遇到的一些问题。<!--more-->

## Hugo处理数学公式

添加 `MathJax` 以便对 `Hugo` 解析公式时提供支持。把以下内容写入 `layouts/partials/mathjax.html` 文件中：

```html
<script type="text/javascript"
        async
        src="https://cdn.bootcss.com/mathjax/2.7.3/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
MathJax.Hub.Config({
  tex2jax: {
    inlineMath: [['$','$'], ['\\(','\\)']],
    displayMath: [['$$','$$'], ['\[\[','\]\]']],
    processEscapes: true,
    processEnvironments: true,
    skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
    TeX: { equationNumbers: { autoNumber: "AMS" },
         extensions: ["AMSmath.js", "AMSsymbols.js"] }
  }
});

MathJax.Hub.Queue(function() {
    // Fix <code> tags after MathJax finishes running. This is a
    // hack to overcome a shortcoming of Markdown. Discussion at
    // https://github.com/mojombo/jekyll/issues/199
    var all = MathJax.Hub.getAllJax(), i;
    for(i = 0; i < all.length; i += 1) {
        all[i].SourceElement().parentNode.className += ' has-jax';
    }
});
</script>

<style>
code.has-jax {
    font: inherit;
    font-size: 100%;
    background: inherit;
    border: inherit;
    color: #515151;
}
</style>
```

然后在 `layouts/partials/footer.html` 文件头部写入下面的代码，因为我知道我的每篇博客都有 `footer`。

```html
{{ partial "mathjax.html" . }}
```

**注意**：上面的语句最后的括号前面有个 `.`

这样 `Hugo` 就能正常解析数学公式了。