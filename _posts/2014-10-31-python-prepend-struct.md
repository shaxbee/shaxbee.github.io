---
title: Performance of prepending to bytearray
layout: post
---

While working on Python flatbuffers binding i've noticed that prepending to bytearray is ~50 times slower than preallocating space and overwriting data directly.

{% highlight bash %}
> # prepend
> python -m timeit -s "b = bytearray()" "b[:0] = 'foo'"
100000 loops, best of 3: 17 usec per loop

> # overwrite
> python -m timeit -s "b = bytearray(1024)" "b[0:3] = 'foo'"
1000000 loops, best of 3: 0.356 usec per loop
{% endhighlight %}

The problem stems from fact that when data is being prepended bytearray is forced to memcpy the whole data block right.

In case of overwriting you need to additionaly maintain position in buffer and resize buffer when space on-left is exhausted.
The cost would be however amortized for multiple prepend operations.

{% highlight python %}
def make_space(buf, size, padding=bytearray(1024)):
    remaining = len(buf) - size
    while remaining > 0:
        buf[:0] = padding
        remaining -= len(padding)
{% endhighlight %}

Resizing operation would be much cheaper if there would be equivalent of C-API PyByteArray_Resize on Python side.

{% highlight python %}
def make_space(buf, size):
   buf.resize(len(buf) + size)
   buf[size:] = buf[0:len(buf) - size]
{% endhighlight %}
