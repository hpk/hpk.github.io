---
layout:     post
title:      Using Keen IO for Whitelabel Analytics
date:       2015-01-27
comments:   true
---

Buy vs. build? It's a tough decision, and it's been around for as long as businesses have been building things. I faced the same quandry at [Retention Hero](http://www.retentionhero.com) when customers began asking for better visibility into their sales data. In addition to Retention Hero's core functionality (predicting customer behavior and helping marketers use those predictions to boost repeat sales), we needed a way to provide advanced analytics so that our users could see how different segments of customers were behaving.

Rather than build and maintain an analytics infrastructure, I chose to use [Keen IO](https://keen.io) for gathering, storing, and querying our event data. Our own Postgres database is still the "system of record" for sales data, but Keen IO essentially acts as a large-scale cache that backs the time-series reports we include in Retention Hero. I'm going to show you how we built our analytics system on top of Keen IO, and how we considered important factors such as security and performance.

### Getting Started ###

We deploy Retention Hero to a few different environments -- development, staging, and production -- and wanted to make sure the analytics data for each one was separate from the others. So I set up a separate Keen IO project for each environment and added the respective project IDs and keys to our Django settings files. Keen IO has an officially supported Python library, so setting up a client that can talk to their service is as simple as doing:

{% highlight python %}
from django.conf import settings
from keen.client import KeenClient

def get_keen_client():
    return KeenClient(
        project_id=settings.KEEN_PROJECT_ID,
        write_key=settings.KEEN_WRITE_KEY,
        read_key=settings.KEEN_READ_KEY
    )
{% endhighlight %}

When a user first connects their eCommerce platform to Retention Hero, we import their sales history into our database so that we can train our models on the data. We also watch for new orders and import those on an ongoing basis. Once an order is saved in our database, it's pretty simple to record it to Keen as an event:

{% highlight python %}
def record_orders(orders):
    client = get_keen_client()
    for batch in batch_gen(orders, 1000):
        items = []
        for order in batch:
            items.extend(_order_items_to_keen(order))
        client.add_events({
            'orders': [_order_to_keen(order) for order in batch],
            'ordered_items': items,
        })
        batch_ids = [o.id for o in batch]
        Order.objects.filter(pk__in=batch_ids).update(synced_to_analytics=True)
{% endhighlight %}

`record_orders` takes a list of a store's orders and does the following:

- Breaks the list of orders into batches of 1,000
- Generates two types of events for each order: the order itself, and an event for each ordered product (more on this below)
- Sends the event batch to Keen (`client.add_events(...)`)
- Marks each order as "synced to analytics", so that we can easily do a query later to get all un-synced orders, and so that we don't accidentally record an order more than once.

`batch_gen` is a simple utility function that does exactly what it sounds like. To dig into the specifics of `_order_to_keen` and `_order_items_to_keen`, we need to talk about...

### Performance ###

Retention Hero uses, for the most part, a pretty traditional relational database schema. We've got a `customers` table, an `orders` table, an `order_items` table, etc., and foreign key relationships between them where appropriate. That's a great way to model data, but it becomes a performance issue when you want to do any sort of query that involves joins. For example, asking our database to "show me which products are ordered most often by active customers" would involve a join between the customers, orders, and order items tables.

Most analytics tasks are essentially write-once, read-many-times, so we want to optimize for reads. So we want to eliminating joins wherever possible in our data store, even if it means duplicated and un-normalized data. So when we send sales order events to Keen, we have to attach a bunch of related data to each event, and a lot of that data is "redundant" in the traditional database sense. `_order_items_to_keen` in the example above takes a customer's order and turns it into a list of dictionaries for each line item that looks something like this:

{% highlight javascript %}
{
    "customer": {
        "first_name": "John",
        "last_name": "Doe",
        "linked_id": "123456789",
        "uuid": "9EmCFUKCdwrRPEWykcQueT",
        "activity_state": "no_purchases",
        "email": "johndoe@example.com"
    },
    "keen": {
        "timestamp": "2015-01-27T02:57:46.000Z",
        "created_at": "2015-01-25T20:31:10.914Z",
        "id": "54c7f58ed2eaaa648ad2dae9"
    },
    "order": {
        "sequence_index": 0,
        "linked_id": "987654321",
        "buyer_accepts_marketing": false,
        "landing_domain": "www.mycoolstore.com",
        "source": "web",
        "address": {
            "province_code": "MN",
            "zip": "55410",
            "city": "Minneapolis",
            "address1": "321 Blast Off St",
            "address2": "",
            "country_code": "US",
            "region": "Minnesota",
            "company": null
        },
        "uuid": "VSsr6B77cKjnGS9gCBpFrD",
        "currency": "USD",
        "is_repeat": false,
        "num_items": 1,
        "total": 39.99,
        "subtotal": 39.99,
        "discount_used": false,
        "referring_domain": "www.google.com"
    },
    "product": {
        "category": null,
        "variant_linked_id": "167-138",
        "linked_id": "95",
        "uuid": "DR6s9zNpRUyBCwiMmPHSEZ",
        "price": 12,
        "variant_uuid": "Hb5kk6t8RgcVauCzFqsT9T",
        "quantity": 3
    },
    "store_uuid": "a9j5yH9zk8HWJFAuXBoL74"
}
{% endhighlight %}

`_order_to_keen` does something similar, but generates a single event for the order itself.

If a customer's order includes two different products, Retention Hero sends two events like the one above, one for each product. The `order` dictionary is exactly the same for each "ordered product" event. It may seem wasteful to send lots of redundant data, but it makes our lives _immensely_ easier when running queries against these events later on.

For example, if I want to see a time-series chart for sales of a particular product, grouped by whether or not the sale was a repeat (customer has ordered before) or non-repeat (this is the customer's first order), I can do that by simply grouping by `order.is_repeat`. Keen IO won't have to perform any joins, and I can get my data back quickly.

### Security ###

When a user wants to see a chart for a particular analytics query, our front-end client running in their browser makes a request to Keen and draws the plot. So we need to allow our code, running in an untrusted environment, to make authenticated calls to the Keen IO API.

If we simply included our "master" Keen credentials when serving the page to our user, that would be a problem. They would have access not only to their own data, but every other user's data as well. We need a way to "silo" every customer so that they only have access to their account's data.

Keen makes it possible to silo different users with what they call "scoped keys". Essentially, they're cryptographically-secure keys that are derived from the master key (that we keep secret) and have certain query parameters "baked in". When a new user signs up for Retention Hero, we generate a unique scoped key for them with the following:

{% highlight python %}
from keen import scoped_keys
def generate_scoped_read_key(store):
    api_key = settings.KEEN_MASTER_KEY
    return scoped_keys.encrypt(api_key, {
        "allowed_operations": ["read"],
        "filters": [{
            "property_name": "store_uuid",
            "operator": "eq",
            "property_value": store.uuid,
        }]
    })
{% endhighlight %}

This creates a new key that has access to Keen's API, but can only perform the specified operation (in this case, read) and will only include events that match a particular filter. In our case, every store that's connected to Retention Hero has its own read key that automatically filters out every event except ones belonging to their account. So now, rather than exposing our master key to every user, we can expose this limited-scope read key instead. We're guaranteed to never have two accounts' data mixed together.

### Conclusion ###

We're using Keen at Retention Hero to power our in-app analytics, and we don't have to worry about provisioning servers, setting up Cassandra clusters, scaling up as load increases, managing schema changes, and so on. For us, that's a huge win.

But Keen isn't a silver bullet, either. One problem is that our usage is very spiky, and Keen sometimes kicks our account into a higher-cost payment tier as a result. We import all of a store's historical data when they sign up for Retention Hero, so there are initially a lot of events that we send in one big batch, but after that the event volume is much lower. We're happy to pay Keen for the data volume that we use, but I do wish that Keen had a lower-cost bulk import functionality for data that isn't "real time".

Another issue is that, although Keen IO's JavaScript client includes some simple charting functionality that's great for getting started, we've still had to build out the front-end "report builder" UI ourselves. To be honest, that's pretty specific to every unique use case, so I doubt Keen would be able to provide something general enough to work in most cases. But if you're looking for a drop-in, no-coding-required solution, Keen isn't your answer.

In summary, if you're tasked with building analytics and reporting into your product, I highly recommend you take a look at Keen IO. With a few considerations for security and performance, you'll have a solution in a fraction of the time it would take you to build from scratch.

Thanks for reading!