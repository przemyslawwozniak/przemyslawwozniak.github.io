---
title:  "Book review: Functional Programming in Java"
date:   2020-03-10 10:00:00 +0100
categories: reviews backend
excerpt: Functional Programming in Java is an entry-level 200 pager, which gives a nice overview of a programming style made possible since 2014 with the debut of JDK8.
---

Recently I’ve appreciated shorter technical books which promises immediate results. These are especially good choice when you just want to refresh a topic. That was a case for me with functional programming in Java, as I’ve just started to work on an enterprise level project, which utilizes reactive programming. One does not simply code with Reactor and Spring WebFlux without proper understanding on functional programming.

Having this in mind, I’ve chosen *Functional Programming in Java* by Venkat Subramaniam - as a Java developer, you probably associate the guy. The book has rather high overall ratings and is about 180 pages, which seemed to meet my needs. As I’m right afer the lecture (which took me only 2 days, while still working 8 hours and being a father of lovely 2 year old), I’d like to left some of my thoughts in this post.

Chapters 1, 2 and 3 are introductory stuff. Starting with the first one, you’ll learn what kind of edge functional programming has over other styles. The goal is, as always, to stimulate your apetite, but the points given are right. Key takeaways are:
* functional programming is declarative, not imperative (you say, what you want to achieve rather than giving step by step instructions), which makes your code more expressive and readable (and we all know, less code is lower potential for bugs)
* functional programming promotes best coding practices like immutability and reducing side effects (so preferring expressions - actions, which returns something - rather than statements - actions, which returns ‚void’)
* thanks to that, your functional programming is well-prepared for parallel execution and indeed, parallelization wasn’t easier than now
The second chapter goes over working with collections the functional way, while the third one is about working with Strings, comparators and filters. These are mostly focused on exploring the API with focus on so called MapReduce pattern. Nothing too fancy, to be honest.

Chapter 4 is entitled „Designing with Lambda Expressions” and it’s probably the best one in the whole book. I must admit that throughout the book, Venkat pushes the idea that functional programming is a design technique rather than „just coding style” and in this chapter, he really shines on that idea. His approach is rewriting well-known design patterns functional way and I have to admit, these one are much more better than the original, object-oriented one. Following design patterns were taken on a workshop:
* strategy design pattern, which allows separation of concerns on a method level (the implementation in this case is very simplistic and, given the usefulness of the pattern, should be better „taken care of”)
* decorator design pattern - fits brilliant into functional programming paradigm! Have a look yourself:
{% highlight java linenos %}
public class Camera {
    private Function<Color, Color> filter;

    public Color capture(final Color inputColor) {
        final Color processedColor = filter.apply(inputColor);
        return processedColor;
    }

    public void setFilters(final Function<Color, Color>... filters) {
        filter = Stream.of(filters)
                        .reduce((filter, next) -> filter.compose(next))
                        .orElseGet(Function::identity)      //same as: orElse(color -> color)
    }

    public Camera() { 
        setFilters(); 
    }
}
{% endhighlight %}
* creating fluent interfaces on the example of builder pattern (thanks to Lombok, we don’t have to produce this boilerplate code anymore)
Of course, one can say only three design patterns were demoed in functional style, but the point here is to „show the way” rather than provide implementation. Not only here, but in general, Venkat tries to underline the importance of a mind shift and in my opinion descriptions make it possible.

Chapter 5, „Working with Resources”, is exactly what it says it is and is rather boring, so I’ll skip right to the following one, „Being Lazy”. I’d say this one wins the silver medal. It describes what benefits laziness of Streams brings to the performance of the application, and these are quite clever beasts! You’ll also gain some very simple tips on how to lazy evaluate methods arguments. But far more important aspect is how to lazy initialize heavy objects and indeed, the solution given is worth being quoted here, as it ensures thread safety and avoids null references:
{% highlight java linenos %}
public class Holder {
  private Supplier<Heavy> heavy = () -> createAndCacheHeavy();

  public Holder() {
    System.out.println("Holder created");
  }

  public Heavy getHeavy() {
    return heavy.get();
  }

  private synchronized Heavy createAndCacheHeavy() {
    class HeavyFactory implements Supplier<Heavy> {
      private final Heavy heavyInstance = new Heavy();

      public Heavy get() { return heavyInstance; }
    }

    if(!HeavyFactory.class.isInstance(heavy)) {
      heavy = new HeavyFactory();
    }

    return heavy.get();
  }
}
{% endhighlight %}

Chapter 7 is about „Optimizing Recursions”, and to be honest, I skipped it. Not that I don’t see the value in recursions, I simply don’t need it now and the time is precious, as we all know. Chapter 8, „Composing with Lambda Expressions”, has a promising name and begins quite interesting by introducing the concept of „hybrid OOP-functional style”, which is about transforming lightweight objects into other objects rather than mutating state of one particular object. But later on, it’s nothing more than that:
{% highlight java linenos %}
public static void findHighPriced(final Stream<String> symbols) {
final StockInfo highPriced = symbols.map(StockUtil::getPrice)
                                    .filter(StockUtil.isPriceLessThan(500))
                                    .reduce(StockUtil::pickHigh)
                                    .get();
}
{% endhighlight %}
(Strings being transformed into Strings and so on - boring!)
Chapter 9, on the other hand, is just a summary, what I’d call „the golden rules of functional programming”:

  *   More declarative, less imperative
  *   Immutability
  *   Reduce side effects (referential transparency - you can substitute function call with returned value and nothing changes from the program perspective)
  *   Prefer expressions (action + returns result) over statements (action + no result returned)
  *   Design with higher-order functions (pass functions as methods arguments)

And that’s it! Not too much, but fast refresher, which goes above the API methods (so the thing you’ll read about on average webpage - but that’s not the true spirit of functional programming). Besides the things that I’ve already said (mostly simplistic examples), I’m not a big fan of the „writer’s workshop” of Venkat - texts are rather lengthy, there’s a lot of „funny content”, off topic and so on. But I know, it’s the same for me, so I’ll end here.
