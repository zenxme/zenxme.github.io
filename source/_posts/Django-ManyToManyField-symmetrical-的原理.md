---
title: Django ManyToManyField symmetrical 的原理
date: 2019-11-27 05:43:54
tags:
  - python
  - django
---

对Django中ManyToManyField的symmetrical参数的实现进行分析。

<!-- more -->

首先来看官方文档 [ManyToManyField.symmetrical](https://docs.djangoproject.com/en/2.2/ref/models/fields/#django.db.models.ManyToManyField.symmetrical)

>Only used in the definition of ManyToManyFields on self. Consider the following model:
>
>```python
>from django.db import models
>
>class Person(models.Model):
>         friends = models.ManyToManyField("self")
>```
>When Django processes this model, it identifies that it has a ManyToManyField on itself, and as a result, it doesn’t add a person_set attribute to the Person class. Instead, the ManyToManyField is assumed to be symmetrical – that is, if I am your friend, then you are my friend.
>
>If you do not want symmetry in many-to-many relationships with self, set symmetrical to False. This will force Django to add the descriptor for the reverse relationship, allowing ManyToManyField relationships to be non-symmetrical.

symmetrical字面意思为对称，文档里也说了，这个参数仅仅在ManyToManyFields指向self的时候有用。

而举的例子也很简单明了，如果我是你的朋友，那么你也是我的朋友，这是默认的情况。如果不想用这种默认情况，那么就需要手动把它设置为False。

那么这种默认情况下，django到底是怎么实现的？首先一个中间表应该是必须的。

另外我猜到可能有2种实现：

- 每次都存储2份数据，比如存一列用户a是b的朋友，再存一列b是a的朋友
- 每次只存1份，把用户id较小的放到第一列。

下面写代码看下数据库是怎么存的：
```python
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    friends = models.ManyToManyField("self")
```
使用`python manage.py shell` 进入django的shell
```python
>>> from user.models import User
>>> foo = User()
>>> foo.username = 'foo'
>>> foo.set_password('123')
>>> foo.save()
>>> bar = User()
>>> bar.username = 'bar'
>>> bar.set_password('123')
>>> bar.save()
>>> foo.friends.add(bar)
>>> foo.friends.all()
<QuerySet [<User: bar>]>
>>> bar.friends.all()
<QuerySet [<User: foo>]>

>>> tom = User()
>>> tom.username = 'tom'
>>> tom.set_password(123)
>>> tom.save()
>>> tom.friends.add(foo)
>>> foo.friends.all()
<QuerySet [<User: bar>, <User: tom>]>
```


```sqlite
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .tables
auth_group                  django_session            
auth_group_permissions      user_user                 
auth_permission             user_user_friends         
django_admin_log            user_user_groups          
django_content_type         user_user_user_permissions
django_migrations

sqlite> .schema user_user_friends
CREATE TABLE IF NOT EXISTS "user_user_friends" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "from_user_id" integer NOT NULL REFERENCES "user_user" ("id") DEFERRABLE INITIALLY DEFERRED, "to_user_id" integer NOT NULL REFERENCES "user_user" ("id") DEFERRABLE INITIALLY DEFERRED);
CREATE UNIQUE INDEX "user_user_friends_from_user_id_to_user_id_1f3f3c7e_uniq" ON "user_user_friends" ("from_user_id", "to_user_id");
CREATE INDEX "user_user_friends_from_user_id_317b081e" ON "user_user_friends" ("from_user_id");
CREATE INDEX "user_user_friends_to_user_id_c77c2cf7" ON "user_user_friends" ("to_user_id");

sqlite> .header on

sqlite> select * from user_user_friends;
id|from_user_id|to_user_id
1|1|2
2|2|1
3|3|1
4|1|3
```

可以看到django生成了一个名字叫user_user_friends的中间表，而且从建表语句可以看到有3个id列，分别是1个自增主键和2个外键。

查看数据可以发现是之前的第一种猜想。

在调用` foo.friends.add(bar)`的时候，插入了两行数据`(1, 2)`和`(2, 1)`。

在调用`tom.friends.add(foo)`的时候，插入了两行数据`(3, 1)`和`(1, 3)`。

每次先插入的行都是调用者的id在前。

在经过了一段时间的查找之后，在django的源码里也找到了相关的代码，印证了之前的看法。

`django/db/models/fields/related_descriptors.py`里`ManyRelatedManager`类的部分代码如下：

```python
        def add(self, *objs, through_defaults=None):
            self._remove_prefetched_objects()
            db = router.db_for_write(self.through, instance=self.instance)
            with transaction.atomic(using=db, savepoint=False):
                self._add_items(
                    self.source_field_name, self.target_field_name, *objs,
                    through_defaults=through_defaults,
                )
                # If this is a symmetrical m2m relation to self, add the mirror
                # entry in the m2m table. `through_defaults` aren't used here
                # because of the system check error fields.E332: Many-to-many
                # fields with intermediate tables must not be symmetrical.
                if self.symmetrical:
                    self._add_items(self.target_field_name, self.source_field_name, *objs)
```

可以看到是在一个事务里面进行的，调用了两次`self._add_items`，第一次是`source_field_name`在前，第二次是`target_field_name`在前。

django默认的这种行为虽然也不错，但是会占用2倍的空间，如果用户量越来越大的时候劣势会更明显。但是暂时还不太清楚django默认这种机制的道理。

在目前的我看来，更倾向于第二种想法，就是在插入之前先判断id的大小，每次都使得小的id在前，这样可以节省空间。所以我一般都会把symmetrical设为False。