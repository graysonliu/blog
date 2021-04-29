---
layout: post
title: "Python Formatting in VS Code: Force Whitespace around Arithmetic Operator"
categories: [python, vscode, coding]
comments: true
---

In VS Code, you can use `Ctrl+Shift+I` (Linux) to format a document by [default shortcuts](https://code.visualstudio.com/shortcuts/keyboard-shortcuts-linux.pdf). For python, you need to install some formatter packages like `autopep8` or `black`.

I use `autopep8`. Install it by `pip install autopep8`. By default, `autopep8` won't fixing missing whitespace around arithmetic operators. For example, if one line of your python code is `a = 1+2`, after using `Ctrl+Shift+I`, `autopep8` won't insert missing whitespace around the plus sign. If you want this line to be formatted into `a = 1 + 2`, you have to tweak the settings of `autopep8`.

According to [the `autopep8` page](https://pypi.org/project/autopep8/), four errors/warning are ignored by default:

> `--ignore errors`
>
> do not fix these errors/warnings (default:E226,E24,W50,W690)

And `E226` is `fix missing whitespace around arithmetic operator`. Therefore, we have to bring `E226` back. We can use `--select` options to tell `autopep8` which errors and warnings should be fixed. In VS Code settings, search for `autopep8`, and add following two arguments:

![]({{ site.baseurl }}/images/autopep8_args.png)

This means `autopep8` will fix all code style errors and warnings, including previously ignored `E226`. Now, `Ctrl+Shift+I` will force whitespace around arithmetic operators.