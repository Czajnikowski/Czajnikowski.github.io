---
layout: post
title: Break Out of reduce
---

`reduce` is a very useful method in Swift Standard Library. It works like a `for in` loop, but it's designed specifically to calculate a single value out of the elements in the collection... There is only one difference.

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

In my post I'm gonna show you how I've implemented it as an extension method on `Sequence`. For TL;DR jump to the last code snippet.

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

Let's see how we can implement it. The original reduce method is implemented on the `Sequence` protocol, so let's put an overload in the extension:

{% highlight swift %}
extension Sequence {
    func reduce<Result>(
        _ initialResult: Result,
        _ nextPartialResult: (Result,
            Self.Iterator.Element) throws -> Result,
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

Looks good, let's give it a try in a playground:

{% highlight swift %}
(1 ... 5).reduce(0, +, until: { result in result > 5}) // Returns 6
{% endhighlight %}

"Reduce a sequence of numbers with `+` operator until the result becomes greater then 5". It reads nicely, but there is one more issue. If you run it in a playground you'll see *(6 times)*, and yes - if you increase the size of your sequence to 500 elements it'll say *(501 times)* times. Why do we need to invoke `nextPartialResult` so many times? In our case 3 iterations is all we need...

In our overload (as in the original declaration) of the `reduce` method  we can see that `nextPartialResult` that we're implementing in our overload can throw an exception, so why not leverage it - throw an exception when we pass our condition, handle it so it won't escape the scope of our extension and return a result.

To achieve it we'll need an `Error` conforming type that will pass a `Result` object to the catch closure:

{% highlight swift %}
enum BreakConditionError<Result>: Error {
    case conditionPassedWithResult(Result)
}
{% endhighlight %}

#### That's it:
{: style="text-align: center;"}

{% highlight swift %}
extension Sequence {
     func reduce<Result>(
        _ initialResult: Result,
        _ nextPartialResult: (Result,
            Self.Iterator.Element) throws -> Result,
        until conditionPassForResult: (Result) -> Bool
        ) rethrows -> Result {

        do {
            return try reduce(
                initialResult,
                {
                    if conditionPassForResult($0) {
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
        while conditionPassForResult: (Result) -> Bool
        ) rethrows -> Result {

        do {
            return try reduce(
                initialResult,
                {
                    let _nextPartialResult = try nextPartialResult($0, $1)
                    if conditionPassForResult(_nextPartialResult) {
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
(1 ... 5).reduce(0, +, while: { result in result < 5}) // Returns 3
{% endhighlight swift %}

#### So that's it!
{: style="text-align: center;"}

Hope you enjoyed my second post, there are more to come. If you want to break a reduce method - you can :P
