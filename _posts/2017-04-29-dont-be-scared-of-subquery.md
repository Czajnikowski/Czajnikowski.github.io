---
layout: post
title: Don't Be Scared of SUBQUERY
---

Let's say that you're new in a project. After short onboarding you are getting your first task. Here it is.

"Add search functionality to contacts"

In description you see:

"Search results should include all contacts that match the search query in properties:
- name
- surname
- city"

Ok, you already know some Core Data, NSFetchedResultsController is ready in place, all delegate methods are implemented. You just need to supply a NSFetchRequest - what can be easier than that?

Nothing easier than that:

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

Boom, done, let's give it a try.

** Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<NSDictionaryMapNode 0x608002079b40> valueForUndefinedKey:]: this class is not key value coding-compliant for the key city.'

Aaaaa... not that fast. city is not a property of a Contact. Additionally - single contact can have multiple addresses - they are all placed in a relation Contact.addresses. Actually you'd be safe and figure it earlier if you'd use (you definitely should) a [#keyPath expression](https://medium.com/the-traveled-ios-developers-guide/swift-3-keypath-7a3cf41e603e).

Ok, so here's the model:
public class Contact: NSManagedObject {
    @NSManaged public var name: String
    @NSManaged public var surname: String

    @NSManaged public var addresses: Set<Address>
}

public class Address: NSManagedObject {
    @NSManaged public var city: String
}

This is exactly the situation where a predicate with a SUBQUERY comes handy and is the best possible performance-wise solution (see comparison of results using two other ways you can have). But SUBQUERY is such an obscure thing that no one really knows how it works, right? Actually the concept is very simple - SUBQUERY is a kind of an inner loop (or an inner filter) that works deep down in the Core Data stack to work out a set of objects that match a given criteria.

Really?

Let's for a moment step back from managed objects and Core Data stack and see what logic do we need to have in place to get the results we expect. Let's also simplify our task a bit and focus on the hard part - matching against city in the addresses relation.

Our goal is to find all contacts that have address which city matches our query, so if we'd work on a regular collection the most basic and explicit implementation we can imagine is:

1  let contacts = [Contact]()
2
3  var results = [Contact]()
4  for contact in contacts {
5      var addressesMatchingQuery = [Address]()
6      for address in contact.addresses {
7          if address.city == "Kąkolówka" {
8              addressesMatchingQuery.append(address)
9          }
10     }
11
12     if addressesMatchingQuery.count > 0 {
13         results.append(contact)
14     }
15 }

Let's review some interesting places:

In lines 1-3 we define our collection to be filtered and create initial set of results. In lines 4-15 there is actual filtering. As we don't match against any property of the contact itself we create another loop that will be matching against properties of the address objects in addresses relation need to dig deeper and filter a relation. where line 6 is our predicate. Yes, this is filtering, so why not use filter here. Let's rewrite it like this:

1 let contacts: [Contact] = [/*List of contacts...*/]
2
3 let results = contacts.filter { contact in
4     for address in contact.addresses {
5         return address.city == "Kąkolówka"
6     }
7
8     return false
9 }

It looks good for now. Actually there is another, inner filter here, we can rewrite it like this:

1 let contacts: [Contact] = [/*List of contacts...*/]
2
3 results = contacts.filter { contact in
4     contact.addresses.filter { address in
5         address.city == "Kąkolówka"
6     }.count > 0
7 }

This is pretty clear - return all contacts that have some address which city is equal to "Kąkolówka". Great our goal is achieved, so let's get back to NSManagedObjects and Core Data and see how we can leverage SUBQUERY to achieve the same goal while ensuring peak performance.

The implementation that we left of is equivalent and syntactically close to what we want to achieve with subquery. Let's give our code some specific layout and see them side by side:

Fetch request returns a list of contacts.
If it has an associated predicate it filters all contacts and returns only contacts that satisfy a given predicate.
If predicate has a SUBQUERY the framework creates an inner filter.
Filter is executed against objects from a given relation (first parameter of the SUBQUERY).
To be able to filter objects in a relation framework needs to define a variable (second parameter of the SUBQUERY that starts with $ - equivalent of address closure parameter) that is used then to define a predicate that is finally used to filter all the objects from the relation.

Thanks to great post that helped me set up a Playground https://www.andrewcbancroft.com/2016/07/10/using-a-core-data-model-in-swift-playgrounds/


















`reduce()` is a very useful method in Swift Standard Library. It works like a `for in` loop ([it is the one in fact](https://github.com/apple/swift/blob/master/stdlib/public/core/SequenceAlgorithms.swift.gyb#L616)), but it's designed specifically to calculate a single value out of the elements in the collection... There is only one difference.

#### We cannot do this:
{: style="text-align: center;"}

{% highlight swift %}
reduce(initialValue) {
     ...
     if resultIsGoodEnough($0) {
          break // and return current $0
     }
}
{% endhighlight %}

This post is actually inspired by a [question](http://stackoverflow.com/questions/29906765/is-there-a-way-to-break-from-an-arrays-reduce-function-in-swift) on Stack Overflow and I'm gonna show you how I've implemented it as an extension method on `Sequence`. For TL;DR jump to the <a href="#hereItIs">last section of the post</a> or check out my playground [here](https://github.com/Czajnikowski/Playgrounds).

#### We can do this:
{: style="text-align: center;"}

{% highlight swift %}
reduce(initialValue) {
     ...
     if resultIsGoodEnough($0) {
          return $0
     }
}
{% endhighlight %}

But now the `if` statement becomes part of the `nextPartialResult` closure and clutters the logic here.
That's not good. Let's imagine the better solution, how about this:

{% highlight swift %}
reduce(initialValue, { ... }, until: resultIsGoodEnough)
{% endhighlight %}

That's better!

Let's see how we can implement it. The original reduce method is implemented on the `Sequence` protocol, so let's put an overload in the extension (you can find code in my playground [here](https://github.com/Czajnikowski/Playgrounds)):

{% highlight swift %}
extension Sequence {
    func reduce<Result>(
        _ initialResult: Result,
        _ nextPartialResult: (Result, Self.Iterator.Element) throws -> Result,
        until conditionPassForResult: (Result) -> Bool
        ) rethrows -> Result {

        return try reduce(
            initialResult,
            {
                if conditionPassForResult($0) {
                    return $0
                }
                else {
                    return try nextPartialResult($0, $1)
                }
            }
        )
    }
}
{% endhighlight %}

Looks good, let's give it a try:

{% highlight swift %}
(1 ... 5).reduce(0, +, until: { partialSum in partialSum > 5 }) // Returns 6
{% endhighlight %}

"Reduce a sequence of numbers with `+` operator until the partial sum becomes greater then 5". It reads nicely, but there is one more issue.

If you run it in a playground (you can find a prepared playground [here](https://github.com/Czajnikowski/Playgrounds)) you'll see *(6 times)*, and yes - if you increase the size of your sequence to 500 elements it'll say *(501 times)* times. That's because our condition closure is getting called n times, (where n is the size of the collection), + 1 is a call to `reduce`). So why do we need to invoke it as many times?

![Infinite Loop]({{ site.url }}/img/dog.gif){: .center-image }

Computers and smartphones are fast, but we don't want them to be wasteful and behave like that dog. In our case 3 iterations is all we really need...

In our overload (as in the original declaration) of the `reduce` method  we can see that `nextPartialResult` that we're implementing in our overload can throw an exception, so why not leverage it - throw an exception when we pass our condition, handle it so it won't escape the scope of our extension and return a result.

To achieve it we'll need an `Error` conforming type that will pass a `Result` object to the catch closure:

{% highlight swift %}
enum BreakConditionError<Result>: Error {
    case conditionPassedWithResult(Result)
}
{% endhighlight %}

To finish it let's have a full-fledged implementation, that allows us to make a "break it or make it" decision not only based on the partial result, but on the current element of the collection as well (although if it's gonna be sth like `element == someThing` you should rather use `prefix(while:)` before `reduce`).

#### <a name="hereItIs">Here it is:</a>
{: style="text-align: center;"}

{% highlight swift %}
extension Sequence {
    func reduce<Result>(
        _ initialResult: Result,
        _ nextPartialResult: (Result, Self.Iterator.Element) throws -> Result,
        until conditionPassFor: (Result, Self.Iterator.Element) -> Bool
        ) rethrows -> Result {

        do {
            return try reduce(
                initialResult,
                {
                    if conditionPassFor($0, $1) {
                        throw BreakConditionError
                            .conditionPassedWithResult($0)
                    }
                    else {
                        return try nextPartialResult($0, $1)
                    }
                }
            )
        }
        catch BreakConditionError<Result>
            .conditionPassedWithResult(let result) {
                return result
        }
    }
}
{% endhighlight %}

There is another variation of it and you should use it if you expect to reduce collection while the given condition is true:

{% highlight swift %}
extension Sequence {
    func reduce<Result>(
        _ initialResult: Result,
        _ nextPartialResult: (Result, Self.Iterator.Element) throws -> Result,
        while conditionPassFor: (Result, Self.Iterator.Element) -> Bool
        ) rethrows -> Result {

        do {
            return try reduce(
                initialResult,
                {
                    let _nextPartialResult = try nextPartialResult($0, $1)
                    if conditionPassFor(_nextPartialResult, $1) {
                        return _nextPartialResult
                    }
                    else {
                        throw BreakConditionError
                            .conditionPassedWithResult($0)
                    }
                }
            )
        }
        catch BreakConditionError<Result>
            .conditionPassedWithResult(let result) {
                return result
        }
    }
}
{% endhighlight %}

It gives a different result, so you can use it when you don't want to include the element above condition (as it is a case in `until` variation):

{% highlight swift %}
(1 ... 5).reduce(0, +, while: { partialSum, _ in partialSum < 5 }) // Returns 3
{% endhighlight swift %}

#### So that's it!
{: style="text-align: center;"}

I hope you enjoyed my second post. Feel free to give it a try, you can find the sample playground [here](https://github.com/Czajnikowski/Playgrounds). If you want to break a `reduce` - you can :P
