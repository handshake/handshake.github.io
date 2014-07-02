---
layout: post
title:  "Rich Comparison Methods"
date:   2014-07-02 10:30
categories: python
author: matthewstory
---

One of the nicest features of the [Data Model](https://docs.python.org/2/reference/datamodel.html)
is the ability to override the behavior of
[rich comparison operators](https://docs.python.org/2/reference/datamodel.html#object.__lt__):

{% highlight python %}
import functools

@functools.total_ordering
class Generic(object):
    def __init__(self, id_):
        self.id = id_

    def __repr__(self):
        return '<Generic id={}>'.format(self.id)

    def __eq__(self, other):
        if isinstance(other, self.__class__):
            return self.id == other.id
        return NotImplemented

    def __ne__(self, other):
        if isinstance(other, self.__class__):
            return self.id != other.id
        return NotImplemented

    def __lt__(self, other):
        if isinstance(other, self.__class__):
            return self.id < other.id
        return NotImplemented
{% endhighlight %}

Now we can conveniently compare two `Generic` objects to one another without
having to access the `id` attribute over and over:

{% highlight python %}
>>> Generic(1) == Generic(1)
True
>>> Generic(1) < Generic(2)
True
>>> # etc.
{% endhighlight %}

We can even compare `Generic` objects to objects of other types, and because
we're using `NotImplemented`, we'll get a reasonable result:

{% highlight python %}
>>> Generic(1) == 1
False
>>> Generic(1) < 2
False
>>> # etc.
{% endhighlight %}

### Which Method is Called?

In the simple cases above, it's trivial to determine which comparison methods
prefers. But to get a more complete picture of what's going on under the hood,
let's decorate our overridden class to print to `stdout` anytime a rich
comparison method is called:

{% highlight python %}
def chirp_richcomps(cls):
    '''Decorator which prints to stdout everytime a rich-comparison method is
       called'''
    def chirp_factory(method, attr):
        def inner(self, other):
            print '{}: Called {}'.format(self, attr)
            return method(self, other)
        return inner

    for attr in ('eq', 'ne', 'lt', 'le', 'gt', 'ge'):
        attr = '__{}__'.format(attr)
        method = getattr(cls, attr)
        if method is not None:
            setattr(cls, attr, chirp_factory(method, attr))

    return cls

Generic = chirp_richcomps(Generic)
{% endhighlight %}

#### Case #1: Only One Side Has a Comparison Override

{% highlight python %}
>>> Generic(1) < 1
<Generic id=1>: Called __lt__
False
>>> 1 > Generic(1)
<Generic id=1>: Called __lt__
False
{% endhighlight %}

In the case where only one side of the comparison supports an override, the
object with a comparison override is prefferred regardless of side. When
the object with the override is on the right side, and `>`, `>=`, `<` or `<=`
are used, python will use the reflection method of the invoked operator. In
the above example this means `1 < Generic(1)` actually invokes
`Generic(1).__lt__(1)`. Similarly `<` would invoke `__gt__`, `<=` would invoke
`__ge__` and `>=` would invoke `__le__`.

#### Case #2: Both Sides Have a Comparison Override

{% highlight python %}
>>> Generic(1) < Generic(2)
<Generic id=1>: Called __lt__
>>> # set overrides comparison operators
>>> Generic(1) == set()
<Generic id=1>: Called __eq__
>>> # let's take a closer look at that
>>> @chirp_richcomps
... class ChirpSet(set)
...     pass
...
>>> Generic(1) == ChirpSet()
<Generic id=1>: Called __eq__
ChirpSet([]): Called __eq__
>>> ChirpSet() == Generic(1)
ChirpSet([]): Called __eq__
{% endhighlight %}

In the naive case where both sides have a comparison override, Python preferrs
the left side. The comparison against `ChirpSet` above demonstrates what
happens if the left-side comparison returns the `NotImplemented` constant,
which will cause Python to fall-back to use the comparison operator from the
right side if present. The final example from above shows that unlike
`Generic`, `set` does not return `NotImplemented` when compared against a
non-`set`, preferring to simply return False instead (which we see because
`Generic(1).__eq__(ChirpSet())` is not attempted.

#### Case #3: Both Sides Have a Comparison Override, But One Side Inherits The Other

{% highlight python %}
>>> class Specific(Generic):
...     pass
>>> Generic(1) == Specific(1)
<Specific id=1>: Called __eq__
<Generic id=1>: Called __eq__
True
>>> class ReallySpecific(Specific):
...     pass
>>> ReallySpecific(1) == Generic(1)
<ReallySpecific id=1>: Called __eq__
<Generic id=1>: Called __eq__
True
{% endhighlight %}

In this final case we've setup the classes `Specific` and `ReallySpecific`
which directly and indirectly inherit `Generic` to demonstrate what happens
when one side of the comparison includes the `type` of the other side in its
`mro`. In this case, the more specific `type` is preferred, regardless of
which side it happens to be on. Note that this doesn't happen in the case of
shared inheritence:

{% highlight python %}
>>> class OtherSpecific(Generic):
...     pass
>>> Specific(1) == OtherSpecific(1)
<Specific id=1>: Called __eq__
<OtherSpecific id=1>: Called __eq__
<OtherSpecific id=1>: Called __eq__
<Specific id=1>: Called __eq__
False
>>> OtherSpecific(1) == ReallySpecific(1)
<OtherSpecific id=1>: Called __eq__
<ReallySpecific id=1>: Called __eq__
<ReallySpecific id=1>: Called __eq__
<OtherSpecific id=1>: Called __eq__
False
{% endhighlight %}

As to why each method is called twice in this case ... that's another topic
for another day.
