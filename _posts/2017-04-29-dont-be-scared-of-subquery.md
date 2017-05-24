---
layout: post
title: SUBQUERY Is Not That Scary
---

Let's say that you're new in the project. After short onboarding you are getting your first task, here it is:

> Add search functionality to the contacts

In description you see:

> Search results should include all contacts that match the search query by:
>
>  - name
>  - surname
>  - city

Ok, UI with all delegate methods is implemented, you know some Core Data, `NSFetchedResultsController` is already in place, you just need to supply a `NSFetchRequest` - what can be easier than that?

#### Nothing easier than that
{: style="text-align: center;"}

{% highlight swift %}
let fetchRequest = NSFetchRequest<Contact>(
    entityName: Contact.entity().name!
)
fetchRequest.predicate = NSCompoundPredicate(
    orPredicateWithSubpredicates: [
        "name",
        "surname",
        "city"
        ]
        .map {
            NSPredicate(
                format: "SELF.%K == %@",
                $0,
                searchQuery
            )
        }
)
{% endhighlight %}

#### Boom, done, let's give it a try!
{: style="text-align: center;"}

`** Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<NSDictionaryMapNode 0x608002079b40> valueForUndefinedKey:]: this class is not key value coding-compliant for the key city.'`

Aaaaa... not that fast. `city` is not a property of a `Contact`. Actually you'd be safe and figured it out earlier if you'd use a [#keyPath expression](https://medium.com/the-traveled-ios-developers-guide/swift-3-keypath-7a3cf41e603e) (you definitely should), but that's not my goal in this post. We need to dig deeper. 

#### Ok, so here's the model
{: style="text-align: center;"}

{% highlight swift %}
public class Contact: NSManagedObject {
    @NSManaged public var name: String
    @NSManaged public var surname: String

    @NSManaged public var addresses: Set<Address>
}

public class Address: NSManagedObject {
    @NSManaged public var city: String
}
{% endhighlight %}

So `city` is a property of the `Address` entity. Additionally single contact can have multiple addresses - they are all placed in the `addresses` relation.

Our `NSFetchRequest` is configured to iterate over (and return) `Contact` objects, so with a regular `NSCompoundPredicate(orPredicateWithSubpredicates: [..., NSPredicate("SELF.city == %@")]` we cannot achieve what we need. We need to iterate over all `addresses`. We need some kind of predicate that within a given `Contact` will iterate over all addresses to decide if any of them matches our search query (via `city` property). This is exactly the situation where a predicate with a `SUBQUERY` comes handy and is the best possible performance-wise solution (see comparison of results using two other ways you in [my playground](https://github.com/Czajnikowski/Playgrounds/tree/master/Subquery.playground)).

For many people `SUBQUERY` is an obscure topic but actually the concept behind it is very simple - `SUBQUERY` is a kind of an inner loop (or an inner filter) that works [deep down in the Core Data stack](https://www.objc.io/issues/4-core-data/core-data-fetch-requests/) to work out a set of objects that matches a given criteria.

#### How to understand it?
{: style="text-align: center;"}

Let's for a moment step back from managed objects and Core Data and see what logic do we need to have in place to get the results we expect. Let's also simplify our task a bit and focus on the hard part - matching against `city` in the `addresses` relation.

Our goal is to find all contacts that have address which city matches our query, so if we'd work on a regular collection the most basic and explicit implementation we can imagine is:

{% highlight swift %}
1  let contacts: [Contact] = ...
2
3  var results = [Contact]()
4  for contact in contacts {
5      var addressesMatchingQuery = [Address]()
6      for address in contact.addresses {
7          if address.city == searchQuery {
8              addressesMatchingQuery.append(address)
9          }
10     }
11
12     addressesMatchingQuery.count > 0 ? results.append(contact) 
13 }
{% endhighlight %}

#### Let's rewrite it a bit
{: style="text-align: center;"}

In lines 1-3 we define our collection to be filtered and create initial set of results. We expect a `results` collection to be a subset of `contacts`, so we may simplify our code by applying a `filter` method. Let's rewrite it like this:

{% highlight swift %}
1  let contacts: [Contact] = ...
2
3  let results = contacts.filter { contact in
4      var addressesMatchingQuery = [Address]()
5      for address in contact.addresses {
6          if address.city == searchQuery {
7              addressesMatchingQuery.append(address)
8          }
9      }
10
11     return addressesMatchingQuery.count > 0
12 }
{% endhighlight %}


In lines 4-9 we have the same kind of situation again - we have an inner filter here - we expect to get a subset of `contact.addresses` into `addressesMatchingQuery` based on the value of the expression in line 6. Let's use a `filter` method again:

{% highlight swift %}
1 let contacts: [Contact] = ...
2
3 let results = contacts.filter { contact in
4     let addressesMatchingQuery = contact.addresses.filter { address in
5         address.city == searchQuery
6     }
7
8     return addressesMatchingQuery.count > 0
9 }
{% endhighlight %}

Let's take it to the extreme and compact it even more:

{% highlight swift %}
1 let contacts: [Contact] = ...
2
3 let results = contacts.filter { contact in
4     contact.addresses.filter { address in
5         address.city == searchQuery
6     }.count > 0
9 }
{% endhighlight %}

Great, this is pretty clear - return all contacts that have more than zero addresses which `city` property is equal to `searchQuery`. Let's leave it like this and now get back to our `NSFetchRequest` and Core Data and see how our implementation actually matches what `SUBQUERY` has to offer.

#### How does one relate to another?
{: style="text-align: center;"}

This is a `SUBQUERY` that is **functionally equivalent** to our non Core Data implementation (if applied to our `NSPredicate`).

{% highlight swift %}
"SUBQUERY(addresses, $address, $address.city ==[cd] \"\(searchQuery)\").@count > 0"
{% endhighlight %}

The first parameter of the `SUBQUERY` is a relation name, second one is a temporary variable name that is used to define a predicate - a third parameter - that is used to match against every object from the relation. And yes - second parameter doesn't need to be named `$x`.

There is a reason why I've taken you through that non Core Data implementation and then shown you this. That's because two constructs are doing the same thing and are **almost identical syntactically**. If we give our code some specific layout we can see it more clearly (be sure to zoom in the image below):

<img id="myImg" src="{{ site.baseurl }}/img/subquery.jpg" alt="Double Filter Vs SUBQUERY">

So a `SUBQUERY` behaves like an inner filter that works on a relation configured as a first parameter, variable (that starts with the `$` symbol) that is equivalent to the parameter of the inner `filter`'s closure, and the predicate/condition that is built upon that variable.

I bet you can easily tweak our initial implementation now to leverage `SUBQUERY` and make a good first impression on your new team colleagues. Sometimes things are really simple, but it all depends on the right perspective.

If you want to play with the code, or see the banchmark of three different strategies of matching that I've mentioned before you can find my playground [here](https://github.com/Czajnikowski/Playgrounds/tree/master/Subquery.playground).
