---
layout: post
title: "Swift Parsers - Combinations"
date: 2014-09-04 11:04:36 +0100
comments: true
categories: 
---

In the previous post I've talked about an Interface and Functions for a simplified XML parser. In this post I'll cover how these elements can be composed to map XML to a Model Object.

### Requirements of a Parser

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

There's a lot going on and this would require a substantial effort in an Imperative style using Objective-C. Libraries and Frameworks can help a lot, but they can only go so far. For example it isn't possible to inspect the expected type of the destination property in the Model. This is a common landmine, an Objective-C class isn't checked that it is the appropriate class when it is cast so failure will occur away from the source of the error[^assertions-exceptions].

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
		<facilities>
	</zoo>


The Model includes the parts of the XML that we care about and ignores others.

### Applying Functional Principles

The structure of Imperative parsing code is as follows[^nested-branching]:
	
	public enum Result<V> {
	  case Error(NSError)
	  case Value(Box<V>)
	}
	
	...
	
	
	if let 
	
I'm using the fantastic and lightweight[^swiftz-lightweight] [Swiftz](https://github.com/maxpow4h/swiftz/) for go-to implementations of many of the Functional primitives.


The Branching in Imperative Parsing code exists as it is how operations are chained together. Instead of:

Interfacing with other parsing functions

Bind:

	public func >>-<A, B>(a: A?, f: A -> B?) -> B?

Apply:

	public func <*><A, B>(f: (A -> B)?, a: A?) -> B?

Fmap:

	public func <^><A, B>(f: A -> B, a: A?) -> B?

Breaking each optional down.
Futher parsers such as parseNum
Result
Auto-result
Model of Submodels

Lets take a look at what this gives us:

1. No Branching
2. Detailed Errors with automatically generated messages at the cause of the error.

### Teach the Controversy

I don't deny some of this doesn't have a cost in terms of a leaning curve, it can look very alien at first. However, this isn't less true of any Domain Specific Language that we can graft on top of Objective-C or with a [more flexible language](http://robots.thoughtbot.com/writing-a-domain-specific-language-in-ruby). There are a whole host of formal and informal Objective-C libaries have many ways of expressing[^objc-dsls]. It just so happens that these Higher-Level  operators are applicable outside the domain of parsing and exist in concept and practice across other programming languages.

We have had a similar behaviour in Objective-C since its inception; [Sending Messages to Nil](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/ProgrammingWithObjectiveC/WorkingwithObjects/WorkingwithObjects.html).  This behaviour is just as alien to newcomers to Objective-C as some of these functional concepts are! 

Sure, we'll see a lot code that goes crazy with Infix operators and devolves into an incomprehensible meta-language that is only understandable by the creator. However, there is a great deal of merit in the targeted and limited
Macros are DSLs allready
Monads are established
Get to the best level of abstraction earlier

[^monadic-parser]: The concept of a 'Parser' as a distinct type that implements the 'Monad' Typeclass [does indeed exist in Pure functional language](http://eprints.nottingham.ac.uk/223/1/pearl.pdf).
[^nested-branching]: The nested branching is inconvenient, especially if you like to return early on failure. Fast Failure can be implemented with force unwrapping
		
		let ma
		

[^swiftz-lightweight]: Swiftz has a Core library as well as a more full-featured Framework. There's a lot of interesting stuffin the Full library, but just getting started with the Core is a good place to start.
