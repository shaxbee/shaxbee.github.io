---
title: Python Binary Data Part 2 - Struct Unpacking
layout: post
---

While writing code for handling fixed size structs in flatbuffers I had to make a choice between defereed 
access (reading field on demand) or reading all fields at once.

It seems that difference between unpacking single and multiple values is neglible:

{% highlight bash %}
> # single value
> python -m timeit -s 'import struct; s = struct.Struct("B"); b = bytearray([0x42])' 's.unpack(b)'
1000000 loops, best of 3: 0.373 usec per loop

> # three values
> python -m timeit -s 'import struct; s = struct.Struct("3B"); b = bytearray([0x42] * 3)' 's.unpack(b)'
1000000 loops, best of 3: 0.406 usec per loop

> # six values
> python -m timeit -s 'import struct; s = struct.Struct("6B"); b = bytearray([0x42] * 6)' 's.unpack(b)'
1000000 loops, best of 3: 0.423 usec per loop
{% endhighlight %}

Also unpacking the whole structure at once into a tuple allows interesting optimization
used in namedtuple - deriving class from tuple and delegating property access to __getitem__.
That makes instances immutable, reduces memory footprint and reduces overhead of accessing property.

{% highlight python %}
import struct
import operator

class Vec3(tuple):
    def __new__(cls, x, y, z):
        return tuple.__new__(cls, (x, y, z))

    x = property(operator.itemgetter(0))
    y = property(operator.itemgetter(1))
    z = property(operator.itemgetter(2))

    _fmt = struct.Struct('<fff')

    def pack_into(self, buf, offset):
        _fmt.pack_into(buf, offset, *self)

    @classmethod
    def unpack_from(cls, buf, offset):
        return tuple.__new__(cls, _fmt.unpack_from(buf, offset))
{% endhighlight %}
