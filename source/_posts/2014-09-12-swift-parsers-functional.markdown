---
layout: post
title: "Swift Parsers - Functional"
date: 2014-09-12 22:30:02 +0100
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

In the [previous post](/blog/2014/09/12/swift-parsers-imperative/) we built a ```decode``` function to parse data out from XML and into an ```Animal``` Model using Imperative techniques. This required some efforts in order to satisfy some of the robustness requirements from the [first post](/blog/2014/09/12/swift-parsers-introduction/).

In this post we'll cover how we can use Functional Programming techniques on top of the language features of Swift to decode XML to an ```Animal``` model. This post assumes that you are comfortable with the [Parameterized Types and Generics](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-XID_278). Sample Code for is [available on GitHub](http://github.com/lawrencelomax/XMLParsable).

### Thinking Functionally

I'm using the fantastic and lightweight [Swiftz](https://github.com/maxpow4h/swiftz/) for go-to implementations[^swiftz-lightweight] of many [Higher-Order Functions](http://en.wikipedia.org/wiki/Higher-order_function). It also has a great implementation of the [```Result``` type from the previous post](/blog/2014/09/12/swift-parsers-imperative/).

One important concept in Functional Programming is that functions can be thought of as [First-Class types](http://en.wikipedia.org/wiki/First-class_function) just like any other variable or constant in code. Objective-C has had blocks for some time now, allowing us to think in terms of passing functions around; assigning a block to a property of a Object. However this isn't the same thing as First-Class functions. Swift allows Functions to be First-Class whilst providing equivalence in functions no matter how they are declared, a [Closure in Swift](https://developer.apple.com/library/prerelease/mac/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html) is just another kind of function [just like a method](http://oleb.net/blog/2014/07/swift-instance-methods-curried-functions/) or [Free Function](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Functions.html#//apple_ref/doc/uid/TP40014097-CH10-XID_243). Blocks are no longer the best way of passing around a computation, we can consider the equivalence of functions declared with ```func``` in a Global or Class scope with local Closures:

	let intToNumber:  Int -> NSNumber = NSNumber.numberWithInt
	let intToNumber1: Int -> NSNumber = { NSNumber.numberWithInt($0) }

By now it should be second nature to think of the concatenation of two Strings using the [+ operator](http://swifter.natecook.com/operator/pls/). As functions are types like any other, two functions can be joined together just like a String.

	public func •<A, B, C>(f: B -> C, g: A -> B) -> A -> C	// The 'compose' operator from Swiftz
	
	func prefixer(prefix: String)(string: String) -> String { // Equivalent to String -> String -> String
		return prefix + string
	}
	func postfixer(postfix: String)(bar: String) -> String { // Equivalent to String -> String -> String
		return string + postfix
	}
	
	let happyPrefix = prefixer("😃") // String -> String. The first String argument of 'prefixer' is applied.
	let sadPostfix = postfixer("😞") // String -> String. The first String argument of 'postfixer' is applied.
	let infixer = happyPrefix • sadPostfix // String -> String. Two functions are joined, the output of the first becomes the input of the second.
	let string = infixer("Good Morning") // "😃 Good Morning 😞"

In this example, the Swiftz [compose operator (•)](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Functions.swift#L59) is used in conjunction with [Curried Functions](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Declarations.html). This might look a little crazy for now, but two important concepts are:

1. A specialized function can be created from a curried function, by applying some, but not all of the arguments.
2. New functions can be created locally by composing other functions with fancy operators.

### Building new Parsers

With the idea of composing functions [without having to declare one](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Declarations.html#//apple_ref/doc/uid/TP40014097-CH34-XID_598) can be carried over to our problem of decoding XML to a Model;  A specialized Parser function can be made for each of the values that need to be parsed out in the Model. In order to construct an [```Animal``` Model](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsableTests/Fixtures/Zoo.swift#L13) three parsers are required, with the values placed into the context of a ```Result``` for failure information:

	let kindParser: XMLParsableType -> Result<String> = ...
	let nameParser: XMLParsableType -> Result<String> = ...
	let urlParser: XMLParsableType -> Result<String> = ...

Further, we can think of each of the properties of a decoded ```Animal``` as the application of the above functions to the source ```XMLParsableType``` from the XML Document.

	let kind: Result<String> = kindParser(xml)
	let name: Result<String> = nameParser(xml)
	let url: Result<String> urlParser(xml)

We've allready seen that we can use Swift's Optional Chaining in our parser to limit the number of occurences of handling failure. However, ```Result``` isn't a blessed by the language with [special syntax for chaining](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/OptionalChaining.html#//apple_ref/doc/uid/TP40014097-CH21-XID_356). It would be great to get the same behaviour for the Result type.

Turns out that Swiftz has the [following function declared](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Optional.swift#L42)[^swiftz-generics-simplification]:

	public func >>-<A, B>(a: Result<A>, f: A -> Result<B>) -> Result<B>

It is called ['Bind'](http://bit.ly/1weNYBM) and it can be described in the following way:

> _"If the parameter *a* is a Value case of the result apply the function *f* returning the *Result* of *f*. If the parameter *a* is an Error, just return the original *Result*"_

Sounds the same as Optional Chaining! In fact Optional Chaining [can be defined in this way](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Optional.swift#L42). It should be possible to construct a ```Result<String>``` for the  ```kind``` value of an ```Animal``` this way:

	let kind: Result<String> = xml.parseChildren("kind").first >>- { $0.parseText() } // Compiler Error!

Damn, looks like the types don't match up since the ```kind``` value should be a ```Result<String>``` instead of a ```String?```. Besides, this looks much worse with needing to use an inline closure to invert the ```parseText``` 0-arg instance method on ```xml``` into a function that takes an ```xml``` parameter and executes ```parseText```. There has to be a better way of doing this...

### XMLParser: A Helper

A new approach is to create a few Helper Functions that build on top of the basic functionality of ```XMLParsableType```[^associated-types] to extract a Single child of a given Element name, providing the value in the context of a ```Result``` type, and accept the ```XMLParsableType``` as a parameter of function that isn't bound to an instance:

	public final class XMLParser {
		public class func parseChild<X: XMLParsableType>(elementName: String)(xml: X) -> Result<X> 
		public class func parseText<X: XMLParsableType>(xml: X) -> Result<String>
	}

This can then be used to extract out the ```kind``` value with our understanding of the ```Bind```[^breaking-down-bind] operator:

	let kind: Result<String> = XMLParser.parseChild("kind")(xml: xml) >>- XMLParser.parseText

Now we're talking! The ```XMLParser.parseText``` function is not invoked directly and is placed into a chain of operations to extract out text of an XML Element. These functions are being joined as if it were any other type of data.

The  ```name``` property of the ```Animal``` model is nested a few XML Elements deep, but it can be extracted by chaining the Parsing functions in the same way:

	let name: Result<String> = XMLParser.parseChild("nested_nonsense")(xml: xml) >>- XMLParser.parseChild("name") >>- XMLParser.parseText 

The ```XMLParser.parseChild("name")``` expression evaluates to a function of ```XMLParsableType -> Result<XMLParsableType>``` instead of just a ```Result``` value. We're seeing that currying is being used to create a specialized Parser function for a specific XML element, chained together in a sequence of functions.

This is the heart of Function Composition, with ```Bind``` performing the behaviour of continuing on success, bailing out as soon as an Error occurs. The possibility of failure within any of the sequences of operations is an essential property of this Parser. Previously this was handled by Optional Chaining and ```if``` statements in the [Imperative version](/blog/2014/09/12/swift-parsers-imperative/). Using ```>>-``` there isn't a branching statement in sight[^non-optional-guarantees].

### Common Operations

There appeares to be some common chained functions appearing. These can be extracted out and into the ```XMLParser``` class[^xmlparser-class] and built using the same Higher-Order Functions and other functions and methods that have been defined for ```XMLParserType```

	extension XMLParser {
		public class func parseChildText<X: XMLParsableType>(elementName: String)(xml: X) -> Result<String> {
			let textParser: X -> Result<String> = promoteXmlError("Could not parse text for child \(elementName)") • { $0.parseText()}
			return self.parseChild(elementName)(xml: xml) >>- textParser
		}
		
		public class func parseChildRecursive<X: XMLParsableType>(elementNames: [String])(xml: X) -> Result<X> {
			return elementNames.reduce(Result.value(xml)) { (parsable: Result<X>, currentElementName) in
				return parsable >>- self.parseChild(currentElementName)
			}
		}
		
		public class func parseChildRecusiveText<X: XMLParsableType>(elementNames: [String])(xml: X) -> Result<String> {
			let textParser: X -> Result<String> = promoteXmlError("Could not parse text for child \(elementNames.last)") • { $0.parseText()}
			return self.parseChildRecursive(elementNames)(xml: xml) >>- textParser
		}
	}

[```reduce```](http://swifter.natecook.com/func/reduce/) is another Higher-Order function of the [```Sequence``` type](http://swifter.natecook.com/protocol/SequenceType/) making an appearance. Using it means that the ```parseChildRecursive``` [doesn't need to be implemented in a recursive manner](http://bit.ly/1qZxDCb).

These functions can now be used with in the Animal ```decode``` function to make extracting data from the XML more obvious:

	let kind: Result<String> = XMLParser.parseChildText("kind")(xml: xml)
	let name: Result<String> = XMLParser.parseChildTextRecursive(["nested_nonsense", "name"])

Lovely.

### Parsers for Other Types	

We've seen that the definitions inside the ```XMLParser``` Helper don't provide methods for interpreting the ```String``` of a XML Text element as every possible Type as Numeric and Complex types within an XML document are represented in textual form. Our Model classes are concerned with Numeric types such as ```Double``` and ```Int```, so the parsing of these types will need to be incorporated into the ```decode``` method.

Unlike JSON, there is no [explicit syntax for a Numeric value](http://en.wikipedia.org/wiki/JSON#Data_types.2C_syntax_and_example), so the interpretation of these types isn't a requirement of an XML Parser. Instead of bloating the ```XMLParser``` class with every variation of interpreting Text as a other Types, the Parser functions for values in the Model can be composed with ```XMLParser``` functions and other functions that interpret ```String```s as the other types.

Coercing a value from a String to a Numeric type may not always work. The String '14123' can be interpreted as an ```Int``` but the value ```134djk23``` cannot. Again, this falls into our notion of Model decoding failure. The ```NSNumberFormatter``` class is a Cocoa way of interpreting Strings as Numbers, we can write an extension for the  ```Int``` type to intepret a ```String``` as an ```Int```, with ```Optional.None``` used to representing failure.

	public extension Int {
		public static func parseString(string: String) -> Int? {
			let formatter = NSNumberFormatter()
			let number = formatter.numberFromString(string)
			return number?.integerValue
		}
	}

As previously mentioned, Cocoa APIs in Swift expose failure as the ```nil```/```.None``` case of an Optional Type[^nserror-failure]. However, our Parser requires the additional information in a ```Result``` type. One approach is to extend Cocoa classes with additional methods that return a ```Result``` instead of an ```Optional```, but this might not be the ideal solution[^parsing-bloating-categories]. Instead we can again think in terms of Function Composition to make a function that returns ```Result<Int>``` instead of ```Int?```

For example, a definition of ```toiletCount``` requires interpreting a ```String``` in the XML as an ```Int``` in the model. This function can be built from closures:

	let toiletCountParser: String -> Result<Int> = { promoteDecodeError("Could not parse 'disabled_parking")(value: Int.parseString($0)) }

Or we can use the ```Compose``` and ```Bind``` Operators again:

	let toiletCountParser: String -> Result<Int> = promoteDecodeError("Could not parse 'disabled_parking") • Int.parseString
	let toiletCount =  XMLParser.parseChildText(["facilities", "toilet"])(xml: xml) >>- toiletCountParser

The ```Animal``` and ```Zoo``` Models both implement the ```XMLDecoderType``` protocol. In this case there are a number of ```Animal``` Models belonging to a ```Zoo```, so the ```decode``` functions can be reused to extract out each of the ```Animal``` Models contained in a parent ```Zoo```:

	let animals = XMLParser.parseChild("animals")(xml: xml) >>- XMLParser.parseChildren("animal") >>- resultMap(Animal.decode)

There is another function that takes the output of ```XMLParser.parseChildren``` as an input, of type ```Result<[X]>```. The ```resultMap``` function is like a regular ```map``` on an ```Array```, except with the return of a ```Result.Error``` if any of the applications of the ```map``` function fails:

	public func resultMap<A, B>(map: A -> Result<B>)(array: [A]) -> Result<[B]>

### Applicatives

Now we have everything we need to extract values out of ```XMLParsableType``` and into ```Result``` containers for each of the values that need to be extracted. Now the ```Result``` values need to be chained in the following way:

> _"If all of the ```Result```s corresponding to each of the Values are Successful, return our Model with the Values from the Result context applied to the Model Constructor function, otherwise just return the first Errored Result."_

Let's turn back to Curried Functions, this time to a Curried 'Constructor'[^constructor-factory-method]:

	static func build(kind: String)(name: String)(url: NSURL) -> Animal {
		return self(kind: kind, name: name, url: url)
	}

By adding a little more context to each of the applications of Higher-Order functions, we should be able to follow what is going on as the ```Animal.build``` function down to a ```Result<Animal>```:

	// These are previously defined using Higher-Order functions and XMLParser
	let kind: Result<String> = ...
	let name: Result<String> = ...
	let url: Result<NSURL> = ...
	
	// The Build Function is a Curried function that will yeild functions with every application until we have the Animal value.
	let build: String -> String -> NSURL -> Animal = self.build
	// Each of these stage gets the Animal structure closer to being initialized
	let first: Result<String -> NSURL -> Animal> = self.build <^> kind
	let second: Result<NSURL -> Animal> = first <*> name
	let third: Result<Animal> = second <*> url

This feels a lot like solving a Mathmatical equation by reducing the variables to balance each side of the equation. There are a few more Higer-Order functions that are being used here, doing the heavy lifting of pulling values and functions out of a ```Result``` context executing a function and then sticking the return value back in a ```Result``` again.

Firstly, the [```fmap``` operator](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Result.swift#L83):

	public func <^><A, B>(f: A -> B, a: Result<A>) -> Result<B>

It can be thought of in the following way:

> _"If the Result *a* is an Value apply the function *f* and place the result back in a Result, if the Result *a* is an Error, just return the Result immediately"_

The property of this operator is that the applied function ```f``` doesn't have to ever have to be aware that the applied argument has been part of a ```Result``` context.

Secondly, we can take a look the [```apply``` operator](https://github.com/maxpow4h/swiftz/blob/master/swiftz_core/swiftz_core/Result.swift#L94):

	public func <*><A, B>(f: Result<A -> B>, a: Result<A>) -> Result<B>

It too can be described:

>  _"If the Result *f* is an Error just return it immediately, if the Result *a* is an Error just return it immediately. Otherwise pull the function out of *f* and the value out of *a*, apply *a* to *f* then place the returned value in a Result"_

Again, these functions themseves no longer need to be aware that values are contained in a ```Result``` context. 

Between ```<^>```, ```<*>``` and ```>>-``` all the combinations of applying Values to Functions inside and outside of a ```Result``` are covered. These Operators allow us to use pull values and functions into-and-out-of ```Result``` contexts, using functions that don't have to be aware of the fact that the values and Functions may have come out of a ```Result```[^monads-as-contexts].

### Teach the Controversy

There's a lot to take in, some of the benefits should be clear, others are a bit more subtle. I don't deny some of this doesn't have a steep learning curve, it may go against many years of experience with languages that don't have semantics for dealing with Functions as things that can be combined in these ways. All of these concepts and implementations are nothing new and are the products of high-shouldered giants. With the modernity of Swift we have the opportunity to incorporate more Functional Programming into Native iOS and Mac Developments. Given that there are certain programming techniques that can't be carried over from Objective-C, its worth considering incorporating some Functional Programming if it fits the task at hand.

There's a lot of concern that Operator Overloading is going to lead to a lot of Smart People doing some very silly things[^blocks-operator-overloading] for the purpose of making dense, concise and far-too-clever code. The reality of these operators is that they are small in number and aren't specific to just ```Optional``` and ```Result```. As Arithmetic Operators deal with Numeric values, Functional Operators operators deal with the transformation of Functions themselves. These concepts are taken from other languages, so they aren't at all specific to Swift.

There's also some new terminologies and another way of describing the interaction of components, this isn't less true of a [Domain Specific Language](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage/VisualFormatLanguage.html) that we can graft on top of Objective-C[^objc-dsls] or with a [more flexible language](http://robots.thoughtbot.com/writing-a-domain-specific-language-in-ruby). Sure, Objective-C has the benefit of a preprocessor to graft new features onto the language to and reduce the amount of boiler-plate required[^macro-error-handling], but these can become pretty opaque over time. 

However, unlike a Domain Specific Language, all of the code used is valid Swift, there is no transformation from a language in one domain, to a syntax that the compiler understands. The Compiler is fully aware of the Types of all of the transformed Functions leaving no room for ambiguity. The chances of a runtime error because of a [keypath not existing](http://stackoverflow.com/questions/8561558/objective-c-keyvaluecoding-how-to-avoid-an-exception-with-valueforkeypath) or the type being [different to the one expected by the code](http://stackoverflow.com/questions/2455161/unrecognized-selector-sent-to-instance) are greatly diminished.

I honestly believe its worth taking a jump at the conceptual hurdle, whether it is for writing code that is more robust and predictable or to satisfy a curiosity of learning something new. 

Thanks for Reading! I suggest reading [some other brilliant posts](http://robnapier.net/functional-wish-fulfillment) that do a [better job than I](http://robots.thoughtbot.com/efficient-json-in-swift-with-functional-concepts-and-generics) of laying this all out! As a bonus, the next post will focus on how ```XMLParsableType``` can be implemented in a variety of ways.

---

[^swiftz-lightweight]: Swiftz has separated its functionality across a [core](https://github.com/maxpow4h/swiftz/tree/master/swiftz_core) library and the [library proper](https://github.com/maxpow4h/swiftz/tree/master/swiftz). This post will only use the Core library.

[^associated-types]: They consume a Protocol, abstracting away how the XML Parser is implemented. As the Protocol has an associated type requirement it has to be fulfilled with Generics. Composed behaviours don't pollute the Implementation of the Protocol but still augment the behaviour.

[^xmlparser-implementation]: The Implementation for this class is in the [next post in this series](/blog/2014/09/12/swift-parsers-libraries/).

[^swiftz-generics-simplification]: I'm lying, I've changed the Generic Parameters from ```VA``` & ```VB``` to ```A``` & ```B```

[^hypothetical-xml]: These assumptions actually hold true for a [webservice to be consumed](http://www.livedepartureboards.co.uk/ldbws/) in an Application I was prototyping. Depending on the Webservice an Application is consuming, there's a great deal of assumptions that can be made to reduce the complexity of an Implementation.

[^lazy-evaluated-functional-programming]: One of the interesting aspects of Haskell is [Lazy Evaluation](http://en.wikipedia.org/wiki/Lazy_evaluation), in particular how it applies to building up a [data structure and then traversing it](http://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf).

[^breaking-down-bind]: When starting out it can be really helpful to do this, it makes inspecting the types of each of the elements in the chain more visible. You can use Alt+Click on the value name to get XCode to print out the inferred type. Its also a good illustration of the power of type inference.

[^non-optional-guarantees]: Swift makes strong guarantees about the existance of values, there's no need to check for the existance of values in arguments that are non-optional. In an Objective-C implementation this contract can be enforced with a litany of assertions. If the language and compiler can enforce these guarantees we've got a huge productivity win on our hands.

[^xmlparser-class]: As the functions consume a Protocol with associated type requirements the Protocol type has to be defined as a Generic type. The functions are [Pure](http://en.wikipedia.org/wiki/Pure_function) in that they do not consume or modify any Global State, only the arguments are used. The guarantee of the no side-effects cannot be enforced in Swift. In essence the class is a bunch of similar functions that are kind-of-namespaced as Class methods of a ```final``` class with no constructor.

[^model-mapping-traditional]: I certainly remember writing this kind of thing by hand, before [Java annotations could be used for Code Generation](http://docs.oracle.com/javase/tutorial/jaxb/intro/).

[^constructor-factory-method]: 'Factory Method' is probably a better term for this since a 'Constructor' is a term reserved to ```init``` methods in Swift. 

[^nested-branching]: It would look [something like this](https://gist.github.com/lawrencelomax/00ea2c00c9b6ca5bb4ab)

[^monads-as-contexts]: ```Optional``` and ```Result``` are examples of Functors and Monads. This [fantastic article](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) covers the operators in a visual way. There's more of these (Monads) than you might first think and its a [crucial part of many functional languages](http://www.haskell.org/haskellwiki/Monad#Interesting_monads).

[^parsing-bloating-categories]: And it could be damaging to the codebase. Cocoa Classes could easily become polluted with ```Result``` variants of every possible method that could fail and this would continue over time as more classes are required to be parsed. More importantly it is not possible for a General ```Result``` returning type to have some of the crucial context surrounding a failure. We want our ```NSError```s to contain the Element within the XML responsible for the failure, an Extension method would lose this context. _'Failed to interpret 'toilet' as an Int'_ is preferable to _'Failed to interpret '123a' as an Int'_

[^blocks-operator-overloading]: This happens when any [*'new'*](http://en.wikipedia.org/wiki/Operator_overloading#1980s) language feature comes along. I know I abused blocks when they appeared in Objective-C.

[^macro-error-handling]: In Objective-C this can be handled with [Macros and Early Return](https://github.com/rentzsch/NSXReturnThrowError), but we can't rewrite/mangle the rules of the language in Swift as we don't have Macros.

[^decoder-protocols]: This is heavily inspired by the ThoughtBot article on JSON Parsing in Swift. In my Project I have multiple decoder types for JSON, XML and CSV.

[^swiftz-lightweight]: Swiftz has a Core library as well as a more full-featured Framework. There's a lot of interesting stuffin the Full library, but just getting started with the Core is a good place to start.
