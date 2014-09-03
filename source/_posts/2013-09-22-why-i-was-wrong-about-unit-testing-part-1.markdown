---
layout: post
title: "Why I was Wrong about Unit Testing - Part 1"
date: 2013-09-22 20:45
comments: true
categories:
- Testing 
- iOS 
- Patterns 
- Xcode 
- Pragmatism
published: true
---

*Note: I wrote half of this in a crappy coffee shop the weekend before WWDC 2013. There have since been some great blogposts about iOS, Unit Testing & TDD. I hope you get some benefit out of this post. I've split it into two parts lest anyone fall asleep*  

### A Bad Start
I can clearly recall my initial, visceral reaction to Unit Testing when learning JUnit at the end of a ComSci 101 module at University. It was one of confusion at the seeming pointlessness  of it all. This is a common view among Unit Test opponents; *'Write good code, and you won't break anything!'*. Despite everything that books, lecturers and my peers were saying, I continued to dismiss the efficacy of Unit Testing entirely.

I had these (admittedly *lazy*) opinions validated by Agency work, where the continual churn of fixed-price projects, out severely limits the amount of time to get *anything* done. In hindsight I was making excuses for lacking the will to learn, practice or acquire the knowledge to write effective tests.

Additionally, Unit Testing doesn't exactly have the greatest pedigree in the Objective-C development community. Many of the community's [biggest names](http://blog.wilshipley.com/2005/09/unit-testing-is-teh-suck-urr.html) are outwardly hostile to it. 
### A New Hope
In the world of Unit Testing there has been a lot of change recently; both in terms of my own personal views and of the tools available.

My views started to change when working on longer term and continuously developed products. As before, the Success of a Product is measured in terms of how it performs in the market. However, the quality of the codebase in terms of stability and documentation becomes increasingly important.

The excuses and laziness that prevented me from getting stuck into Unit Testing began to disappear when I had to spend less time is taken up with heads in a debugger. Hesitancy to make positive changes to the quality of your codebase through refactoring fades away when Unit Tests give you a safety net and sanity check to explore better ideas. 

The quality and availability of tools has changed a great deal recently too. There are a whole host of Open Source libraries and frameworks that in practice can be an order of magnitude more expressive than the default [OCUnit](http://en.wikipedia.org/wiki/OCUnit) and [XCTest](https://developer.apple.com/technologies/tools/). Apple are now fully on-board with Unit Testing, and to a greater extent, Continuous Integration in [XCode 5](http://images.apple.com/osx/preview/docs/OSX_Mavericks_Core_Technology_Overview.pdf)

There are host of tools that take different approaches to how to go about the process of testing. The are monolithic frameworks such as [Kiwi](https://github.com/allending/Kiwi), or you can take an a-la-carte approach in libraries that handle specific tasks, such as [specta](https://github.com/specta/specta) for assertions, [OCHamcrest](https://github.com/hamcrest/OCHamcrest) for matching, [expecta](https://github.com/specta/expecta) for asynchronous testing and [OCMock](http://ocmock.org) for mocking Objects. With the help of CocoaPods, these frameworks & libraries are [simple to integrate](https://github.com/allending/Kiwi/wiki/Up-and-Running-with-CocoaPods).

### Dodging Dogma
At first, it may seem totally backwards, to make code changes just for the purposes of box ticking. This perception isn't helped by the [Cult of Unit Testing](http://www.codingninja.co.uk/the-cult-of-unit-testing/), which overemphasises the importance of Unit Testing to the point of confusing the means with the end. 

Good iOS & Mac developers [rightly care about better applications for End Users](http://lawrencelomax.github.io/blog/2013/07/17/openness-for-the-developer-and-the-user/). This doesn't exactly fit with  with usual objective metrics and beyond programmer intuition, there are also ways of measuring [how well have tested our code](http://stephenhaunts.com/2013/02/18/unit-test-coverage-code-metrics-and-static-code-analysis/). 

It may be necessary to take actions to fit Unit Testing into the way that you think and work. Some sections of an Application's architecture have particularly big gains in terms of quality and yield great results in the future:

### Test Cases as Documentation 
There is a whole debate between [BDD and TDD](http://nshipster.com/unit-testing/), which can be heaps of fun if you like to have unimportant arguments about methodoligies that achieve a similar result. Whatever approach you use, you'll find that thinking about what your tests need to cover, crystallises behaviour as you write. 

As individuals, it is all too common to [over-estimate obviousness in behaviour, for the code we write](http://www.codinghorror.com/blog/2006/01/code-reviews-just-do-it.html). Ask a handful of engineers to solve a particular problem and you will receive a chorus of solutions. Header documentation is one way of expressing the behaviour of methods and classes, but this is behaviour that you wish to make public. Headers should remain succinct and avoid expressing internal, private behaviours. Unit Tests can certainly help out here.

When developing in teams of more than a few people, a great deal of programmer intent can get lost in a stream of commits. This is especially true of teams that operate remotely. Unit tests can be wonderfully expressive, self-documenting and clearly define the implementation. Commits containing behaviour with corresponding tests, gives explaination in a clear and often concise manner.

Unfortunately, even the best intentions have to be put aside sometimes and *'compromises'* (hacks), may have to be used when in a tight timeframe. Documentation of these moments of voodoo is crucial in a team, this is often done through code comments. Some [refactorings](http://imgur.com/wGUTG) later the voodoo may be removed when a better implementation comes along. The voodoo is removed with out-of-date commit comments remaining as a source of incorrect documentation. Removing and amending tests during refactoring keeps consistency between the documentation and the implementation as well as a living commentary of code changes such as refactoring.

### State trips you up
If you have a lot of internal state within one class, Unit Testing can get a bit messy. An Object with a large amount of accumulated internal state can require disabling a large number of methods with stubbing, or setting many private and public properties to get the Object in a state suitable for performing certain behaviours. This effort is mostly unnecessary, as there is often an implementation of a Class that can have the same behaviour without the same amount of state. 

Not only can state pose a problem in terms of effort required to just run some simple Unit Tests, it can have negative effects on the predictability of an Application. Conditionals can be dependent on internal state. This makes modifying a Class for others a difficult task; often requiring intimate knowledge of how all the state interacts.

Mutable Global State can also be troublesome to the predictability of Applications. Classes may make assumptions about global state that are not true in the clean room of a Unit Test environment. Affecting mutable global state in one test case may causes false failure, or false success in successive tests. The correctness of tests is crucial to the future of your application.

One example of this is seen with [abuses in the usage of Singletons](http://blogs.msdn.com/b/scottdensmore/archive/2004/05/25/140827.aspx). Unit Testing a singleton can be very troublesome, as Singletons can tend to contain all many assumptions about the data that they consume, as well as storing state at the instance level, which is in effect global. Increasing the data dependencies of a Class increases its fragility to changes in the rest of the Application.

At first it may seem that the solution to giant this is refactoring via chains of Singletons with more specific responsibilities. On face value this appears to be a solution, since state is siloed into separate classes, decreasing the amount of accumulated state per class. The assumptions about consumed data still exist, just over more Classes. Writing [methods for these classes that take arguments](https://twitter.com/slicknet/status/372798743948824576) and do not mutate internal state as far as possible makes Singletons far more testable.

Rather than avoiding writing tests because they are painful, there is an opportunity to consider whether an implementation needs to have so much of a reliance on instance and global state. Unit Tests can highlight these giant, messy machines, more effectively than skim reading an implementation. As has been wisely observed, [*‘persistent state is the enemy of unit testing’*](http://blogs.msdn.com/b/scottdensmore/archive/2004/05/25/140827.aspx)


### Dealloc is still a thing
Yes, the [```-dealloc``` method still exists](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#dealloc) in the [wonderful land of ARC](http://sealedabstract.com/rants/why-mobile-web-apps-are-slow/). ARC removes a lot of the boiler plate for object destruction, however destruction code may still have to be written; KVO must be cancelled, NSNotification observers must be de-registered and any C  ```malloc()``` must be ```free()```'d.

[RSpec](http://rspec.info) has been an inspiration for a few iOS Frameworks. It has a syntax that allows you declare setup and teardown actions in nested code blocks, shared between test cases. In these code blocks a fresh instance can be instantiated  for each test case, where application code may be only ever use a single instance. By using a fresh instance for each test case, the cases do not make any false assumptions about the internal state of an object.

With the repeated creation and destruction of objects, problems from incorrect destruction code will rear their ugly head. Ignoring these problems in the present does not solve the problem, it only delays it. Unit Tests can act as stress-test for object lifecycle, profoundly affecting the ease of moving away from Singletons.

### Side Effects made Explicit 
In an imperative, object-oriented language such as Objective-C, setters may have behaviours beyond just setting a value in memory. We don't [write code in a world that enforces a strict mapping of input to output](http://en.wikipedia.org/wiki/Functional_programming) via a function, [with no side effects](http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html). The idea of setter side effects may be distasteful and dangerous for some, since the public interface does not always convey this information. However, setter side effects can be used to encapsulate logic inside the receiving object's class, rather than requiring the caller to call additional methods after a setter.

Test Cases can describe these side effects. For example, a setter may perform an asynchronous task that writes to file. Unit Tests make these behaviours explicit to other developers, as well as asserting that these behaviours should not disappear with future code changes.

### Errors, Fails, Empty, Null
Even the most experienced engineer can omit error conditions and fail to program defensively. Being somewhat somewhat spoiled by Objective-C, it is easy to get a false sense of security for null checking as it [isn't a language that shits the bed anytime a method is called on null](http://docs.oracle.com/javase/7/docs/api/java/lang/NullPointerException.html)

It is wishful thinking that an implementation will stay [frozen in time after a public release](http://kylerichter.com/?p=44), since so many applications rely on Web APIs that can changed or be removed. Areas of responsibility that don't handle certain failure conditions can blow up later when APIs change. //A collection will not be happy with [nil keys or values](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSMutableArray_Class/Reference/Reference.html#//apple_ref/occ/cl/NSMutableArray) and then the Application will be at the mercy of an exception handler.

Network requests can [fail and return errors](https://github.com/AFNetworking/AFNetworking/wiki/Getting-Started-with-AFNetworking#step-4-dive-in). A lazy stream of values can [send errors](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoaFramework/ReactiveCocoa/RACSignal.h), saving a file to disk [can error](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSData_Class/Reference/Reference.html#//apple_ref/occ/instm/NSData/writeToFile:options:error:) and [saving values in a database](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectContext_Class/NSManagedObjectContext.html) can fail. All of this requires some degree of defensive programming, adding to the size of a code base, at the same time as increasing its tolerance to changes in external APIs.

In Mobile we should strive for the leanest and most polished solution for the User. A lean Application is defined by its willingness to cut down on useless features and visual clutter, not minimalism in the number of lines of code. The process of writing Test Cases presents another opportunity to think about behaviours that we expect, such as operations on null objects and Errors being returned from an external service. This may help us program more defensively, if we write tests for these behaviours, crucial to the stability of our code in the future.

### Wrap-Up
Unit Testing is traditionally seen as a way to validate the correctness of an implementation and ensure that it does not regress in the future.

You may have to make changes to your Application code to make it more testable. This may appear to be backwards at first, but it is worth persevering! Altering your implementation to make it more testable has benefits beyond correctness in the present, and preventing regression in the future. I hope to show you more of these effects in Part 2.
