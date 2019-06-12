---
title: '''sudo -H''的作用'
date: 2019-03-19 10:54:41
tags:
  - bash
---

在用pip安装python库时候，经常会看到警告`"The directory or its parent directory is not owned by the current user and caching wheels has been disabled..........`，然后就是建议使用`sudo -H`。那么这个问题的具体原因到底是什么呢？

<!-- more -->

## 查阅sudo的文档

查阅：<https://linux.die.net/man/8/sudo>，其中对-H参数的解释如下：

```
-H' The -H (HOME) option requests that the security policy set the HOME environment variable to the home directory of the target user (root by default) as specified by the password database. Depending on the policy, this may be the default behavior.
```

意思就是加了-H的参数以后会设置`HOME`这个环境变量到root的目录（默认情况下），那么我们手动测试一下：

```bash
sudo python3
```

```python
Python 3.6.7 (default, Oct 22 2018, 11:32:17) 
[GCC 8.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.environ['HOME']
'/home/hacker'
>>> os.environ['USER']
'root'
```

加了-H参数之后

```bash
sudo -H python3
```

```python
Python 3.6.7 (default, Oct 22 2018, 11:32:17) 
[GCC 8.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.environ['HOME']
'/root'
>>> os.environ['USER']
'root'
```

所以结果就很明显了，确实环境变量里的`HOME`被改成了root的目录。

## 查看pip源码

那么到底pip为什么需要这个变量呢，我查阅了位于github上的pip的源码，在这里<https://github.com/pypa/pip/blob/3a77bd667cc68935040563e1351604c461ce5333/src/pip/_internal/commands/download.py>我找到了这段提示信息。

```python
            if options.cache_dir and not check_path_owner(options.cache_dir):
                logger.warning(
                    "The directory '%s' or its parent directory is not owned "
                    "by the current user and caching wheels has been "
                    "disabled. check the permissions and owner of that "
                    "directory. If executing pip with sudo, you may want "
                    "sudo's -H flag.",
                    options.cache_dir,
                )
                options.cache_dir = None
```

根据代码变量的命名，很容易就可以知道这段代码的意思。如果说缓存目录存在，但是不属于当前用户的话，就发出警告，并且不缓存wheels。