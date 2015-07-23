---
layout: post
title:  "Effective Standardized Tracking"
date:   2015-07-22 12:00
categories: python
author: firecrow 
---

Event tracking is a growing and exceptionally valueable peice of product
decistions and business analytics.  More and more, companies are reevaluating
thier products or releasing multiple ideas at once. Long gone are the days when
a decision was made to release a single product that remained in production
till the end of time.  Why chose a single best idea when you can throw out
several of them and watch how users react. Or better yet, why plan a new
project without looking at how users have used your interface in the past. The
possiblities are endless, and your event tracking intfrastructure provides the
visiblity that makes this degree of experimentation possible. 

Because this type of analysis is rarely contained in one type of event,
consistency plays a huge role in the long term success of an implementation.
In my experience the question of what should be tracked is relatively
straightforward. How much to track and how to represent relationships in your
tracking data is where the interesting challenges arise. 

Over the years I've seen some amazing implementations and some failures that
became problematic over time. For that reason I've come up with a uniform
solution that increases consistency and completness across events.

Events are rarely conceived in the way they are later used, this is not any 
fault of the developer or the product team, but a reflection of the reality
that you may not know the questions you will need to ask of your data in the 
future. When data is stored in different ways, such as using different fields
to identify the same type of entities in different places.  For example, if a
page which tracks manufacturer creation sends only the id under the field name
mfr_id, whereas a purchase of an item with a manufacturer sends the 
manufacturer uuid, or just the name. Later from the business analytics side,
when joining events together, the difference in how entities are stored can 
become a huge hurdle in using a variety of events to tell a story. This is a
major reason why we chose to standardize the fields for a given entity being
tracked.

To tackle these concerns at Handshake, we took the approach of standardizing
fields for a given model. Sending the model to a function which generates a
dictionary of attributes, instead of hard coding the attributes into each
instance of tracking data. In addition to code brevity and consistency, this also
encouraged robust and flexible tracking methodologies. Including reasonable
length URL's to track verbose amounts of data. We use a system composed of
three main parts.

1.) The TRACKING_MAP, this is a map of the attributes we want to track for each
model. if that model has relationships, we refer to as children, they are
automatically tracked as well.

{% highlight python %}
TRACKING_MAP = {
    'CompanyAccount': {
        'attributes': ['name', 'is_demo'],
        'key': 'name',
    },
    'User': {
        'attributes': [
            'id',
            'first_name',
            'last_name',
            'email',
            'is_staff',
            'is_active',
            'is_superuser',
            'username',
        ],
        'model':User,
        'key': 'id',
    },
    'BuyerProfile': {
        'attributes': ['uuid','ctime'],
        'children': ['user'],
        'key': 'uuid',
    },
}
{% endhighlight %}

2.) A function we call trackingValues, that receives a series of model objects,
and generates data based on the TRACKING_MAP

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
              for childname in tmap['children']:
                  child = getattr(obj, childname)
                  data.update(trackingValues(child))
    return data

data = {
    'action':'signup_click'
    'name':my fancey things'
}

data.update(trackingValues(buyerA, demoCompany)) 

#output
data = {
    'action':'signup_click',
    'name':my fancey things',
    'BuyerProfile_uuid':'6fa459ea-ee8a-3ca4-894e-db77e160355e', 
    'BuyerProfile_ctime':'2015-07-20 11:21:08.278070' 
    'User_id':123,
    'User_first_name':'TestUser',
    'User_last_name':'LastName',
    'User_email':'test.user@example.com',
    'User_is_staff':False,
    'User_is_active':True,
    'User_is_superuser':False,
    'User_username':'testUserOne',
    'CompanyAccount_name': 'The Demo Account',
    'CompanyAccount_is_demo': True
}

{% endhighlight %}

Notice that the User information was included from its definition as a child
of the BuyerProfile Object.

3.) A function that accepts an abbreviated URL version of the model and the
attribute to look it up under, and then runs through the trackingValues
function to generate the data for that instance of the model.

{% highlight python %}
def getModel(string, value):
    try:
        clsname, attname = string.split('__')
        tmap = TRACKING_MAP.get(clsname)
        model = tmap.get('model'):
        return model.objects.get( **{ attname: value } )



#https://www.example.com?BuyerProfile__uuid=6fa459ea-ee8a-3ca4-894e-db77e160355e&CompanyAccount__name=The%20Demo%20Company

for key, value in request.GET:
    data = {}
    obj = getModel(key, value)
    if obj:
        data.update(trackingValues(obj))

#generates the same output as above
{% endhighlight %}

A few conventions have been defined, firstly for every place in tracking events
where these models appear, all of the information will be included.  This is
important because in the short term it may only require a few attributes to
meet the key result, but it's very possible that tracking data will end up
being used to answer questions other than the original intention.

For this reason having all the data is extremely valuable, also having a
complete set ensures that however an object is referenced, it can be referenced
that way across multiple events. Another benefit of the getModel function is
that it can take any uniquely identifiable field to generate the full set of
data for tracking.  This is useful for situations where you only have access to
the id or just the uuid or just the name or email. Depending on where you are
in your code, getting a reference to all the data points for the model may be
inconvenient. Your tracking url can simple use Class__key=value, and the server
will dig up the rest when it is received.
