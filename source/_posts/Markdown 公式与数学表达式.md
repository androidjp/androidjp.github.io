---
title: Markdown 公式与数学表达式
date: 2019-09-27 16:56:49
tags:
- Markdown
mathjax: true
---

教你如何用Markdown写公式！！
<!-- more -->

> [在线MD公式生成器](https://www.codecogs.com/latex/eqneditor.php)
>
> 如果是遇到一些MD编辑器是不支持公式的，不妨可以借助MathJax引擎:
> 1. 在markdown写作首部添加如下代码：
> ```
> <script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>
> ```
> 2. 之后就可以像在Latex或者Overleaf里面一样写公式喽


更复杂的公式写法，请参考：[这里](https://www.zybuluo.com/codeep/note/163962)，如果这个link出问题了，请看印象笔记，已剪辑。

# 数学表达式的写法

写一个累加表达式：$f(x)=\sum\_{i=1}^n{k_i}$


常见公式汇总：

  | 运算符          | 说明             | 案例                                            | 写法实现                                          |
  | ---             | ---             | ---                                             | ---                                              |
  | `\times`        | 乘               | `x\times{y}`                                   | `$x\times{y}$`                                   |
  | `\cdot`         | 乘               | `x\cdot{y}`                                    | `$x\cdot{y}$`                                    |
  | `\ast`          | 乘               | `x\ast{y}`                                     | `$x\ast{y}$`                                     |
  | `\div`          | 除               | `x\div{y}`                                     | `$x\div{y}$`                                     |
  | `\frac`         | 分数             | `\frac{x}{y}`                                  | `$\frac{x}{y}$`                                  |
  | `^`             | 上标             | `x^y`                                          | `$x^y$`                                          |
  | `_`             | 下标             | `x_y`                                          | `$x_y$`                                          |
  | `\sqrt`         | 开二次方         | `\sqrt{x}`                                     | `$\sqrt{x}$`                                     |
  | `\sqrt`         | 开方             | `\sqrt[x]{y^4+3y-1}`                           | `$\sqrt[x]{y^4+3y-1}$`                           |
  | `\pm`           | 加减             | `x\pm{y}`                                      | `$x\pm{y}$`                                      |
  | `\mp`           | 减加             | `x\mp{y}`                                      | `$x\mp{y}$`                                      |
  | `\leq`          | 小于等于         | `x\leq{y}`                                     | `$x\leq{y}$`                                     |
  | `\geq`          | 大于等于         | `x\geq{y}`                                     | `$x\geq{y}$`                                     |
  | `\ngeq`         | 不大于等于       | `x\ngeq{y}`                                    | `$x\ngeq{y}$`                                    |
  | `\not\geq`      | 不大于等于       | `x\not\geq{y}`                                 | `$x\not\geq{y}$`                                 |
  | `\neq`          | 不等于           | `x\neq{y}`                                     | `$x\neq{y}$`                                     |
  | `\approx`       | 约等于           | `x\approx{y}`                                  | `$x\approx{y}$`                                  |
  | `\equiv`        | 恒等于           | `x\equiv{y}`                                   | `$x\equiv{y}$`                                   |
  | `\bigodot`      | 定义运算符       | `x\bigodot{y}=x+y^2`                           | `$x\bigodot{y}=x+y^2$`                           |
  | `\bigotimes`    | 定义运算符       | `x\bigotimes{y}=x+y^2`                         | `$x\bigotimes{y}=x+y^2$`                         |
  | `\in`           | 属于             | `x\in{y}`                                      | `$x\in{y}$`                                      |
  | `\notin`        | 不属于           | `x\notin{y}`                                   | `$x\notin{y}$`                                   |
  | `\subset`       | 子集             | `x\subset{y}`                                  | `$x\subset{y}$`                                  |
  | `\not\subset`   | 非子集           | `x\not\subset{y}`                              | `$x\not\subset{y}$`                              |
  | `\subseteq`     | 子集             | `x\subseteq{y}`                                | `$x\subseteq{y}$`                                |
  | `\supset`       | 超集             | `x\supset{y}`                                  | `$x\supset{y}$`                                  |
  | `\supseteq`     | 超集             | `x\supseteq{y}`                                | `$x\supseteq{y}$`                                |
  | `\cup`          | 并               | `x\cup{y}`                                     | `$x\cup{y}$`                                     |
  | `\cap`          | 交               | `x\cap{y}`                                     | `$x\cap{y}$`                                     |
  | `\log`          | 对数             | `\log(x)`                                      | `$\log(x)$`                                      |
  | `\overline`     | 平均数           | `\overline{x}`                                 | `$\overline{x}$`                                 |
  | `\overline`     | 连线符号         | `\overline{a+b+c+d}`                           | `$\overline{a+b+c+d}$`                           |
  | `\underline`    | 下划线           | `\underline{a+b+c+d}`                          | `$\underline{a+b+c+d}$`                          |
  | `\overbrace`    | 上大括号         | `\overbrace{a+\underbrace{b+c}\_{1.0}+d}^{2.0}`| `$\overbrace{a+\underbrace{b+c}_{1.0}+d}^{2.0}$` |
  | `\underbrace`   | 下大括号         | `\underbrace{a+d}\_3`                          | `$\underbrace{a+d}_3$`                           |
  | `\partial`      | 部分             | `\frac{\partial x}{\partial y}`                | `$\frac{\partial x}{\partial y}$`                |
  | `\lim`          | 极限             | `\lim\_{x\to\infty}`                           | `$\lim_{x\to\infty}$`                            |
  | `\displaystyle` | 块公式格式       | `\displaystyle\lim\_{x\to\infty}`              | `$\displaystyle\lim_{x\to\infty}$`               |
  | `\sum`          | 求和             | `\sum_1^n`                                     | `$\sum_1^n$`                                     |
  | `\infty`        | 极限             | `\sum\_{i=0}^\infty{i^2}`                      | `$\sum_{i=0}^\infty{i^2}$`                       |
  | `\int`          | 积分             | `\int_0^1{x^2}{dx}`                            | `$\int_0^1{x^2}{dx}$`                            |
  | `\ldots`        | 底端对齐的省略号 | `1,2,\ldots,n`                                 | `$1,2,\ldots,n$`                                 |
  | `\cdots`        | 中线对齐的省略号 | `x_1^2 + x_2^2 + \cdots + x_n^2`               | `$x_1^2 + x_2^2 + \cdots + x_n^2$`               |
  | `\uparrow`      | 上箭头           | `\uparrow`                                     | `$\uparrow$`                                     |
  | `\Uparrow`      | 上箭头           | `\Uparrow`                                     | `$\Uparrow$`                                     |

# 数学符号输出

  | 数学符号  | 写法 |
  | --- | --- |
  | `\alpha`           | `$\alpha$`   |
  | `\beta`            | `$\beta$`    |
  | `\gamma`           | `$\gamma$`   |
  | `\delta`           | `$\delta$`   |
  | `\epsilon`         | `$\epsilon$` |
  | `\zeta`            | `$\zeta$`    |
  | `\eta`             | `$\eta$`     |
  | `\theta`           | `$\theta$`   |
  | `\iota`            | `$\iota$`    |
  | `\kappa`           | `$\kappa$`   |
  | `\lambda`          | `$\lambda$`  |
  | `\mu`              | `$\mu$`      |
  | `\Xi`              | `$\Xi$`      |
  | `\omicron`         | `$\omicron$` |
  | `\pi`              | `$\pi$`      |
  | `\rho`             | `$\rho$`     |
  | `\sigma`           | `$\sigma$`   |
  | `\chi`             | `$\chi$`     |
  | `\upsilon`         | `$\upsilon$` |
  | `\phi`             | `$\phi$`     |
  | `\psi`             | `$\psi$`     |
  | `\omega`           | `$\omega$`   |


## 相关快捷键

1.  喊出命令提示：`ctrl+shift+P / ctrl+ shift+ A`
2.  format 内容：`Format Document`
3.  加粗：`Ctrl + B`
4.  倾斜：`Ctrl + I`
5.  导出 PDF：`Convert Markdown to PDF`
6.  导出 Html：`Markdown: Print current document to HTML`
7.  导出支持公式显示的 Html：`Markdown: Clip Markdown+Math to HTML`
