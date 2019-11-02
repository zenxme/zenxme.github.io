---
title: Django Rest framework 在自定义serializer的Field的时候无法传递None
date: 2019-11-2 11:58:15
tags:
  - python
  - django
---

在使用Django Rest framework框架的时候遇到的一个问题

<!-- more -->

有一个`UserSerializer`的类，我想当没有默认的`avatar_url`的时候返回一个默认的`url`地址，于是写了这样的一个`Field`：

```python
class UserAvatarField(serializers.CharField):
    def to_representation(self, value):
        if value:
            return super().to_representation(value)
        else:
            return get_default_user_avatar_url()
```

然后在定义serializer的时候这么用：

```python
class UserSerializer(serializers.ModelSerializer):
    avatar_url = UserAvatarField(max_length=255)
    # ......
    class Meta:
        model = User
        # ......
```

结果发现，当`User`实例的`avatar_url`属性有数据的时候可以正常返回数据，但是当它为`None`的时候，就不会进入`Field`的`to_represtation`方法，于是去查看了下父类的源码：

```python
    def to_representation(self, instance):
        """
        Object instance -> Dict of primitive datatypes.
        """
        ret = OrderedDict()
        fields = self._readable_fields

        for field in fields:
            try:
                attribute = field.get_attribute(instance)
            except SkipField:
                continue

            # We skip `to_representation` for `None` values so that fields do
            # not have to explicitly deal with that case.
            #
            # For related fields with `use_pk_only_optimization` we need to
            # resolve the pk value.
            check_for_none = attribute.pk if isinstance(attribute, PKOnlyObject) else attribute
            if check_for_none is None:
                ret[field.field_name] = None
            else:
                ret[field.field_name] = field.to_representation(attribute)

        return ret
```

通过这句注释`We skip  to_representation for  None values so that fields do not have to explicitly deal with that case.`以及下面的代码可以很容易看出来，如果为`None`的时候就不会传递给`feild`的`to_representation`方法。

我觉得这个设计很不合理，因为当为`None`的时候，这个`field`根本不会收到这个value，这样即使它想要采取其它的动作，或者返回其它的值，都是不可能的。

但是没有办法，我只能采取另一种方法，重写`serializer`的`to_representation`方法：

```python
    def to_representation(self, instance):
        ret = super().to_representation(instance)
        if ret.get("avatar_url"):
            return ret
        else:
            ret["avatar_url"] = get_default_user_avatar_url()
            return ret
```

而且把之前`avatar_url`字段的定义删掉就行了，因为这里继承的是`ModelSerializer`，如果不改用自定义的字段的话它会默认生成一个和数据库的`field`对应的字段。

