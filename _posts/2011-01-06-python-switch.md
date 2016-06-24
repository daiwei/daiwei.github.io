---
layout: post
title: Python中switch的实现
---

**TL;DR** Python学习

众所周知Python中是没有`switch`的，一般而言是用`if-else`来代替的，如C语言下的

```
switch (key) {
    case 'a':
        /* do_a */
        break;
    case 'b':
        /* do_b */
        break;
    case 'c':
        /* do_c */
        break;
}
```

在Python中一般表示成

```
if key == 'a':
    # do_a
elif key == 'b':
    # do_b
elif key == 'c':
    # do_c
```

`if-else`足够简单，也足够实用，它也能模拟出多个`case`完成同样的事，及`default`情况。

不过也有人喜欢`dict`来实现

```
{
    'a': do_a,
    'b': do_b,
    'c': do_c
}[key](x)
```

不过上面的实现没办法模拟出多个`case`完成同样的事的情况，勉强能够实现`default`，不过就比较丑陋了

```
try:
    {
        'a': do_a,
        'b': do_b,
        'c': do_c
    }[key](x)
except KeyError:
    do_default
```

自己也尝试利用类实现了一个，结合了使用类模拟了`dict`部分属性，来扩展`dict`以可以模拟出多个`case`完成同样的事，及`default`情况。

```
class Switch:
    def __init__(self, data = {}):
        self.data = {}
        for key in data.keys():
            if type(key) == tupe:
                for k in key:
                    self.data[k] = data[key]
            else:
                self.data[key] = data[key]

    def __setitem__(self, key, item):
        self.data[key] = item

    def __getitem__(self, key):
        if key in self.data.keys():
            return self.data[key]
        else:
            return self.data['default']
# end Switch

# 使用，这里展示了一个嵌套switch的例子
Switch ({
    'a':        Switch ({   1:          do_a_1,
                            (2, 3):     do_a_2,
                            'default':  do_a    }),
    'b':        Switch ({   1:          do_b_1,
                            (2, 3):     do_b_2,
                            'default':  do_b    }),
    'default':  Switch ({   'default':  do_default })
}) [key1][key2] ()
```

不过自己看了后觉得依旧丑陋啊。  ==!

上网google了一下，发现了一个大牛的`switch`

```
## http://code.activestate.com/recipes/410692/ (r8)
# This class provides the functionality we want. You only need to look at
# this if you want to know how this works. It only needs to be defined
# once, no need to muck around with its internals.
class switch(object):
    def __init__(self, value):
        self.value = value
        self.fall = False

    def __iter__(self):
        """Return the match method once, then stop"""
        yield self.match
        raise StopIteration

    def match(self, *args):
        """Indicate whether or not to enter a case suite"""
        if self.fall or not args:
            return True
        elif self.value in args: # changed for v1.5, see below
            self.fall = True
            return True
        else:
            return False


# The following example is pretty much the exact use-case of a dictionary,
# but is included for its simplicity. Note that you can include statements
# in each suite.
v = 'ten'
for case in switch(v):
    if case('one'):
        print 1
        break
    if case('two'):
        print 2
        break
    if case('ten'):
        print 10
        break
    if case('eleven'):
        print 11
        break
    if case(): # default, could also just omit condition or 'if True'
        print "something else!"
        # No need to break here, it'll stop anyway

# break is used here to look as much like the real thing as possible, but
# elif is generally just as good and more concise.

# Empty suites are considered syntax errors, so intentional fall-throughs
# should contain 'pass'
c = 'z'
for case in switch(c):
    if case('a'): pass # only necessary if the rest of the suite is empty
    if case('b'): pass
    # ...
    if case('y'): pass
    if case('z'):
        print "c is lowercase!"
        break
    if case('A'): pass
    # ...
    if case('Z'):
        print "c is uppercase!"
        break
    if case(): # default
        print "I dunno what c was!"

# As suggested by Pierre Quentel, you can even expand upon the
# functionality of the classic 'case' statement by matching multiple
# cases in a single shot. This greatly benefits operations such as the
# uppercase/lowercase example above:
import string
c = 'A'
for case in switch(c):
    if case(*string.lowercase): # note the * for unpacking as arguments
        print "c is lowercase!"
        break
    if case(*string.uppercase):
        print "c is uppercase!"
        break
    if case('!', '?', '.'): # normal argument passing style also applies
        print "c is a sentence terminator!"
        break
    if case(): # default
        print "I dunno what c was!"

# Since Pierre's suggestion is backward-compatible with the original recipe,
# I have made the necessary modification to allow for the above usage.
## end of http://code.activestate.com/recipes/410692/
```

不过回头看看，觉得为了一个`switch`作这么多蛋疼的事，倒不如老老实实的用`if-else`，正应了一句话“步子迈大了，会扯着蛋”。

写这么多屁话，只是想找个地方保存一些代码。
