---
layout: post
title: Anonymous nil
---

Anonymous `nil` is a subtle code smell that is present in many codebases. What is it? Why is it a problem? How can we solve it?

Let's take a brief, made up and very contrived example. Let's say there is a method in your codebase that shows view with an animation.

{% highlight swift %}
func show(withAnimation animation: Animation?)
{% endhighlight %}

You've seen this invocation in your code many, many times.

{% highlight swift %}
show(withAnimation: nil)
{% endhighlight %}

### What does it mean that `animation` parameter is `nil`?

Every time you've been going through this line of code you've been asking yourself this question.

There are at least four ways you can figure it out:

- If you work in a team and API is a craft of your teammates you can ask them
- But why should you - there's no secrets and after some time of digging you can easily get the answer
- You can run your code couple of times or unit test it (if possible) and figure it out
- You can assume that it is reasonably written and `nil` as a parameter means "no animation at all"

Whatever way you decided to follow the answer is:

> "If you pass `nil` to this method your view will be presented with animation, that starts from the bottom of the screen."

![Why?]({{ site.url }}/img/why.jpg){: .center-image }

Obvious, right? No, it's not obvious. It's not even logical. It's because the example is very contrived, but I hope you get what the problem is.

### What's the problem?

Whatever choice you've made to get to this information why have you had to ask this question to yourself at all? Good API is self explanatory and every necessity to explain what it does or how it will perform given certain input is a failure of it's designer. It's like with physical things - the best designs doesn't have to be explained, you just get it.

The problem here originates from optional overuse. The meaning of optional is strict, I mean: ["there is a value, and it equals x or there isn’t a value at all"](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html). **Nothing more. That's it**. In our case the definition of `nil` is "default animation from the bottom of the screen". **This `nil` has a meaning, but it's meaning is different. This is what I call an anonymous `nil` - it's a `nil` that is used as an actual parameter or variable/constant that means something more than "no value at all".**

### What are the consequences?

Wherever you use `nil` as a substitute for a default value you're making users of your API sad (including future self):

- They can waste time digging to figure out what do this `nil` mean (sometimes the answer may be hidden really deep down)
- They can encounter many other methods that use similar signature, but every time they see them they need to ask themselves if the rule from the other method still holds true in similar places?
- They need to maintain volatile meta-knowledge about the API in their head
- They can even have wrong assumptions about the system's behaviour

### How can you make it better?

Often we can't afford to make a proper dependency injection or DI is not the right tool to solve this kind of issue. Let's try to understand the problem a bit more and come up with feasible, lightweight solution.

In our case there are two possible outcomes for this function:

- Animation will pop up from the bottom
- Specific, non-`nil` animation will be executed

Isn't it like an enum with two cases? Yes, indeed. What if we use such enum in our code?

That's simple.

{% highlight swift %}
enum WrappedAnimation {
    case fromBottom
    case specific(Animation)
}
{% endhighlight %}

Now there is **no ambiguity** on call side:

{% highlight swift %}
func show(withWrappedAnimation: .fromBottom)
{% endhighlight %}

How about implementation side?

To be able to use Animation object inside we need to extend our enum to unwrap the wrapped value.

{% highlight swift %}
enum WrappedAnimation {
    case fromBottom
    case specific(Animation)

    func unwrapped() -> Animation {
        switch self {
        case .fromBottom:
            return createFromBottomAnimation()     //     Here is a meaning of the nil
        case .specific(let animation):
            return animation
        }
    }
}
{% endhighlight %}

So somewhere deeper we can write:

{% highlight swift %}
func perform(wrappedAnimation: WrappedAnimation) {
    let animation = wrappedAnimation.unwrapped()
    animation.perform()
}
{% endhighlight %}

You can easily adapt this solution to be generic or use parameters in unwrapped method.

The example here is contrived and oversimplified, but I hope I've highlighted an issue and sensitize you to this subtle problem.
