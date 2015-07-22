---
layout: post
title:  "But Python Closes Files Automatically"
date:   2015-07-22 12:00
categories: python
author: firecrow 
---

Consistency is an important part of tracking events across your site. Tracking
how users interact with your product is an invaluable part of business
analytics. It can also be way of validating of debunking ideas about how your
product is being used. Usually in my experience the question of what should be
tracked is relatively straight forward. However how much to track and how to
standardized that tracking is more complicated. 

Events are rarely conceived in the way they are later used, this is not any
fault of the developer or the product team, but a reflection of the reality
that you may not know the questions you will need to ask of your data in the
future. If manufacturers are refered by mfr_id in some places mfr_uuid and
mfr_slug in others it can quickly becomes attribute soup and cuase barriers to
generating shared views.

To tackle these concerns at handshake, we took the aproach of standardizing
fields for a given model. Only refering to the model itself, and then using a
standard set of attributes every time that model was set up for tracking. In
addition to enhancing code breivity it also encouraged robust and flexible
tracking methodologies. Including resonable length URL's to track verbose
amounts of data. We use a system composed of three main parts.

1.) The TRACKING_MAP, this is a map of the attributes we want to track for each
model.  And if that model has relationships, we refer to as children, they are
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

2.) A function we call trackingValues, that recieves a series of model objects,
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

Notice that the User information was included from it's definition as a child
of the BuyerProfile Object.

3.) a function that accepts an abreviated URL version of the model and the
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
    obj = getModal(key, value)
    if obj:
        data.update(trackingValues(obj))

#generates the same output as above
{% endhighlight %}

A few conventions have been defined, firstly for every place in tracking events
where these models appear, all of the information will be included.  This is
important because in the short term it may only require a few attributes to
meet the key result, but it's very possible that tracking data will end up
being used to answer questions other than the original intention. 

For this reason haveing all the data is extremely valuable, also having a
complete set ensures that however an object is referenced, it can be referenced
that way across multiple events. Another benefit of the getModel function is
that it can take any uniquely identifiable field to generate the full set of
data for tracking.  This is useful for situations where you only have access to
the id or just the uuid or just the name or email. Depending on where you are
in your code, getting a reference to all the data points for the model may be
inconvienent. Your tracking url can simple use Class__key=value, and the server
will dig up the rest when it is recieved.

