---
layout: post
title: "Swift Parsers - Introduction"
date: 2014-09-12 22:30:00 +0100
comments: true
published: true
categories:
- Functional
- Swift
- iOS
- Objective-C
- Parsers
- Backends
---

This the first in a series of four posts on Parsing code in Swift. This first post takes a look at the development landscape for Cocoa, Swift, Objective-C & Parsers.

### Introduction

iOS Development has been a fun ride over the last few years. Frameworks are added and improved and generally speaking follow some common idioms. UIKit is well-trodden ground with significant changes as the level of interactivity and responsiveness of what a mobile experience is moves ever forward. When new API introduced or refined, compatibility issues occur, we fix them, we add delete some code that new API subsumes, we write some new features using the new API.

The Backend of an Application is still a bit of a frontier. With no ```BackEndKit``` to provide a unified interface of a Network Connections and the Persistent State of the Application, there are no 'rails' to start on and how a Backend is built can vary enormously from Application-to-Application. A expanding world of Libraries has sprung up to simplify and consolidate the thinking of how back-end components intact in Objective-C. Much of the dynamism and flexibility of the Objective-C runtime as well as preprocessor macros is leveraged in order to kill repetitious code with metaprogramming. Now that a new Programming language has arrived and new [OS features that all but a modular design ](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/ExtensibilityPG/index.html) the time is now to look at best practice in iOS App Backends.

There is a lot of concern how Swift can (and can't be used) used with existing ways of getting-stuff-done in Objective-C. WWDC 2014 and the following months have had a ton to learn, so lets review a little of what we know about how working with Swift will effect iOS Development.

### Cocoa & Swift

Swift Interoperability with Cocoa is a mostly known quantity. Objective-C interoperability is deeply ingrained in the language, regardless of the philosophical departures that Swift makes from Objective-C. Cocoa will remain, repeated casting will be inconvenient, Strong Types with the Optional type contract will allow us to reason better about the state of the Application. Smaller components of the Application can be migrated to a Swift world in a piecemeal way without drastically changing the underlying architecture due to the extent of the [interoperability features](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/index.html#//apple_ref/doc/uid/TP40014216).

There's a lot of new language features, with a focus on moving up the levels of abstraction[^abstraction-swift-metal]. Having [Lazy Sequences as part of the Standard Library](http://swifter.natecook.com/type/LazySequence/) and setter [side-effects as part of the language](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Declarations.html) all show that things are getting a lot more modern.

The compiler is now doing a lot more with type inference can be used to increase code density without sacrificing readability or intent. Functions are types like any other value, people are rejoicing as the syntax of the language can be pared back by removing the need to worry about pointers.

### Headers & Classes

Having headers is fantastic from a documentation point of view, but the evils of duplication makes can make them unwieldy as well as informative. Swift leans on Header generation[^header-generation] and access control to automate the process of generating an Interface from the Implementation itself. Implementation details don't need to be leaked and API stubs don't need to be copy-pasted between ```.h``` and ```.m```

The implementation of both State and Behavior become considerably 'cheaper' to write, re-write and update. Classes require a fraction of the number of characters and lines-of-code to achieve the same results in terms of defining Properties and Methods to their Objective-C brethren[^documentation-loc]. By making the time taken to write and re-write a Class significantly easier, a great deal of the resistance of decomposing Classes into smaller and more testable Units is removed. The Units become smaller and easier to test, revising code becomes easier as part of a virtuous circle.

### Metaprogramming & Dynamism

Chief among the concerns with Swift is how we will cope without the dynamism that Objective-C offers. The malleability of the runtime allows for the creation and injection of functions at runtime[^swift-runtime]. Property Mapping libraries and Model Descriptions inspect classes at runtime, defining the interaction of Classes at a much higher level of abstraction. Reducing boiler plate code isn't just about convenience at the current moment in time, it reduces the entropy of the codebase over time.

However, applying these methods will just not work in Swift. Pure Swift constructs are closed at compile time and cannot be tinkered with at runtime. There is no preprocessor available for code generation before it hits the compile. Safety is king, requiring the code to be totally unambiguous about the semantics of the code if there is insufficient information for the Compiler to reason with. There is no Reflection package or framework for performing the same kind of interactions with Classes at runtime, so this seems like a huge problem.

However there are other ways of constructing and building Applications, using Abstractions that seem far away from the Gang-of-Four, Object Oriented Programming and the more dynamic idioms of Objective-C. Functional Programming provides us with abstractions that help us to dispense with the repetition of Imperative programming but without sacrificing safety and predictability. These techniques can be employed without the need to read 42 tutorials about what a Monad is and isn't. Functional Programming has a few fundamental particles, once we learn how they can be applied, the mathematics behind them become a mainly academic exercise that can be ignored if we want to. I'll cover this in the third article in this series. 

### Parser Requirements

Let's take a look at the how we can go about one component within an Application in Swift, Parsing.

I'm defining 'Parsing' to be the process of extracting data from [Data Serialization format](http://en.wikipedia.org/wiki/Comparison_of_data_serialization_formats) such as XML or JSON, to a native Model structure, valid and usable by the rest of the Application[^monadic-parser]. The Parser itself can have some or all of the following requirements:

1. Check that a Value exists for a given Key at the current level, or navigate the Data to a deeper level.
2. Distinguish between a terminal value in the Data Serialization itself versus the value not existing at all. A ```null``` value in a JSON needs to be represented in a distinct way to a value not existing at all. 
3. Coerced a value in the Serialization format into the type of the value that is expected in the Model.
4. Fail early if the the Serialization Format differs in any way from our expectations, without crashing.
5. If Parsing failure occurs, five valuable output to make it easier to isolate whether the Parsing code's assumptions about the structure of the data are wrong, or that the structure of the data itself is wrong, or both a wrong.
6. Provide defaults, or ways of deterministically deriving defaults for values that are known to be optional in the data structure.
7. The interface to the Serialization format's contents should provide a level of abstraction consistent with how it should be used.
8. Don't let any external state effect the Parsing unless absolutely necessary. Really, don't.
9. Seriously, don't even think about modifying external state. Being able to make a parser concurrent with the rest of the application makes everybody happy.
10. It should be easy to Parse serialized data from a variety of sources into the parser. The data may come bundled with the Application, from a HTTP request, or a fixture in a Unit Test.
11. Write Unit Tests for all of the boundary conditions, so we can be super-certain that our understanding of the serialization format holds true in the implementation.
12. If our Unit Tests don't cover all of the possible conditions in the structure of the format and a new condition is discovered, write a test for to ensure as much of the information about the structure of the data has been captured as possible.
13. Don't make the interface for pulling data out of the Serialization more expansive than necessary. If functionality is required that can be expressed in terms of more fundamental extraction functions, define it as a composition of the more fundamental functions.

This is not an exhaustive list of requirements and it has already exhausted you and I. Admittedly, some of these requirements are true of Software Development generally, but it is always worth thinking about.

In Objective-C, Parsing can be a minefield of typing and branching. I don't think I'd be going out on a limb to say that it is one of the most fragile parts of most codebases. Because of its fragility, if an Objective-C Application has any Unit Tests whatsoever, the Parsing components will have the best coverage. This is a big driving force behind Libraries such as the brilliant [Mantle](https://github.com/mantle/mantle)

Regardless of the awesomeness of an Objective-C library for parsing data, it won't ever be possible to inspect the expected type of the destination property in the Model at runtime and thereby avoid issues with Model Objects having values of the wrong type assigned[^typing-code-generation]. Not only that but Objective-C doesn't fail-fast if an Object assigned an Object of a class differs from the expected class that the assignee Object expects[^dynamic-typing-assignment]. This leads to explosions that occurs far away from the original erroneous assignment[^assertions-exceptions].

However, it would appear that the situation is even worse for Swift. Without much in the way of metaprogramming or some runtime magic it looks like the process of parsing will be incredibly repetitive and laborious.

### XML

XML is an interesting serialization format because it isn’t JSON. It's a format that we like to chuckle about and deride because of the complexity, but the reality is there are many APIs that Applications need to connect to that use XML. I think that many of the problems that we have with parsing XML comes from the fact that the abstractions we use to extract data from XML aren't that great. Handling all of the failure points in Parsing is a hard task and that's why we want to automate it with metaprogramming and make it bulletproof with Unit Tests.

XML is also format because there is a good amount of variety in the ways that we model an XML document and its content in API[^dom-sax]. Typically JSON is just extracted into the native Array, Dictionary, String & Numeric types of the targeted programming language. XML grew in popularity at a time when system resources like available memory were at much more of a premium than they are today. Constraints like this tend to drive innovation in a few directions, which make the process of parsing a little bit more interesting that pulling values out of a language-native Dictionary object.

If you aren't a crazy person like me you'll probably stick a Webservice in front of whatever else you need to consume and do all the complex data processing on a Server[^server-processing], then spit everything out in a RESTful JSON service that normalizes any intricacies and madness in the original data. However the reality isn't always so ideal and its up to the Client Software to process data from a variety of Sources and Serialization Formats.

### Next Time...

In the [next post](/blog/2014/09/11/swift-parsers-imperative/), I'll be a crazy person and create a simple XML interface in Swift, and extract data out of it in an Imperative style.

[^abstraction-swift-metal]: The inevitable is happening in iOS Land; Higher Levels of Abstraction as device performance increases, Frameworks and specialized languages for performance critical code. On iOS this is the availability of Accelerate and Metal. The amount of C & C++ in iOS Development may diminish, but I don't see it going away within the next decade.

[^header-generation]: This can be seen in any imported Frameworks, whether they are provided by the User, a 3rd Party, or Apple. Just CMD+Click on a third party Class in Xcode and it will take you to the Class or Method definition.

[^documentation-loc]: This means we'll all write better method documentation right?

[^swift-runtime]: A lot of the malleability of the Objective-C runtime has allowed to be built on top of it in the first place!

[^typing-code-generation]: It is possible to do this by attatching data about Classes of the properties of a Class, manually or automatically (this is part of the role of [NSManagedObjectModel](https://developer.apple.com/library/mac/documentation/Cocoa/reference/CoreDataFramework/Classes/nsManagedobjectModel_Class/Reference/Reference.html)). However, this isn't the same thing as being able to know what the Class of an ```@property``` is from the Objective-C runtime.

[^dynamic-typing-assignment]: I think my Pro Strong-Typing-bias is showing. I'm just describing one of the disadvantages of a weak type system, when there are also benefits.

[^assertions-exceptions]: The contracts of assignment can be a good deal stronger with liberal use of assertions like ```NSParameterAssert```.

[^dom-sax]: This is covered in more detail in the [fourth in this series]().

[^server-processing]: You'll also have your pick of languages on the server, so you can use a language & framework that works well in this domain. Either way the Client App will have to consume data at some point and the App should be fault-tolerant enough that it doesn't fall over if there's a hiccup on the server.
