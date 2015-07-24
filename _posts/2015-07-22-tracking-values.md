---
layout: post
title:  "Effective Standardized Tracking"
date:   2015-07-22 12:00
categories: python
author: firecrow 
---

### The Value of Event Tracking

Event tracking is a growing and exceptionally valuable piece of product
decisions and business analytics.  More and more, companies are reevaluating
their products or releasing multiple ideas at once. Long gone are the days when
a decision was made to release a single product that remained in production
till the end of time.  Why chose a single best idea when you can throw out
several of them and watch how users react. Or better yet, why plan a new
project without looking at how users have historically used your interface. The
possibilities are endless. This degree of experimentation is made possible by a
robust Event tracking infrastructure. 

Telling the stories that drive these insights is rarely contained to one type
of event. In my experience the question of what should be tracked is relatively
straightforward. How much to track and how to represent relationships in your
tracking data is where the interesting challenges arise.  Events are rarely
conceived in the way they are later used. This is a reflection of the fact that
it is unlikely that you will know the right questions until you see historical
results, or product priorities may change in a way that requires new insights. 

Over the years I've seen some amazing implementations and some failures. For
that reason I've come up with a solution that increases consistency and
completeness across events. One of the most common consistency issues that
arrises is when data is stored in different ways, such as using different
fields to identify the same type of entity. For example, if a page which tracks
`manufacturer` creation sends only the `id`, whereas a purchase of a **product** with a
`manufacturer` sends the manufacturer `uuid`, or just the `name`, reconciling the
`manufacturer` in these cases can be cumbersome. In addition to being more
complicated to query, it can have enormous performance implications if you
need to query a separate database to flesh out the missing attributes in your
data.

To tackle these concerns at Handshake, we took the approach of building a
convention around generating which pieces of data are relevant to tracking an
entity. This includes the attributes of any entity implicitly related to the
model, such as a child or parent. This is done by creating a central
definition for each entity, and then passing the model to a function which
generates the key/values to be sent to the tracking event.

### The Base Framework

In our system we handle a lot of wholesale orders. The objects we work with
comonly are the `products`, `orders`, `lines` of each `order`, and
manufacturers of those `products`.

*The `TRACKING_MAP`, this is a map of the attributes we want to track for each
model. if that model has relationships, they are
automatically tracked as well.*

{% highlight python %}
TRACKING_MAP = {
    'Product': {
        'attributes': ['id','name', 'price', 'sku'],
        'related': ['manufacturer_id'],
        'key': 'id',
    },
    'Manufacturer': {
        'attributes': [
            'id',
            'uuid',
            'name',
            'externalID',
        ],
        'key': 'id',
    },
    'OrderLine': {
        'attributes':['id', 'qty'],
        'related': ['product', 'order']
        'key':'id',
    },
    'Order': {
        'attributes': ['id', 'uuid', 'total'],
        'related': ['retail_company'],
        'key': 'uuid',
    }
}
{% endhighlight %}

*A function we call `trackingValues`, that receives a series of model objects,
and generates data based on the `TRACKING_MAP`.*

{% highlight python %}
def trackingValues(**args):
    data = {}
    for obj in args:
        clsname = obj.__class__.__name__
        prefix = clsname + '_'
        try:
            tmap = TRACKING_MAP.get(clsname)
            for name in tmap['attributes']:
                data[prefix+name] = value

        if tmap.get('children'):
              for relname in tmap['related']:
                  rel = getattr(obj, relname)
                  data.update(trackingValues(rel))
    return data

data = {
    'action':'orderLineUpdated'
}

{% endhighlight %} 

*Lets say we have an `orderline` with a quantity, that contains an `product`, and
belongs to an `order`, and the `product` has a `manufacturer`. We can track all that
by passing in the `orderline`.*

{% highlight python %}

data.update(trackingValues(orderLine)) 

{% endhighlight %} 

*Will produce the output below.*

{% highlight python %}
data = {
    'action':'orderLineUpdated'
    'OrderLine_id':1001,
    'OrderLine_qty': 3,
    'Product_id': 123,
    'Product_name':'Gold Plated Duck Tape',
    'Product_price': 100,000.00
    'Product_sku': 'gquack',
    'Order_id':357,
    'Order_uuid':'2500b32e-7c1b-11e2-830a-70cd60f2c980'
    'Order_total':714,328.00
    'Manufacturer_id':789,
    'Manufacturer_uuid':'f81d4fae-7dec-11d0-a765-00a0c91e6bf6',
    'Manufacturer_name': 'Mario Bros Plumbing Supply',
    'Manufacturer_externalID': "MBroPS',
}
{% endhighlight %}

Notice that the models related to the `orderline` are included here. One thing to
keep in mind is that in this function, the order total is representative of an
order with several order lines, beyond just the duck tape. It's tempting in
situations like this to say that the `order` details should not be in the event,
but that breaks the simplicity of having this data automated. And when it comes
to event tracking, extra data is far better than not enough.

A few conventions have been defined, firstly for every place in tracking events
where these models appear, all of the information will be included.  This is
important because in the short term it may only require a few attributes to
meet the key result, but it's very possible that tracking data will end up
being used to answer questions other than the original intention.

### Where Do We Go From Here

This is very exciting to me. The code to track events, which will be sprinkled
all over the place, has been simplified. Both by using only a model in the
calls to generate tracking data, and by including the relationships, we get a
robust consistent implementation with only a few lines of code. 

Form here we can extend the framework to include one to many relationships, or
metadata for the state of each entity. It will be trivial to extend this methodology to
include additional arguments to the function, or add lambda functions to the
`TRACKING_MAP`. We have can add values to the data object directly, in the above
case the 'action' value, if we could also add custom values to the entities
themselves we could further customized our platform for targeted user events. 

This implementation is just the first step, based on the ideas of generating
values in one place so they are homogenous for all. Stay tuned for the next
iteration of this system, now that we have a concise and powerful base, event
tracking for highly intricate insights can be the next big thing.
