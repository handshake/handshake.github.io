---
layout: post
title:  "But Python Closes Files Automatically"
date:   2014-08-05 10:30
categories: python
author: matthewstory
---

Sometimes in code review I see code that fails to guarentee that all opened
files are properly closed. You've likely seen code like this too, it looks
like this:

{% highlight python %}
def save_to_file(name, data):
    '''Save `data` to a file named `name`'''
    file_ = open(name, 'w+')
    file_.write(data)
{% endhighlight %}

There are a number of ways to refactor this to ensure the opened file(s) are
properly closed. The most elegant (and Pythonic) way uses the `with` keyword,
and takes advantage of the fact that `file` objects are [their own context
managers](http://preshing.com/20110920/the-python-with-statement-by-example/):

{% highlight python %}
def save_to_file(name, data):
    '''Save `data` to a file named `name`, and explicitly close by way of a
       context manager.'''
    with open(name, 'w+') as file_:
        file_.write(data)
{% endhighlight %}

As simple as this is, I will sometimes get pushback that it is unnecessary,
because _python closes files automatically_.

### Always Explicitly Close Yr Files

Python does close files automatically when the file object reference
count is decremented to `0`. But there are a number of reasons why that might
not happen when you exit the current scope.

The most obvious case of when a file may not be closed when you think, is when
the scope is exited due to an unhandled exception. Let's look at a slightly
modified, and more than slightly contrived revision of our above functionality:

{% highlight python %}
import os
import errno
import gc
import sys

fd = None

def save_to_file(name, data):
    '''Write non-empty `data` to a file named `name'''
    global fd
    file_ = open(name, 'w+')
    fd = file_.fileno()
    if not data:
        raise ValueError('data cannot be empty')
    file_.write(data)

def is_still_open():
    '''Return True if file is still open, False if not'''
    try:
        os.fstat(fd)
    except OSError, e:
        if e.errno == errno.EBADF:
            return False
        raise
    return True

### With No Exception
save_to_file('./foo.txt', 'hi')
print 'No Exception, File is {}'.format('Open' if is_still_open() else 'Closed')

### With a Handled Exception
try:
    save_to_file('./foo.txt', '')
except ValueError:
    pass
print 'With Exception, File is {}'.format('Open' if is_still_open() else 'Closed')

### Try running the garbage collector, just in case
unreachable = gc.collect()
print 'Even after gc.collect, File is {} ({} unreachable'\
      ' object{})'.format('Open' if is_still_open() else 'Closed',
                          unreachable, '' if unreachable == 1 else 's')
{% endhighlight %}

When we run the code above, we generate the following output:

    No Exception, File is Closed
    With Exception, File is Open
    Even after gc.collect, File is Open (0 unreachable objects)

Which indicates that for the normal case when no exception is raised, Python
is automatically closing the `file` object when it goes out of scope, but in
the case where `save_to_file` raises an exception, the file is in fact not
closed ... even after successfully running a full collection with `gc`.

### WAT?

Surprising as it may be here, Python is playing by the same rules in all cases
above, it's just that the `file` object's reference count has yet to hit `0`
in the case where `save_to_file` raises an exception. The final reference(s)
to it just happen to be stored thread-locally in
[`sys.exc_traceback`](https://docs.python.org/2/library/sys.html#sys.exc_traceback).
By adding a `sys.exc_clear` to our above report:

{% highlight python %}
### Clear sys.exc_* variables
sys.exc_clear()
print 'After sys.exc_clear, File is {}'.format('Open' if is_still_open() else 'Closed')
{% endhighlight %}

we finally close the file:

    No Exception, File is Closed
    With Exception, File is Open
    Even after gc.collect, File is Open (0 unreachable objects)
    After sys.exc_clear, File is Closed

All of this nonsense is easily avoided by modifying `save_to_file` to use `with`:

{% highlight python %}
def save_to_file(name, data):
    '''Write non-empty `data` to a file named `name'''
    global fd
    with open(name, 'w+') as file_:
        fd = file_.fileno()
        if not data:
            raise ValueError('data cannot be empty')
        file_.write(data)
{% endhighlight %}

which gives us the desired result in all cases:

    No Exception, File is Closed
    With Exception, File is Closed

and ensures that buffers are flushed, locks are released and files are closed
exactly when we want them to be.
