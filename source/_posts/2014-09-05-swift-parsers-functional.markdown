---
layout: post
title: "Swift Parsers - Functional"
date: 2014-09-05 15:00:00 +0100
comments: true
categories: 
---

In the previous post we built a XML to Model parser using Imperative tecniques trying to implement some of the robustness requirements from the first post. In this post I'll cover how we can use some of the features of Swift as well as some Functional Programming to create a parser that can map XML to a Model Object.

### Helper Functions

Using what we've learned from the Imperative approach we can build a Class of Higher-Level functions for common extraction tasks. We can also build Higher-Level functions that give more information about where failure occurs. Optional Chaining is convenient, but the failure is silent and it will take some poking around to figure out where the failure actually occured in the chained Optional expression.

Higher-Level Parser functions can be build on top of ```XMLParsableType``` with a Class of Static functions[^xmlparser-implementation]:

	public final class XMLParser {
		// Takes an ```XMLParsableType``` and a String and returns an Array of Children with the given Element Name.
		public class func parseChildren<X: XMLParsableType>(childTag: String)(xml: X) -> Result<[X]>   
		// Takes an ```XMLParsableType``` and a String and returns the first child with the given Element Name.
		public class func parseChild<X: XMLParsableType>(childTag: String)(xml: X) -> Result<X> 
		// Takes an ```XMLParsableType``` and an array of Strings and returns the first child by recursively applying parseChild.
		public class func parseChild<X: XMLParsableType>(childTags: [String])(xml: X) -> Result<X>
		// Takes an ```XMLParsableType``` and a String and returns the Text of the Child of the given Element Name.
		public class func parseChildText<X: XMLParsableType>(childTag: String)(xml: X) -> Result<String>
		public class func parseChildText<X: XMLParsableType>(childTags: [String])(xml: X) -> Result<String>
	}

Even if the comments were not present, looking at the types of the arguments and the return value, it should be possible to see what these functions accomplish. There are some important properties of these functions to consider:

1. They consume a Protocol, abstracting away how the XML Parser is implemented. As the Protocol has an associated type requirement it has to be fulfilled with Generics. Composed behaviours don't pollute the Implementation of the Protocol but still augment the behaviour.
2. The functions are [Curried](http://en.wikipedia.org/wiki/Currying). A Curried function allows us to apply the function's arguments at different times and Swift [provides a convenient syntax](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Declarations.html). Each time an argument is applied, a new function is returned that accepts the next argument, or the result of all of the arguments is returned. By doing this we can derive functions that are chainable as they accept the Parsable at a later time.
3. The functions are [Pure](http://en.wikipedia.org/wiki/Pure_function)[^class-methods-free-functions], they do not consume or modify any Global State, only the arguments are used. The interface can't guarantee that side-effects won't occur but this is something implementors should aim towards.
4. Every function can Fail. The as an XML node may-or-may-not have text and a child of a particular name, failure can occur at any stage. If these functions were used in a correct imperative Parser, the failure case would be handled with conditionals and early return. This can be seen in other API, with the presence of an ```Optional``` return type. 
5. Input values must exist. Swift makes strong guarantees about the existance of values, there's no need to check for the existance of reference types. In an Objective-C implementation this contract can be enforced with a litany of assertions. If the language and compiler can enforce these guarantees we've just had a huge productivity win.
6. They are relivately minimal and do not include interpretation of every possible Type. Numeric and complex types within an XML document are represented in textual form. Unlike JSON, there is no explicit syntax for a Numeric value. Instead of bloating this Parser class with every variation of interpreting Text as a Numeric value, the parsing of ```Double```, ```Int```, ```Bool``` and other types can be composed from other functions. This is an important point to understand about Functional Programming.

### Applying Functional Principles

I'm using the fantastic and lightweight[^swiftz-lightweight] [Swiftz](https://github.com/maxpow4h/swiftz/) for go-to implementations of many of the Functional primitives. It also has the ```Result``` type the ```XMLParser``` Class uses. 

One important conceptual change to grasp is that functions can be assigned to variables and constants just like values can! Objective-C has had blocks for some time now, assigning a block to a property of a Class is second nature to many Objective-C developers. Swift takes this a step further, a [Closure in Swift](https://developer.apple.com/library/prerelease/mac/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html) is just a special kind of function! Blocks are no longer the best way of passing around a computation, we can consider the equivalence of functions declared with ```func``` in a Global or Class scope with local Closures:

	let intToNumber:  Int -> NSNumber = NSNumber.numberWithInt
	let intToNumber1: Int -> NSNumber = { NSNumber.numberWithInt($0) }

#### Cascading Failure

We've allready seen that we can use Swift's Optional Chaining in our parser to limit the number of occurences of handling failure. However, the ```Result``` isn't a blessed type in the language with special language syntax for chaining. However, we want idenical behaviour with the ```Result```.

Swiftz has the [following function declared](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Optional.swift#L42)[^swiftz-generics-simplification]:

	public func >>-<A, B>(a: Result<A>, f: A -> Result<B>) -> Result<B>

It is called Bind and it can be thought of in the following terms:

> _"If the parameter *a* has a Value apply the function *f* and return it in a *Result*. If the parameter *a* is an Error, just return the original *Result*"_

This is essentially same as Optional Chaining! In fact Optional Chaining [can be defined in this way](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Optional.swift#L42). Lets look at this in practice extracting the ```name``` value of an ```Animal```:

	let name = XMLParser.parseChild("nested_nonsense")(xml: xml) >>- XMLParser.parseChildText("name") // A Result<String>

The type of name is ```Result<String>```, just what we want. The function can be broken down, as it becomes easier to see that there is something else going on[^breaking-down-bind]:

	let parseName = XMLParser.parseChildText("name") // This is a function, the type is X -> Result<String>
	let nestedNonsense = XMLParser.parseChild("nested_nonsense")(xml: xml)	// This is a Result<X>
	let name = nestedNonsense >>- parseName // The Bind operator joins parseName and nestedNonsense together.

The Bind operator requires that the Right-Hand argument is a function matching up with the Generic type on the Left-Hand argument. The ```parseName``` 

#### Applicatives

Lets assume that we have the correct code that produces a ```Result``` container for each of values that needs to be extracted from the XML and assigned to the Model. The computation that needs to be performed is _"If all of the Results corresponding to each of the Values are Successful, return our Model with the values applied inside a Result context, otherwise return the first Error Result."

This computation can be performed by Currying the Constructor for the Model and then applying each Result in order. A Curried 'Constructor'[^constructor-factory-method] for the ```Animal``` Model looks like this:

    static func build(type: String)(name: String)(url: NSURL) -> Animal {
      return self(type: type, name: name, url: url)
    }

Given that we have the previous ```Result```s corresponding to each of the parameters the computation can be broken down:

    let type: Result<String> = ...
    let name: Result<String> = ...
    let url: Result<NSURL> = ...
    
	let build: String -> String -> NSURL -> Animal = self.build
    let first: Result<String -> NSURL -> Animal> = self.build <^> type
    let second: Result<NSURL -> Animal> = first <*> name
    let third: Result<Animal> = second <*> url

It should be a little clearer what is going on given the additional annotations for each value, however the compiler would automatically infer them. There are some infix functions doing the heavy lifting of pulling values out of the ```Result``` context and sticking them back in again.

Firstly, the ```fmap``` operator:

	public func <^><A, B>(f: A -> B, a: Result<A>) -> Result<B>

This function is used to apply a function to a the value contained in the ```Result``` context, without the function applied to have to be aware of the context itself!

Secondly the ```apply``` operator:

	public func <*><A, B>(f: Result<A -> B>, a: Result<A>) -> Result<B>

Very similar to ```fmap``` except the function can be in a ```Result``` context too!

#### Function Composition	

So far, the Parsing code we've covered has only been concerned with String values. This is of course very simple as our XML interface exposes Strings. However, our Model is concerned with other types; ```Bool```, ```Double```, ```Int```, ```CLLocationCoordinate2D```. Additionally, the Models form a heirarchy, with a number of ```Animal``` structures belonging to a ```Zoo```. It can't be assumed that all of the types are Strings.

Coercing a value from a String to a Numeric type may not always work. The String '14123' can be interpreted as an ```Int``` but the value ```134djk23``` cannot. ```NSNumberFormatter``` is a Cocoa way of interpreting Strings as numbers, we can write an extension for ```Int``` to intepret a ```String``` as an ```Int```, with ```Optional.None``` used to representing failure:

	public extension Int {
	  public static func parseString(string: String) -> Int? {
	    let formatter = NSNumberFormatter()
	    let number = formatter.numberFromString(string)
	    return number?.integerValue
	  }
	}

Cocoa APIs in the Swift world mostly convey failure with a ```nil``` return value in an Optional type[^nserror-failure]. Our Parser requires the failure context provided by an Error, but. One approach is to add a second ```parseString``` method that returns  as Extensions to all of the types that we want to be able to transform between. This has two disadvantages:

1. Cocoa Classes are polluted with ```Result``` variants of every possible method that could fail, and this is expanded as more of these methods are needed
2. Some of the Context surrounding the failure is lost. We would like our ```NSError```s to contain the element within the XML responsible for the failure, an Extension method would lose this context. _'Failed to interpret 'toilet' as an Int'_ is preferable to _'Failed to interpret '123a' as an Int'_

Instead of writing a general function that returns a ```Result<Int>``` we can create a function local to the ```decode``` function which captures context about the failure using a closure:

	let toiletCountBoolParser: String -> Result<Bool> = { promoteDecodeError("Could not parse 'disabled_parking")(value: Bool.parseString($0)) }

Composing two functions where the Output of the first becomes the input of the second is such a common pattern it is generalized in Functional Programming with the 'Compose' operator:

	public func •<A, B, C>(f: B -> C, g: A -> B) -> A -> C

The previous ```toiletCountBoolParser``` can be re-written without a closure, and integrated into the parsing of the ```toiletCount``` property using the previous ```>>-``` bind operator::

	let toiletCountBoolParserA: String -> Result<Bool> = promoteDecodeError("Could not parse 'disabled_parking") • Bool.parseString
	let toiletCount =  XMLParser.parseChildText(["facilities", "toilet"])(xml: xml) >>- toiletCountBoolParser


#### The Benefits

There's a lot to take in here, some of the benefits should be clear, others are a bit more subtle.

All of this code features absolutely no branching with conditional statements. The branching and conditionals all occur inside the Higher-Order functions. These patterns are applicable outside of Parsing code, the infix functions are general enough that they can be applied anywhere that ```Optional``` and ```Result``` containers are[^containers-functors]

In the same way we can compose data by applying sequences of operations, we can compose *functions* themselves. This is a massive boon for code re-use and removes a great deal of the Ambient State needed to move data between functions. The type signatures of functions are a massive giveaway to what they do. 

We have computational pattern identical to Optional Chaining with the added information that a ```Result``` provides us with in the case of an Error. These errors are trivially parameterized removing all of the boiler plate associated with constructing an ```NSError``` in the Imperative Implementation[^macro-error-handling].

### Teach the Controversy

I don't deny some of this doesn't have a cost in terms of a leaning curve, it can look very alien at first. However, this isn't less true of any Domain Specific Language that we can graft on top of Objective-C or with a [more flexible language](http://robots.thoughtbot.com/writing-a-domain-specific-language-in-ruby). There are a whole host of formal and informal Objective-C libaries have many ways of expressing[^objc-dsls]. It just so happens that these Higher-Level  operators are applicable outside the domain of parsing and exist in concept and practice across other programming languages.

However, the differences are mainly conceptual. There are plenty of other fantastic posts that will do a better job of explaining Bind, FMap, Apply but I hope the rationale for using these functions is reasonably clear.

We have had a similar behaviour in Objective-C since its inception; [Sending Messages to Nil](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/ProgrammingWithObjectiveC/WorkingwithObjects/WorkingwithObjects.html).  This behaviour is just as alien to newcomers to Objective-C as some of these functional concepts are! 

THIS IS BETTER THAN METAPROGRAMMING, THE COMPILER TELLS US WHAT IS WRONG.

Sure, we'll see a lot code that goes crazy with Infix operators and devolves into an incomprehensible meta-language that is only understandable by the creator. However, there is a great deal of merit in the targeted and limited
Macros are DSLs allready
Monads are established
Get to the best level of abstraction earlier

[^xmlparser-implementation]: The Implementation for this class is in the [next post in this series]().

[^swiftz-generics-simplification]: I'm lying, I've changed the Generic Parameters from ```VA``` & ```VB``` to ```A``` & ```B```

[^hypothetical-xml]: These assumptions actually hold true for a [webservice to be consumed](http://www.livedepartureboards.co.uk/ldbws/) in an Application I was prototyping. Depending on the Webservice an Application is consuming, there's a great deal of assumptions that can be made to reduce the complexity of an Implementation.

[^lazy-evaluated-functional-programming]: One of the interesting aspects of Haskell is [Lazy Evaluation](http://en.wikipedia.org/wiki/Lazy_evaluation), in particular how it applies to building up a [data structure and then traversing it](http://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf).

[^breaking-down-bind]: When starting out it can be really helpful to do this, it makes inspecting the types of each of the elements in the chain more visible. You can use Alt+Click on the value name to get XCode to print out the inferred type. Its also a good illustration of the power of type inference.

[^model-mapping-traditional]: I certainly remember writing this kind of thing by hand, before [Java annotations could be used for Code Generation](http://docs.oracle.com/javase/tutorial/jaxb/intro/).

[^monadic-parser]: The concept of a 'Parser' as a distinct type that implements the 'Monad' Typeclass [does indeed exist in Pure functional language](http://eprints.nottingham.ac.uk/223/1/pearl.pdf).

[^constructor-factory-method]: 'Factory Method' is probably a better term for this since a Constructor is a concept specific to ```init``` methods in Swift. 

[^nested-branching]: It would look [something like this](https://gist.github.com/lawrencelomax/00ea2c00c9b6ca5bb4ab)

[^containers-functors]: ```Optional``` and ```Result``` are examples of Functors and Monads. This [fantastic article](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) covers the operators in a visual way.

[^macro-error-handling]: In Objective-C this can be handled with [Macros and Early Return](https://github.com/rentzsch/NSXReturnThrowError), but we can't rewrite/mangle the rules of the language in Swift as we don't have Macros.

[^decoder-protocols]: This is heavily inspired by the ThoughtBot article on JSON Parsing in Swift. In my Project I have multiple decoder types for JSON, XML and CSV.

[^swiftz-lightweight]: Swiftz has a Core library as well as a more full-featured Framework. There's a lot of interesting stuffin the Full library, but just getting started with the Core is a good place to start.
