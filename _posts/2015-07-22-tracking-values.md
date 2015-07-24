---
layout: post
title: "Effective Standardized Tracking"
date: 2015-07-22 12:00
categories: python
author: Firecrow Silvernight
---

### The Value of Event Tracking

Event tracking is a growing and exceptionally valuable part of product decisions and business analytics. More and more, companies regularly re-evaluate existing features and release multiple ideas at once. Gone are the days when decisions were made to release single, unchanging features that sit in production till the end of time. Why choose a single, “best,” idea when you can try out several at once, watch how users react, and react in kind? Why plan a new project without looking at how users have historically interacted with your product? This degree of experimentation is made possible by a robust event-tracking infrastructure.

The stories that drive customer insights are rarely contained to any one type of event. The question of what should be tracked is generally straightforward [???]. How much to track and how to represent properties and relationships in your tracking data is where the interesting challenges arise. Events are rarely conceived in the way they are later used. It’s unlikely that you will know the right questions to ask till you see historical results and product priorities may change in a way that requires new insights.

Over the years I’ve seen some amazing event tracking implementations along with a few failures. One of the most common issues that arises is a lack of consistency, especially when the same data is stored in different ways. For example, one of our pages may track a new manufacturer and sends `id` as its unique identifier, whereas an page tracking the purchase of a manufacturer’s product may track the manufacturer using `uuid` or `name`. This can make reconciling the `manufacturer` across events quite cumbersome. In addition to being more complicated to query, it can have enormous performance implications if you’re missing data and—if you can—need to hit a separate database to flesh out the missing attributes in your data.

To tackle these concerns at Handshake, we took the approach of building a convention around generating which pieces of data are relevant to tracking an entity. This includes the attributes of any entity implicitly related to the model, such as a child or parent. This is done by creating a central definition for each entity and passing the model object to a function that generates key/value pairs to be sent with the tracking event.

### The Base Framework

In our system we handle a lot of wholesale orders. The objects we work with commonly are manufacturers, their products, orders and their line items.

We use a dictionary to map keys to model attributes we want to track. If the model has any relationships, they are tracked as well, using the relationship’s dictionary.

{% highlight python %}
TRACKING_MAP = {
    "Product": {
        "attributes": ["id", "name", "price", "sku"],
        "related": ["manufacturer_id"],
        "key": "id",
    },
    "Manufacturer": {
        "attributes": [
            "id",
            "uuid",
            "name",
            "externalID",
        ],
        "key": "id",
    },
    "OrderLine": {
        "attributes": ["id", "qty"],
        "related": ["product", "order"]
        "key": "id",
    },
    "Order": {
        "attributes": ["id", "uuid", "total"],
        "related": ["retail_company"],
        "key": "uuid",
    },
}
{% endhighlight %}

A related function takes a series of model objects and generates data base on the tracking map.

{% highlight python %}
def trackingValues(**args):
    data = {}
    for obj in args:
        clsname = obj.__class__.__name__
        prefix = clsname + "_"
        try:
            tmap = TRACKING_MAP.get(clsname)
            for name in tmap["attributes"]:
                data[prefix + name] = value
        [???] except:

        if tmap.get("children"):
              for relname in tmap["related"]:
                  rel = getattr(obj, relname)
                  data.update(trackingValues(rel))
    return data

data = {
    "action": "orderLineUpdated",
}
{% endhighlight %}

Lets say we have a line item, `orderLine`, and an associated quantity. This line item belongs to an order and references a product, which in turn belongs to a manufacturer. We can track all of the above by merely passing in the `orderLine`.

{% highlight python %}
data.update(trackingValues(orderLine))
{% endhighlight %}

Once updated, `data` looks like this:

{% highlight python %}
{
    "action": "orderLineUpdated",
    "OrderLine_id": 1001,
    "OrderLine_qty": 3,
    "Product_id": 123,
    "Product_name": "Gold Plated Duck Tape",
    "Product_price": 100000.0,
    "Product_sku": "gquack",
    "Order_id?: 357,
    "Order_uuid": "2500b32e-7c1b-11e2-830a-70cd60f2c980",
    "Order_total": 714328.0
    "Manufacturer_id": 789,
    "Manufacturer_uuid": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
    "Manufacturer_name": "Mario Bros Plumbing Supply",
    "Manufacturer_externalID": "MBroPS",
}
{% endhighlight %}

Each model related to the `orderLine` is included. Note that the order total is included, which is data beyond the scope of the specific line referenced. It’s tempting to say that the `order` details should not be in the event, but that breaks the simplicity of having this data automated. When it comes to event tracking, extra data is far better than not enough.

Wherever a tracking event includes a model, all of its associated information will be included, not just the attributes you think you need beforehand to meet the key result. It’s very possible that the additional data will be used to answer questions beyond the original intent.

### Where Do We Go From Here?

Our event tracking code, which had originally been spread all over, has been simplified and unified. By using the model itself to generate tracking data, and by including its relationships, we get a robust and consistent implementation with only a few lines of code.

From here we can extend the framework to fit our needs, including one-to-many relationship support and metadata for the state of each entity [???]. It’s trivial to add additional arguments to the function or add lambda functions to the `TRACKING_MAP` itself. [???] We can add values to the data object directly, in the above case the “action” value, if we could also add custom values to the entities themselves we could further customized our platform for targeted user events.

Generating values in one place to be homogenous in all is just the first step. Stay tuned for the next iteration, where we build on our concise and powerful base to [???].
