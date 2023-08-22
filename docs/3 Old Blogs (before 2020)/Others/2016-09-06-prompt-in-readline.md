---
layout:     post
title:      "在readline中使用颜色prompt导致显示bug"
subtitle:   "在C语言中`readline`，python中`raw_input`和`input`中，prompt中如果直接使用颜色会出现bug。显示错位并覆盖。"
date:       2016-09-06 12:00:00
author:     "Readm"
header-img: "img/post-bg-2015.jpg"
tags:
    - 技术
    - C
    - Python
    - Bug
---

# 在readline中使用颜色prompt导致显示bug

在C语言中`readline`，python中`raw_input`和`input`中，`prompt`中如果直接使用颜色会出现bug。显示错位并覆盖。 这是因为在`readline`中颜色字符占用了长度，计算出错。

解决方法： 在`readline.h`中有如下定义：

````
/* Definitions available for use by readline clients. */

define RL_PROMPT_START_IGNORE '\001'
define RL_PROMPT_END_IGNORE '\002'
```

所以应该在颜色字符的前后加入`'\001'`和`'\002'`。

注意：

当非prompt中加入这些字符时会出现乱码(ubuntu中已验证)，故在其他输出中不应使用这一方法。