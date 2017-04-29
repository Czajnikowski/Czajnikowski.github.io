---
layout: post
title: SUBQUERY Is Not That Scary
---

Let's say that you're new in the project. After short onboarding you are getting your first task, here it is.

> Add search functionality to the contacts

In description you see:

> Search results should include all contacts that match the search query in properties:
>  - name
>  - surname
>  - city

Ok, you already know some Core Data, `NSFetchedResultsController` is ready in place, all delegate methods are implemented. You just need to supply a `NSFetchRequest` - what can be easier than that?

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

Aaaaa... not that fast. `city` is not a property of a `Contact`. Additionally - single contact can have multiple addresses - they are all placed in a relation `addresses`. Actually you'd be safe and figure it out earlier if you'd use a [#keyPath expression](https://medium.com/the-traveled-ios-developers-guide/swift-3-keypath-7a3cf41e603e) (you definitely should).

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

This is exactly the situation where a predicate with a `SUBQUERY` comes handy and is the best possible performance-wise solution (see comparison of results using two other ways you can have). But `SUBQUERY` for many people is an obscure thing and no one really knows how does it work, right? Actually the concept behind it is very simple - `SUBQUERY` is a kind of an inner loop (or an inner filter) that works deep down in the Core Data stack to work out a set of objects that match a given criteria.

#### What we need in our task
{: style="text-align: center;"}

Let's for a moment step back from managed objects and Core Data stack and see what logic do we need to have in place to get the results we expect. Let's also simplify our task a bit and focus on the hard part - matching against `city` in the `addresses` relation.

Our goal is to find all contacts that have address which city matches our query, so if we'd work on a regular collection the most basic and explicit implementation we can imagine is:

{% highlight swift %}
1  let contacts: [Contact] = [...]
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
12     if addressesMatchingQuery.count > 0 {
13         results.append(contact)
14     }
15 }
{% endhighlight %}

#### Let's review some interesting parts
{: style="text-align: center;"}

In lines 1-3 we define our collection to be filtered and create initial set of results. In lines 4-15 there is actual filtering. As we don't match against any property of the contact itself we create another loop that will be matching against properties of the address objects in addresses relation need to dig deeper and filter a relation. where line 6 is our predicate. Yes, this is filtering, so why not use `filter` here? Let's rewrite it like this:

{% highlight swift %}
1 let contacts: [Contact] = [...]
2
3 let results = contacts.filter { contact in
4     for address in contact.addresses {
5         return address.city == searchQuery
6     }
7
8     return false
9 }
{% endhighlight %}

It looks good for now. Actually there is another, inner filter in lines, and we can rewrite it to look like this:

{% highlight swift %}
1 let contacts: [Contact] = [...]
2
3 results = contacts.filter { contact in
4     contact.addresses.filter { address in
5         address.city == searchQuery
6     }.count > 0
7 }
{% endhighlight %}

This is pretty clear - return all contacts that have some address which city is equal to `searchQuery`. Great our goal is achieved, so let's get back to `NSManagedObjects` and Core Data and see how we can leverage `SUBQUERY` to achieve the same goal while ensuring peak performance.

The implementation that we left of is equivalent and syntactically close to what we want to achieve with `SUBQUERY`. Let's give our code some specific layout and see them side by side:

![Double Filter Vs SUBQUERY]({{ site.url }}/img/subquery.jpg){: .center-image }

#### How does one relate to each other
{: style="text-align: center;"}

Fetch request returns a list of contacts. If it has an associated predicate it filters all contacts and returns only contacts that satisfy a given predicate. If predicate has a `SUBQUERY` the framework creates an inner filter. Filter is executed against objects from a given relation (first parameter of the `SUBQUERY`). To be able to filter objects in a relation framework needs to define a variable (second parameter of the `SUBQUERY` that starts with `$` - equivalent of `address` parameter in the closure) that is used then to define a predicate that is finally used to filter all the objects from the relation.

Thanks to great post that helped me set up a Playground https://www.andrewcbancroft.com/2016/07/10/using-a-core-data-model-in-swift-playgrounds/
