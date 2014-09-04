---
layout: post
title: "Swift Backends - Interfaces"
date: 2014-09-03 17:46:45 +0100
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

## Applying to Parsing

It looks like ['How to Parse JSON in Swift'](http://robnapier.net/functional-wish-fulfillment) as become the ['Haskell Monad Tutorial'](http://www.haskell.org/haskellwiki/Monad_tutorials_timeline). However, it is a great place to get started with some new ways of thinking as many of us understand how the Static-but-inferred typing of Swift changes how we can think about programming problems.

In Objective-C, Parsing can be a minefield of typing and branching. I don't think I'd be going out on a limb to say that it is one of the most fragile parts of most codebases. If an Objective-C Application has any Unit Tests whatsoever, the Parsing components will have the best coverage.

I'm defining 'Parsing' to be the process of extracting data from [data serialisation format](http://en.wikipedia.org/wiki/Comparison_of_data_serialization_formats) to a Model construct that is usable by the rest of the Application. The Parser itself can have some or all of the following requirements:

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

### XML

XML is an interesting serialisation format because it isn’t JSON. It is a format that we all like to chuckle about and mock, but the reality is there are many APIs that still use it for a whole host of reasons. Many of the problems with XML comes from the interfaces and abstractions that we use to extract data out of it, rather than with XML itself. XML is also interesting because over time and for a number of historical[^dom-sax] reasons there are Event and Data based interfaces for extracting data. This has lead to a number of libraries and frameworks from a variety of vendors[^xml-libraries].

If you aren't a crazy person like me you'll probably stick a Webservice in front of whatever else you need to consume and do all the complex data processing on a Server, then spit everything out in a RESTful JSON service that normalizes any intricacies in the original data. In reality this isn't always possible and the Client needs to process data from a variety of sources and Serialization Formats.

### Let's Stub an Interface!

Stubbing a Protocol or Interface is a great way of getting to grips with the problem that you want to Solve. It also helps you to crystallise what is important, what details we can ignore to solve the problem that we want.

In parsing some [purely hypothetical XML](http://www.livedepartureboards.co.uk/ldbws/). I've made a few assumptions:

1. I care about the data contained in Text Nodes.
3. I need to be able to recursively address Elements within a Tree-like structure.
4. I need to be able to enumerate Elements of the same name at the same level in the tree.
5. I don't care about anything else (namespaces, attributes, schemas).

Here's a protocol for fetching data from a Parsable XML Node:

    public protocol XMLParsableType {
      func parseChildren(childTag: String) -> [Self]
      func parseText() -> String?
    }

*"That's It?"*. Yep. Everything else can be composed on top of this minimal protocol. Higher-level interaction can be build on top of these fundementals. We can even explain what an XML Parsable is in one sentence:

> _"An XML Parsable has an ordered collection of Child Parsables and may have associated text"_

Protocols are permitted to have a recusive definition using the `Self` placeholder type. How and where the fetched data is stored is left to the implementing class/struct/enum. Obtaining a child may be implemented by traversing a fully reified data structure, or moving a cursor along a buffer.

### Helper functions

An XML Parsable has an array of children (that can be empty) For example we can define a function that obtains the first child of a given tag:

	public class func parseChild<X: XMLParsableType>(childTags: [String])(xml: X) -> X? 

A function that obtains the first child recursively:

	public class func parseChildren<X: XMLParsableType>(childTags: [String])(xml: X) -> X?

A function that obtains the Text element of a child:

	public class func parseChildText<X: XMLParsableType>(childTag: String)(xml: X) -> String?

The declaration of these functions have some important properties:

1. They consume a Protocol, abstracting away how the XML Parser is implemented. As the Protocol has an associated type requirement it has to be fulfilled with Generics. Composed behaviours don't pollute the Implementation of the Protocol but still augment the behaviour.
2. The functions are Curried. A Curried function allows us to apply the function's arguments at different times. Each time an argument is applied, a new function is returned that accepts the next argument, or the result of all of the arguments is returned. By doing this we can derive functions that are chainable as they accept the Parsable at a later time.
3. The functions are pure[^class-methods-free-functions], there is no State consumed or modified, only the parameter are used in the function. The interface can't guarantee that side-effects won't occur but this is something implementors should aim towards.
4. Every function can fail. The as an XML node may-or-may-not have text and a child of a particular name, failure can occur at any stage. If these functions were used in a correct imperative Parser, the failure case would be handled with conditionals and early return. 
5. Input values must exist. Swift makes strong guarantees about the existance of values, there's no need to check for the existance of reference types. In an Objective-C implementation this contract can be enforced with a litany of assertions. If the language and compiler can enforce these guarantees we've just had a huge productivity win.

In the next post, I'll talk about how these functions can be applied.

[^header-generation]: This can be seen in any imported Frameworks, whether they are provided by the User, a 3rd Party, or Apple. Just CMD+Click on a third party Class in XCode and it will take you to the Class or Method definition.
[^dom-sax]: I cut my programming teeth on Java. When consuming an XML Document of greater than a MB or so the JVM Heap could get hammered when using a DOM style parser, as all of the elements in the document were read into memory. This could be hugely problematic when intertwined with a Garbage Collector. For this reason the Event-Based Streaming SAX parser could be used instead, the whole document need not be read into memory at the expense of a more troublesome interface. Eventually a compromise was found with the StAX parser, buffering the input document with a Cursorable navigation mechanism.
[^xml-libraries]:  On iOS alone we is [libxml](http://xmlsoft.org) and [NSXMLParser](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/nsxmlparser_class/reference/reference.html). libxml also [Event](http://xmlsoft.org/html/libxml-SAX2.html) and [Tree](http://xmlsoft.org/html/libxml-tree.html) based interfaces.
[^objc-dsls]: Macros exist all over Objective-C projects.