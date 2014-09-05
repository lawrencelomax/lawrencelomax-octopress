---
layout: post
title: "Swift Parsers - Libraries"
date: 2014-09-06 15:00:00 +0100
comments: true
categories: 
---

In the first post in this series I mentioned that there are myriad libraries and interfaces for parsing an XML document. I'm going to use the immensely popular [libxml2]() library. It is a C Framework bundled iOS and Mac OS X with no external dependencies. 

## [Why Lisa Why?](http://www.youtube.com/watch?v=pjc4ZPTX1XQ)

All of this sounds like a fun excercise, I found it to be a great way to introduce myself to Swift and apply some of the Functional Concepts into my Code. I needed to Parse XML in a Project that I was working on in a few spare weeks between jobs.

The majority of this code was written with an Application in mind. I didn't have long at all (in fact I wasn't even close to having the right amount of time). I was working on an iOS App that would show the fastest train home (not necessarily the next train) to help out commuters decide which train would give them the most time at the office and at home [_(I'm about to become one of these people)_](https://www.google.co.uk/maps/dir/Coventry,+United+Kingdom/London+Euston,+United+Kingdom/@51.9631488,-1.3744911,9z/data=!3m1!4b1!4m14!4m13!1m5!1m1!1s0x48774bae1512f593:0x9881aa5f47d741c1!2m2!1d-1.51345!2d52.400828!1m5!1m1!1s0x48761b25a93c9b83:0xdbced150c2761d8f!2m2!1d-0.133898!2d51.528136!3e3).

Regardless, the code is [available on GitHub to pick apart and prod]().

## DOM

A DOM or Tree Interface to XML loads every single node into memory. Attributes, Elements, Text, all of it. This makes the memory usage proportional to the size of the XML Document. This is fine for small documents, but for larger ones it can severely impact performance[^dom-performance]. If the Document is going to be repeatedly navigated, this is benefitial. Creating the resource once and then navigating it does all the expensive work up front, navigating the Tree is no more expensive than walking a data structure. 

However, the cost of constructing the tree is not worth the effort if we're only concerned with a small number of elements within the Tree. As the performance and amount of memory available in iOS devices improves, we've been able to solve performance problems by throwing more system resources at the problem. There's nothing inherently wrong with this, but we can still better spend those constrained resources elsewhere. Reducing the amount of resources an Application uses is crucuially important as far as battery life is concerned.

## Event Based 

An E

## Reader

### Working with Swift

I like to follow a principle when developing software:

> _"Get to the highest level of Abstraction that makes sense, as early as possible. If the abstraction has too much of a performance impact, worry about that later"_

Not only does this lead to better architecture, it also makes easier to understand units of work.

### C Structure Access

libxml2 is a C library, the C functions can be imported into Swift using an Bridging Header. Working with the structures requires quite a bit of unwrapping and manupilating 'Unsafe' containers that wrap all C structures in Swift. Again, a strong reason to move up to the Swift level of abstraction.

Direct access of C structure members is not permitted in Swift as Swift does not know about the shape of structures. Any access of structure members will need to be wrapped in C/Objective-C functions and imported in the Bridging Header.

For example, this function will extract the `content` member from a Node:

	 NSString * LibXMLDOMGetText(xmlNodePtr node)
	 {
	   NSCParameterAssert(node != NULL);
	   NSCParameterAssert(node->type == XML_TEXT_NODE);
	   	
	   const char *name = (const char *) node->content;
	   NSCParameterAssert(name != NULL);
	   
	   return [NSString stringWithCString:name encoding:NSUTF8StringEncoding];
	 }


### GOTO Swift

Skip Objective-C and use elements of the Swift Standard Library to consume data. Swift provides us with some abstractions that are superior to the Objective-C equivalents, there's no need for an intermediate stage unless it is needed.

Conveniently, XCode will parse headers with enumerations declared with the ```NS_ENUM``` macro:

	typedef NS_ENUM(NSInteger, LibXMLReaderType) {
	  LibXMLReaderTypeNONE = 0,
	  LibXMLReaderTypeELEMENT = 1,
	  LibXMLReaderTypeATTRIBUTE = 2,
	  LibXMLReaderTypeTEXT = 3,
	  LibXMLReaderTypeCDATA = 4,
	  LibXMLReaderTypeENTITY_REFERENCE = 5,
	  LibXMLReaderTypeENTITY = 6,
	  LibXMLReaderTypePROCESSING_INSTRUCTION = 7,
	  LibXMLReaderTypeCOMMENT = 8,
	  LibXMLReaderTypeDOCUMENT = 9,
	  LibXMLReaderTypeDOCUMENT_TYPE = 10,
	  LibXMLReaderTypeDOCUMENT_FRAGMENT = 11,
	  LibXMLReaderTypeNOTATION = 12,
	  LibXMLReaderTypeWHITESPACE = 13,
	  LibXMLReaderTypeSIGNIFICANT_WHITESPACE = 14,
	  LibXMLReaderTypeEND_ELEMENT = 15,
	  LibXMLReaderTypeEND_ENTITY = 16,
	  LibXMLReaderTypeXML_DECLARATION = 17
	};

This is a redeclaration of libxml enum types, with a naming convention that works well in Swift:

	  switch LibXMLDOMGetElementType(child) {
	  case .ELEMENT_NODE:
	    children.append(LibXMLNodeParserDOM.createTreeRecursive(child))
	  case .TEXT_NODE:
	    text = LibXMLDOMGetText(child)
	  default:
	    break
	  }

Swift provides the ```SequenceType``` and ```GeneratorType``` protocols to allow us to use the ```for ... in ...``` syntax, or use the higher-level ```map```, ```search``` and ```filter``` functions. We can define a ```Sequence``` type for the Child nodes of an XML Node:

	typealias LibXMLDOMNodeSequence = SequenceOf<xmlNodePtr>
	
	internal class func childrenOfNode(node: xmlNodePtr) -> LibXMLDOMNodeSequence {
		var nextNode = LibXMLDOMGetChildren(node)
		let generator: GeneratorOf<xmlNodePtr> = GeneratorOf {
		  if (nextNode == nil) {
		    return nil
		  }
	  
		  let currentNode = nextNode
		  nextNode = LibXMLDOMGetSibling(nextNode)
	  
		  return currentNode
		}
	
		return SequenceOf(generator)
	}

Moving enumeration to these interfaces totally removes the need for consumers of this function to know how the list of Child nodes is navigated.

## Attempt 1: Build a Swift Data Structure

In our first attempt, we will convert the tree in ```libxml``` into a Data Structure in Swift. The Tree of Nodes can implement the previously defined ```XMLParsableType```.

	public final class XMLNode: XMLParsableType {
	  public let name: String
	  public let text: String?
	  public let children: [XMLNode]
	  
	  init (name: String, text: String?, children: [XMLNode]) {
	    self.name = name
	    self.text = text
	    self.children = children
	  }
    
	  public func parseChildren(childTag: String) -> [XMLNode] {
	    return self.children.filter { node in
	      return node.name == childTag
	    }
	  }
	  
	  public func parseText() -> String? {
	    return self.text
	  }
	}

Leaning on the ```childrenOfNode``` function that yeilds a ```Sequence`` and a number of other wrapper functions that return Swift types, building of this tree becomes a simple recusive function:

	private class func createTreeRecursive(node: xmlNodePtr) -> XMLNode {
	    let name: String! = LibXMLDOMGetName(node)
	    var text: String?
		
	    var children: [XMLNode] = []
		
	    for child in LibXMLDOM.childrenOfNode(node) {
	      switch LibXMLDOMGetElementType(child) {
	      case .ELEMENT_NODE:
	        children.append(LibXMLNodeParserDOM.createTreeRecursive(child))
	      case .TEXT_NODE:
	        text = LibXMLDOMGetText(child)
	      default:
	        break
	      }
	    }
		
	    return XMLNode(name: name, text: text, children: children)
	}


## Attempt 2: Wrap the libxml Tree

## Attempt 3: Navigate with a Reader

## Testing

One of the great aspects of building an XML Parser in this way is that the Implementations are trivial to swap out and can be tested against for correctness against the same Parsing Code. Additionally, we can use the new Performance Testing features of XCode 6 to see what the performance is like for these varying methods.
 

[^dom-performance]: I cut my programming teeth on Java. When consuming an XML Document of greater than a MB or so the JVM Heap could get hammered when using a DOM style parser. This could be hugely problematic when intertwined with a Garbage Collector. I wonder if the ubiquity of JSON Parsers that output to a fully reified Data Structure, rather than Event Based parsers has anything to do with the increased availability of resources since JSON came into vogue.


 For this reason the Event-Based Streaming SAX parser could be used instead, the whole document need not be read into memory at the expense of a more troublesome interface. Eventually a compromise was found with the StAX parser, buffering the input document with a Cursorable navigation mechanism.
 


