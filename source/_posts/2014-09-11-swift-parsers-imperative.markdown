---
layout: post
title: "Swift Parsers - Imperative"
date: 2014-09-11 22:30:00 +0100
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

In the previous post I talked about some of the possible requirements can be for Parser that extracts data from a  Serialization Format and places it in a Language-Native Model Object. In this post I'll cover how an XML Parser for a Model Object can be built using some of the familiar Imperative features of Swift.

### Model and XML

Let's define a hypothetical XML that we wish to parse:

	<zoo>
		<animals>
			<animal>
				<type>cat</type>
				<name>grumpy</name>
				<url>http://en.wikipedia.org/wiki/Grumpy_Cat</url>
			</animal>
			<animal>
				<type>cat</type>
				<name>long</name>
				<url>http://knowyourmeme.com/memes/longcat</url>
			</animal>
			<animal>
				<type>dog</type>
				<name>i have no idea what i'm doing</name>
				<url>http://knowyourmeme.com/memes/i-have-no-idea-what-im-doing</url>
			</animal>
		</animal>
		<facilities>
			<toilet>42</toilet>
			<disabled_parking>true</disabled_parking>
			<a_random_assortment_of_characters>sdfdf821n9sfa</a_random_assortment_of_characters>
			<seriously>
				<crazy_nested>
					<drainage>Good</drainage>
				</crazy_nested>
			</seriously>
		</facilities>
	</zoo>

The Model includes the parts of the XML that our Application cares about and ignores others:

	public struct Animal {
	  let type: String
	  let name: String
	  let url: NSURL
	}
	
	public struct Zoo {
	  let toiletCount: Int
	  let disabledParking: Bool
	  let drainage: String
	  let animals: [Animal]
	 }

This is simple and immutable, the Parser forms part of the backend for the User Interface to consume. There's no reason for the User Interface to be able to manipulate these Models directly.

### An Interface to XML

Stubbing a Protocol or Interface is a great way of getting to grips with the problem that needs to be solved. It also helps to determine what is necessary for to implement, as well as the details that can be ignored to solve the problem at hand. There is also a bunch of [previously laid out requirements to be considered]() 

In parsing this XML[^hypothetical-xml]. I've made a few assumptions:

1. I care about the data contained in Text Nodes.
2. I need to be able to recursively address Elements within a Tree-like structure.
3. I need to be able to enumerate Elements of the same name at the same level in the tree.
4. I don't care about anything else (namespaces, attributes, schemas).

With those assumptions in mind, a Protocol for defining how data can be extracted from a Parsable XML Node can be made:

    public protocol XMLParsableType {
      func parseChildren(childTag: String) -> [Self]
      func parseText() -> String?
    }

*"That's It?"*. Yep. Everything else can be composed on top of this minimal protocol; more complex data extraction functions can be built on top of these fundamentals. It's easy to define the responsibility of this Protocol in one sentence:

> _"An XML Parsable has an ordered collection of Child Parsables and may have an associated String of Text"_

Protocols are permitted to have a recursive definition, using the `Self` placeholder type. How and where the underlying XML document is stored is left to the implementing class/struct/enum[^lazy-evaluated-functional-programming]. As we will see in [Part 4](), obtaining a child may be implemented by traversing a fully reified data structure, or moving a cursor partially visible representation of the structure.

As well as a representation of the Data Serialization itself, there needs to be a consistent way of defining that a Model can extract out values of Data Serialization. The entry point can be defined in terms of a decode protocol[^decoder-protocols] that the Model structures should implement:

	public protocol XMLDecoderType {
	  class func decode<X: XMLParsableType>(xml: X) -> Result<Self>
	}

This function will be where the action is and can be implemented in the Model type definitions themselves or separately as Extensions. As ```XMLParsableType``` has a ```Self``` requirement, the usage of an ```XMLParsableType``` protocol needs to be [satisfied with Generics](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-XID_286).

### Surfacing Failure

In the interface for ```XMLParsableType```, failure is indicated with the return of [the ```None``` case of the ```Optional``` enum](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-XID_483). An ```Optional``` surrounds the value with a [Context describing the availability of the value](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html). The absence of a value in a ```XMLParsableType``` indicates some kind of failure, but without any information about how the Error came about.

Some Cocoa APIs use ```nil``` as the return value represent failure[^nil-vs-none] alone, but idiomatic Cocoa will also include a by-reference ```NSError``` to return Error Information in the event of failure. Exceptions are being [weeded out of Cocoa API](https://twitter.com/atnan/status/506832064633901056) for all but programmer error. ```NSError``` has the an associated ["Error Domain"](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSError_Class/Reference/Reference.html#//apple_ref/occ/instm/NSError/domain) as well as [```String``` Description](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSError_Class/Reference/Reference.html#//apple_ref/doc/uid/20001704-CJBIAHGF) of the cause of the failure. This can be massively helpful as finding the code or resource that is responsible is a nightmare if the only information is "a problem occurred somewhere". For example the ```NSJSONSerialization``` API will give the line-number of a syntax error in a JSON resource.

Moving to a Safe Swift world, return of an ```NSError``` and a possible value can be encapsulated in the same value using an [Enumeration with Associated Values](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Enumerations.html#//apple_ref/doc/uid/TP40014097-CH12-XID_227), rather [dereferencing pointers](https://www.google.co.uk/search?client=safari&rls=en&q=nserror+dereference+pointer&ie=UTF-8&oe=UTF-8&gfe_rd=cr&ei=3MEOVNesH4G28AOjw4D4Bw#rls=en&q=nserror+dereference+pointer). This is the ``Result`` with the same availability semantics as an ```Optional```, with additional failure information provided with an ```NSError``` in the failure case:

	public enum Result<V> {
	  case Error(NSError)
	  case Value(Box<V>)
	}

As there are potentially many sources of failure in the ```decode``` method, it is handy to write a Helper Method that can "promote" an ```Optional``` to a ```Result``` with an Error if the Optional does not contain a value. This will populate the ```NSError``` with a default Error Domain and attach a User Defined message:

	public func promoteDecodeError<V>(message: String)(value: V?) -> Result<V>

### An Imperative Approach

From the previous post, I mentioned that the lack a dynamic runtime in Swift will mean that bringing over some Objective-C programming techniques will be impossible. As a Swift Parser won't be able to use these techniques, a more traditional approach to parsing will have to be used[^model-mapping-traditional]. 

Failing-Fast was outlined as a potential feature of a Parser in the previous post, the implementation of an XML-to-Animal decoding function will need to take this into account. The following Imperative approach shows how it may be possible to extract a ```Result<Animal>``` from an ```XMLParsableType```:

	  static func decode<X: XMLParsableType>(xml: X) -> Result<Animal> {
	    if let type = XMLParser.parseChildText("type")(xml: xml).toOptional() {
	      if let name = XMLParser.parseChildText("name")(xml: xml).toOptional() {
	        if let urlString = XMLParser.parseChildText("url")(xml: xml).toOptional() {
	          return Result.value(self(type: type, name: name, url: NSURL.URLWithString(urlString)) )
	        }
	        return Result.Error(decodeError("Could not parse 'urlString' as a String"))
	      }
	      return Result.Error(decodeError("Could not parse 'name' as a String"))
	    }
	    return Result.Error(decodeError("Could not parse 'type' as a String"))
	  }

This doesn't look great. The nesting is terrible, the Failure and Success conditions are separated around the conditional. In this case, there are only three values, a Model with more properties would make the nesting significantly worse. In Objective-C this can be better tackled by returning early on failure, however this would require lots of force unwrapping[^nested-branching]. Conditional Statements are required as Failure when one of the values is missing is a requirement of our Model and is guaranteed to exist in the XML. 

Despite these problems, there are some patterns are emerging. Firstly, it looks like extracting the Text from a Child is a common enough task that can be converted to a function in its own right. Obtaining a Child's Child's looks like an interesting area for some more meaningful functions. Optional Chaining is being used to avoid further conditionals. 

Most importantly is that the nesting is an Imperative way of implementing that the successful condition for this function is dependent on the availability of three values in the XML. Moving our understanding of the pattern from an Imperative model to a Declarative model will be crucial to building a better way of constructing a ```decode``` function.

### Next Time...

Next time, we'll take a Functional approach to the ```decode``` method, allowing us to think at a much higher level about how a Model is built.

[^hypothetical-xml]: These assumptions actually hold true for a [webservice to be consumed](http://www.livedepartureboards.co.uk/ldbws/) in an Application I was prototyping. Depending on the Webservice an Application is consuming, there's a great deal of assumptions that can be made to reduce the complexity of an Implementation.

[^nil-vs-none]: It is really important to understand the semantic differences between ```nil``` in Objective-C and ```None```/```nil``` in Objective-C. With Swift/Objective-C interoperability they are used interchangeably. In Swift they can be thought of as "the absence of a value", but in Objective-C they can be both "the absence of a value" and a [terminal](http://en.wikipedia.org/wiki/Null-terminated_string). Swift features [Optional Chaining](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/OptionalChaining.html#//apple_ref/doc/uid/TP40014097-CH21-XID_356) to replicate the nil-messaging of Objective-C. As Objective-C APIs cannot make guarantees about the availability of a Reference Type all of the Objective-C bridged APIs, [Reference Types are exposed as Optionals & Force-Unwrapped Optionals](ceptual/BuildingCocoaApps/InteractingWithObjective-CAPIs.html#//apple_ref/doc/uid/TP40014216-CH4-XID_30).

[^lazy-evaluated-functional-programming]: One of the most interesting aspects of Haskell is [Lazy Evaluation](http://en.wikipedia.org/wiki/Lazy_evaluation), in particular how it applies to building up a [data structure and then traversing it](http://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf).

[^decoder-protocols]: This is heavily inspired by the [ThoughtBot article on JSON Parsing in Swift](http://robots.thoughtbot.com/efficient-json-in-swift-with-functional-concepts-and-generics). In my Project I have multiple decoder types for JSON, XML and CSV.

[^model-mapping-traditional]: I certainly remember writing this kind of thing by hand, before [Java annotations could be used for Code Generation](http://docs.oracle.com/javase/tutorial/jaxb/intro/).

[^nested-branching]: It would look [something like this](https://gist.github.com/lawrencelomax/00ea2c00c9b6ca5bb4ab)
