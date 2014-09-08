---
layout: post
title: "Swift Parsers - Imperative"
date: 2014-09-04 15:00:00 +0100
comments: true
categories: 
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

Stubbing a Protocol or Interface is a great way of getting to grips with the problem that you want to Solve. It also helps you to crystallise what is important, what details we can ignore to solve the problem that we want. It also helps when [considering the requirements that were previously laid out]() 

In parsing this XML[^hypothetical-xml]. I've made a few assumptions:

1. I care about the data contained in Text Nodes.
3. I need to be able to recursively address Elements within a Tree-like structure.
4. I need to be able to enumerate Elements of the same name at the same level in the tree.
5. I don't care about anything else (namespaces, attributes, schemas).

With those assumptions in mind, a Protocol for defining how data can be extracted from a Parsable XML Node can be made:

    public protocol XMLParsableType {
      func parseChildren(childTag: String) -> [Self]
      func parseText() -> String?
    }

*"That's It?"*. Yep. Everything else can be composed on top of this minimal protocol. More complex extraction functions can be built on top of these fundementals. By defining a Protocol in this way it will become significantly easier to build an implementation of this Protocol, as we will see in [Part 4](). This leads to a definition of the what this Protocol represents in one sentence:

> _"An XML Parsable has an ordered collection of Child Parsables and may have associated text"_

Protocols are permitted to have a recursive definition, using the `Self` placeholder type. How and where the fetched data is stored is left to the implementing class/struct/enum[^lazy-evaluated-functional-programming]. Obtaining a child may be implemented by traversing a fully reified data structure, or moving a cursor along a buffer.

As well as a representation of the Data, there needs to be a consistent way of defining that a Model can extract out values of The question now becomes how we can best extract values from our XML interface, into our Model classes. The entry point can be defined in terms of a decode protocol that the Model structures should implement[^decoder-protocols]:

	public protocol XMLDecoderType {
	  class func decode<X: XMLParsableType>(xml: X) -> Result<Self>
	}

### Surfacing Failure

In the interface for ```XMLParsableType``` failure is indicated with the an Optional return type. We can use the ``Result`` type as it availability semantics that are similar to an ```Optional```, with additional information provided with an ```NSError``` in the failure case:

	public enum Result<V> {
	  case Error(NSError)
	  case Value(Box<V>)
	}
	
```NSError``` has the an associated of an "Error Domain" as well as ```String``` Description of the cause of the failure. ```NSError``` passed by reference is also ubiquitous and idiomatic in Cocoa and plays nicely with the ```Result``` type. Exceptions are being [weeded out of Cocoa API](https://twitter.com/atnan/status/506832064633901056) for all but programmer error.

Because constructing an ```Result``` from an ```Optional``` is a repetitive task in our ```Decoder``` it is worth writing a helper function to 'promote' an ```Optional``` to a ```Result``` with a default Error Domain:

	public func promoteDecodeError<V>(message: String)(value: V?) -> Result<V>

### An Imperative Approach

From the previous post, I discussed how the lack of a dynamic runtime in Swift will prevent us from using some of the programming techniques of Objective-C. So we will have to take a more traditional approach to parsing that will look a lot more like the techniques we may have used when we first learned programming[^model-mapping-traditional]. 

A key requirement of our Parsing code is that it should fail if one of the values is missing in the Serialization Format. The following Imperative approach shows how it may be possible to extract a ```Result<Animal>``` from an ```XMLParsableType```:

	  static func decodeImperative<X: XMLParsableType>(xml: X) -> Result<Animal> {
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

This doesn't look great. The nesting is terrible, the Failure and Success conditions are separated around the conditional. In this case, there are only three values, a Model with more properties would make the nesting significantly worse. In Objective-C this can be better tackled by returning early on failure, however this would require lots of force unwrapping[^nested-branching].

However, there are some patterns that are emerging. Firstly, it looks like extracting the text of a child is important this can probably be extracted out. Obtaining a grandchild also looks to be something we can extract out. Optional Chaining is being used to avoid further nesting, this looks familiar coming from Objective-C. 

### Next Time...

Next time, we'll take a Functional approach to these problems and reap the rewards.

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
