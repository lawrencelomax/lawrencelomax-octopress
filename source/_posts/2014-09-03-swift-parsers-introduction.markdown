---
layout: post
title: "Swift Parsers - Introduction"
date: 2014-09-03 15:00:00 +0100
comments: true
categories: 
---

There are certain components within the composition of an Application that are well-trodden ground. UIKit is gradually improved year-on-year with substantial but not 'stop-the-world' panic. We get new API introduced or refined, compatibility issues occur, we fix them, we add delete some code that new API subsumes, we write some new features using the new API.

How the backend of an Application is constructed with the interaction of persistence, networking and model-mapping is not handled inside any of the iOS Frameworks, there is no ```BackEndKit``` for providing a unified interface of a Network Connection and the state of Core Data. A cottage industry of libraries and helpers has sprung up to simplify and consolidate the thinking of how back-end components intact in Objective-C. Much of the dynamism and flexibility of the Objective-C runtime as well as preprocessor macros is leveraged in order to kill repetitious code with metaprogramming.

## Swift

We've heard a lot of noise about what Swift is and isn't over the last few months, lets look at a few realities:

### Interoperability with Cocoa

Swift Interoperability with Cocoa is a mostly known quantity. Objective-C interoperability is deeply ingrained in the language, regardless of the philosophical differences of Swift. Cocoa will remain, repeated casting will be inconvenient, stronger types and the contract of Optionals will allow us to reason better about the state of the Application at a given point in time.

### Syntax and Language Features

New syntax and language features, type inference can be used to increase code density without sacrificing readability or intent. Smaller components of the Application can be migrated to a Swift world in a piecemeal way without drastically changing the underlying architecture.

### Headers

Having headers is fantastic from a documentation point of view, but the evils of duplication makes them as unwieldy as they are informative. Swift leans on Header generation[^header-generation] and access control to automate the process of generating an interface from the Implementation itself. Implementation details don't need to be leaked, API stubs don't need to be copy-pasted between ```.h``` and ```.m```

The implementation of both State and Behaviour considerably 'cheaper' to write, re-write and update. Structs & Classes require a fraction of the number of characters to achieve the same results. This removes much of the resistance of decomposing Classes into smaller and more testable Units. The Units become smaller and easier to test, revising code becomes easier as part of a virtuous circle.

### Metaprogramming & Dynamism

Chief among the concerns with Swift is how we will cope without the dynamism that Objective-C offers. The malleability of the runtime allows for the creation and injection of functions at runtime. Property Mapping libraries and Model Descriptions can inspect classes at runtime, defining the interaction of Classes at a much higher level. Reducing boiler plate code isn't just about convenience at the current moment in time, it reduces the entropy of the codebase over time. This isn't without a cost........

Applying these methods will just not work in Swift. Pure Swift constructs are closed at compile time and cannot be tinkered with at runtime. There is no preprocessor available for generating large amounts of code fed into the compiler. Safety is king, requiring the code to be totally unambiguous about the semantics of the code if there is insufficient information for the Compiler to reason with.

However there are other ways of constructing and building Applications, using Abstractions that seem far away from the Gang-of-Four, Object Oriented Programming and the more dynamic idioms of Objective-C. Functional Programming provides us with abstractions that help us to dispense with both the repetition but without sacrificing safety and predictability. These techniques can be employed without the need to read 42 tutorials about what a Monad is and isn't. Functional Programming has a few fundamental particles, once we learn how they can be applied, the mathematics behind them become a mainly academic exercise that can be ignored if we want to.

## Parser Requirements

In Objective-C, Parsing can be a minefield of typing and branching. I don't think I'd be going out on a limb to say that it is one of the most fragile parts of most codebases. If an Objective-C Application has any Unit Tests whatsoever, the Parsing components will have the best coverage.

I'm defining 'Parsing' to be the process of extracting data from [data serialisation format](http://en.wikipedia.org/wiki/Comparison_of_data_serialization_formats) to a Model construct that is usable by the rest of the Application[^monadic-parser]. The Parser itself can have some or all of the following requirements:

1. Check that a value exists for a given key or node in a tree-like structure exists.
2. Distinguish between a terminal value in the Data Serialization itself versus the value not existing at all. For example a ```null``` value in a JSON Object rather than a value missing from the JSON itself.
3. Check that the value for a given key in the Serialized format can be coerced or interpreted as the type of the value that we expect in the Model.
4. Fail early if the Parser knows that there is incoming data that differs from our expectations of the data.
5. Give valuable output for why the parsing has failed so that we can check whether the code's assumptions about the data are wrong, or that the data itself is wrong.
6. Provide defaults for values that we know are valid when Optional in the data, but can be created with a default value in the Model.
7. The interface to the Serialization format's contents should be abstracted to a point where the Serialization format doesn't unnecessarily influence the implementation of the Parser.
8. Don't make too many assumptions about a fully realised object graph. The quantity of the data in the Serialization my be too great to keep in memory without degrading performance and battery life.
9. Don't let any external state effect the Parsing. Really, don't.
10. Seriously, no external state. We might want the parsing to occur asynchronously in a trivial manner. 
11. Write Unit Tests for all of the boundary conditions, so we can be super-certain that our understanding of the serialisation format holds true in the implementation.
12. Provide a minimal interface for pulling data out of the Serialization, avoiding duplication. Any conveniences for common compound data fetches can be composed on top of the minimal interface.

This is not an exhastive list of requirements and it has already exhausted you and I. 

There's a lot going on and this would require a substantial effort in an Imperative style using Objective-C. Libraries and Frameworks can help a lot, but they can only go so far. 

For example it isn't possible to inspect the expected type of the destination property in the Model. This is a common landmine, an Objective-C class isn't checked that it is the appropriate class when it is cast so failure will occur away from the source of the error[^assertions-exceptions].

### XML

XML is an interesting serialisation format because it isn’t JSON. It is a format that we all like to chuckle about and mock, but the reality is there are many APIs that still use it for a whole host of reasons. Many of the problems with XML comes from the interfaces and abstractions that we use to extract data out of it, rather than with XML itself. XML is also interesting because over time and for a number of historical[^dom-sax] reasons there are Event and Data based interfaces for extracting data. This has lead to a number of libraries and frameworks from a variety of vendors[^xml-libraries].

If you aren't a crazy person like me you'll probably stick a Webservice in front of whatever else you need to consume and do all the complex data processing on a Server, then spit everything out in a RESTful JSON service that normalizes any intricacies in the original data. In reality this isn't always possible and the Client needs to process data from a variety of sources and Serialization Formats.

### Next Time...

In the next post, I'll be a crazy person and create a simple XML interface in Swift, and extract data out of it in an Imperative style.

[^header-generation]: This can be seen in any imported Frameworks, whether they are provided by the User, a 3rd Party, or Apple. Just CMD+Click on a third party Class in XCode and it will take you to the Class or Method definition.
[^dom-sax]: I cut my programming teeth on Java. When consuming an XML Document of greater than a MB or so the JVM Heap could get hammered when using a DOM style parser, as all of the elements in the document were read into memory. This could be hugely problematic when intertwined with a Garbage Collector. For this reason the Event-Based Streaming SAX parser could be used instead, the whole document need not be read into memory at the expense of a more troublesome interface. Eventually a compromise was found with the StAX parser, buffering the input document with a Cursorable navigation mechanism.
[^xml-libraries]:  On iOS alone we is [libxml](http://xmlsoft.org) and [NSXMLParser](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/nsxmlparser_class/reference/reference.html). libxml also [Event](http://xmlsoft.org/html/libxml-SAX2.html) and [Tree](http://xmlsoft.org/html/libxml-tree.html) based interfaces.
[^objc-dsls]: Macros exist all over Objective-C projects.