---
layout: post
title: "Why I was Wrong about Unit Testing - Part 1"
date: 2013-09-08 20:47
comments: true
categories: 
---

# Why I was Wrong about Unit Testing - Part 1
*Note: I wrote half of this in a crappy coffee shop the weekend before WWDC 2013. There have since been some great blogposts about iOS, Unit Testing & TDD. I hope you get some benefit out of this post. I've split it into two parts for your convenience*  

### A Bad Start
I can clearly recall my initial, visceral reaction to Unit Testing when learning JUnit at the end of a ComSci 101 module at University. It was one of confusion at the seeming pointlessness  of it all. This is a common view among Unit Test opponents; *'Write good code, and you won't break anything!'*. Despite everything that books, lecturers and my peers were saying, I continued to dismiss the efficacy of Unit Testing entirely.

I had these (admittedly *lazy*) opinions validated by Agency work, where the continual churn and of getting fixed-price projects, out severely limits the amount of time to get *anything* done. In hindsight I was making excuses for lacking the will to learn, practice or acquire the knowledge to write effective tests.

Additionally, Unit Testing doesn't exactly have the greatest pedigree in the Objective-C development community. Many of the community's [biggest names](http://blog.wilshipley.com/2005/09/unit-testing-is-teh-suck-urr.html) are evenly outwardly hostile to it. One of the biggest arguments against it is that it diverts time away from the business of *"Real Testing"* i.e. Beta Testing.  

### A New Hope
In the world of Unit Testing there has been a lot of change recently; both in terms of my own personal views and of the tools available.

My views started to change when working on larger projects and the emphasis shifts. Executing well is defined in terms of getting something out quickly less, and predictability and stability much more. 

The excuses and laziness that prevented me from getting stuck into Unit Testing began to disappear when I had to spend less time is taken up with heads in a debugger. Hesitancy to make positive changes to the quality of your codebase through refactoring fades away when Unit Tests give you a safety net to explore better ideas. 

{% img center caption /blog_images/no_idea_what_im_doing.jpg 200 200 TITLE %}

The quality and availability of tools has changed a lot recently too. There are a whole host of Open Source libraries and frameworks that feel an order of magnitude more expressive than the default [OCUnit](http://en.wikipedia.org/wiki/OCUnit). Apple are now fully on-board with Unit Testing, and to a greater extent, Continuous Integration in [XCode 5](http://images.apple.com/osx/preview/docs/OSX_Mavericks_Core_Technology_Overview.pdf)

There are host of tools that take different approaches to how to go about the process of testing. The are monolithic frameworks such as [Kiwi](https://github.com/allending/Kiwi), or you can take an a-la-carte approach in more focused libraries such as [specta](https://github.com/specta/specta). Some libraries like [OCHamcrest](https://github.com/hamcrest/OCHamcrest) and [expecta](https://github.com/specta/expecta) help simplify the creation of assertions. With the help of CocoaPods, these frameworks & libraries are [simple to integrate](https://github.com/allending/Kiwi/wiki/Up-and-Running-with-CocoaPods).

### Dodging Dogma
At first, it may seem totally backwards, to make code changes just for the purposes of box ticking. This perception isn't helped by the [Cult of Unit Testing](http://www.codingninja.co.uk/the-cult-of-unit-testing/), which overemphasises the importance of Unit Testing to the point of confusing the means with the end. 

Good iOS & Mac developers [rightly care about better applications for End Users](http://lawrencelomax.github.io/blog/2013/07/17/openness-for-the-developer-and-the-user/). This doesn't exactly fit with  with usual objective metrics and beyond programmer intuition, there are also ways of measuring [how well have tested our code](http://stephenhaunts.com/2013/02/18/unit-test-coverage-code-metrics-and-static-code-analysis/). 

It may be necessary to take actions to fit Unit Testing into the way that you think and work. Some sections of an Application's architecture have particularly big gains in terms of quality and yield great results in the future:

### Test Cases as Documentation 
There is a whole debate between [BDD and TDD](http://nshipster.com/unit-testing/), which can be fun if you like to have unimportant arguments. Whatever approach you use, you'll find that thinking about what your tests need to cover, crystallises behaviour. 

As individuals we can tend towards [over-estimating the obviousness what the code we write actually does](http://www.codinghorror.com/blog/2006/01/code-reviews-just-do-it.html). Ask a bunch of engineers to solve a particular problem and you will get more solutions than there are engineers. Header documentation is one way of expressing the behaviour of methods and classes, but are insufficient to express all behaviours.

Unit tests can be wonderfully expressive, self-documenting and clearly define the implementation. This has enormous benefit when code is shared between individuals.

Unfortunately, even the best intentions have to be ignored sometimes and *'compromises'* (hacks), may have to be used when in a tight timeframe. Documentation of these moments of voodoo is crucial in a team, this is often done through code comments. Things can get really messy later, when the voodoo is removed and the comments remain. The implementation then becomes incredibly confusing. In these instances, fragile Unit Tests can  be beneficial and the solution is clear; remove the out-of-date test and future confusion.

### State trips you up
If you have a lot of internal state within one class, Unit Testing can get a bit messy. An Object with a large amount of accumulated internal state often requires, disabling a large number of methods, further mutating the behaviour of it. In other words [*‘persistent state is the enemy of unit testing’*](http://blogs.msdn.com/b/scottdensmore/archive/2004/05/25/140827.aspx)

Mutable global state can also be troublesome to the predictability of Applications. Classes may make assumptions about global state that are not true in the clean room of a Unit Test environment. Affecting mutable global state in one test case may causes false failure, or false success in successive tests. This is very bad news for the future of your application.

Rather than avoiding writing tests because they are painful, this gives an opportunity to consider whether your implementation needs to have so much of a reliance on instance and global state. Unit Tests can highlight these giant, messy machines, more effectively than skim reading an implementation. This is especially true of global state. If you are struggling write test cases due to all of this messy state, perhaps it is time to get your [refactor hat](http://imgur.com/wGUTG) on.

![But I just need one more Singleton and everything will be solved!](images/tangled_cat.jpg)

### Dealloc is still a thing
Yes, the [```-dealloc``` method still exists](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#dealloc) in the [wonderful land of ARC](http://sealedabstract.com/rants/why-mobile-web-apps-are-slow/). ARC removes a lot of the boiler plate for object destruction, however destruction code may still have to be written; KVO must be cancelled, NSNotification observers must be de-registered.

[RSpec](http://rspec.info) has been an inspiration for a few iOS Frameworks. It has a syntax that allows you declare setup and teardown actions in code blocks that are shared between test cases. In these code blocks you can instantiate a fresh instance for each test case, where application code may be using a global instance. Instances will get created then destroyed in successive cases. By using a fresh instance for each test case, the cases do not make any false assumptions about the internal state of an object.

With the repeated creation and destruction of objects, problems from incorrect destruction code will rear their ugly head. Ignoring these problems in the present is not solving the problem, only delaying it. Unit Tests can become a stress-test for object lifecycle. If more instances are necessary in the future [correct destruction code is essential](http://blogs.msdn.com/b/scottdensmore/archive/2004/05/25/140827.aspx).

### Side Effects made Explicit 
In an imperative, object-oriented language such as Objective-C, setters may have behaviours beyond just setting a value in memory. We don't [write code in a world that enforces a strict mapping of input to output](http://en.wikipedia.org/wiki/Functional_programming) via a function, [with no side effects](http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html). The idea of setter side effects may be distasteful and dangerous for some, but they can be used to encapsulate logic inside the receiving object's class rather than requiring the caller to perform this logics.

Test Cases can describe these side effects. For example, a setter may perform an asynchronous task that writes to file with a value passed through a setter. Unit Tests make these behaviours explicit to other developers, as well as asserting that these behaviours should not dissapear with future code changes.

### Errors, Fails, Empty, Null
Even most experienced engineer can omit error conditions and fail to program defensively. Being somewhat somewhat spoiled by Objective-C, it is easy to get a false sense of security for null checking as it [isn't a language that shits the bed anytime a method is called on null](http://docs.oracle.com/javase/7/docs/api/java/lang/NullPointerException.html)

It is often easy to think that an implementation will stay frozen in time after a public release and that sections of responsibility that don't handle certain failure conditions will blow up later. A collection will not be happy with [nil keys or values](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSMutableArray_Class/Reference/Reference.html#//apple_ref/occ/cl/NSMutableArray) and then the Application will be at the mercy of an exception handler.

![Don't [throw exceptions when you can't parse a file](http://developer.apple.com/library/ios/#documentation/cocoa/conceptual/ProgrammingWithObjectiveC/ErrorHandling/ErrorHandling.html), or you will make the cat cry](images/cat_like_unit_test.jpg)

Network requests can [fail and return errors](https://github.com/AFNetworking/AFNetworking/wiki/Getting-Started-with-AFNetworking#step-4-dive-in). A lazy stream of values can [send errors](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoaFramework/ReactiveCocoa/RACSignal.h), saving a file to disk [can error](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSData_Class/Reference/Reference.html#//apple_ref/occ/instm/NSData/writeToFile:options:error:) and [saving values in a database](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectContext_Class/NSManagedObjectContext.html) can fail.

In Mobile we often strive for the leanest and most polished solution for the User. A lean Application is defined by its willingness to cut down on feature set, not cutting corners with unreliable code. The process of writing Test Cases requires us to think about all of behaviours that we expect, including Error conditions. Perhaps next time we write some, we might be a bit more aware of defensive programming techniques.

### Wrap-Up
Unit Testing is traditionally seen as a way to validate the correctness of an implementation and ensure that it does not regress in the future.

You may have to make changes to your application code to make it more testable. This may appear to be backwards at first, but persevere! Altering your implementation to make it more testable  has additional benefits. I hope to show you more of these effects in Part 2.


