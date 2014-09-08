---
layout: post
title: "Swift Parsers - Libraries"
date: 2014-09-06 15:00:00 +0100
comments: true
categories: 
---

In the previous posts in this series the Parsers have used a hypothetical ```XMLParserType``` Protocol for fetching data from an XML Element. I previously mentioned that there are myriad libraries and methods for parsing an XML document. In this post I'm going to consider how you would implement such an interface.

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

## libxml2

The immensely popular Open Source project [libxml2](http://xmlsoft.org) library is a C Framework bundled with iOS and Mac OS X and has no external dependencies. It provides implementations of the DOM, SAX and Reader interfaces so is an ideal candidate for our XML Parser.

NSXMLParser has been part of Cocoa for many years now and would also be suitable, however it isn't as interesting since it only has a ```SAX``` interface. While it is certainly possible to make a ```XMLParserType``` implementation using NSXMLParser, I won't consider it for now.

### C Structures in Swift

As ```libxml2``` is a C library, the C functions can be imported into Swift using an Bridging Header. Working with the structures requires quite a bit of unwrapping and manupilating 'Unsafe' containers that wrap all C structures in Swift. Again, a strong reason to move up to the Swift level of abstraction.

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

I like to follow a principle when developing software, regardless of the language:

> _"Get to the highest level of Abstraction that makes sense, as early as possible. Don't think about the overhead of the abstration until you can prove that it is detrimental to performance"_

```libxml2``` sits very far down the ladder of Abstraction, when working with it in Swift best practice will be pull data out of C structures and into Swift native ones as early as possible. This means skipping Objective-C[^string-nsstring] and using elements of the Swift Standard Library to consume data. Swift provides us with some abstractions that are superior to the Objective-C equivalents, there's no need for an intermediate stage unless it is needed.

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

By moving enumeration to a ```Sequence``` type, consumers of the DOM don't need to be concerned with how the list of Child Node is navigated.

## Attempt 1: Build a Swift Data Structure

The first thing that springs to mind is that we allready have a Data Structure in ```libxml``` and we can extract it out to a pure Swift data structure[^class-vs-struct]. This tree of Swift structures can trivially implement the previously defined ```XMLParsableType```.

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

The ```childrenOfNode``` function is placed in the ```LibXMLDOM``` class for separating the concerns of how the Tree is exposed and how it is navigated. By using this function that returns a  ```Sequence`` and a number of other wrapper functions (prefixed with ```LibXMLDOM```) returning Swift types, the construction of the tree can be defined recusively:

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

That's pretty much it, there's a little more in terms of plumbing and error handling, but its not too hard to take an XML Document from ```libxml2``` and get the data into Swift.

## Attempt 2: Wrap the libxml Tree

The Building of the tree as a fully realized structure is a Simple implementation, but it certainly isn't the least resource intensive. Swift ```String```s for the Text and Element Name of an XML Element are created regardless of whether they are used or not.

There's no reason that we can't just wrap the whole ```libxml2``` Tree structure in a Class implementing ```XMLParsableType``` that knows how to fetch the Text and Children of an Element.

	public final class LibXMLNavigatingParserDOM: XMLNavigatingParserType, XMLParsableType {
	  private let node: xmlNodePtr
	  private let context: LibXMLDOM.Context?
	  
	  internal init (node: xmlNodePtr) {
	    self.node = node
	  }
	  
	  internal init (context: LibXMLDOM.Context) {
	    self.context = context
	    self.node = context.rootNode
	  }
	  
	  deinit {
	    self.context?.dispose()
	  }
	  
	  public func parseChildren(childTag: String) -> [LibXMLNavigatingParserDOM] {
	    let foundChildren = filter(LibXMLDOM.childrenOfNode(self.node)) { node in
	      LibXMLDOMGetElementType(node) == LibXMLElementType.ELEMENT_NODE && LibXMLDOMElementNameEquals(node, childTag)
	    }
	    return foundChildren.map { LibXMLNavigatingParserDOM(node: $0) }
	  }
	  
	  public func parseText() -> String? {
	    let foundChild = findSeq(LibXMLDOM.childrenOfNode(self.node)) { node in
	      return LibXMLDOMGetElementType(node) == LibXMLElementType.TEXT_NODE
	    }
	    return foundChild >>- LibXMLDOMGetText
	  }  
	}

The ```LibXMLDOM.Context``` is just a internal class responsible for cleaning up the manual-memory-managed ```libxml2``` structures on deallocation:

	internal final class LibXMLDOM {
	  struct Context {
	    let document: xmlDocPtr
	    let rootNode: xmlNodePtr
    
	    init (document: xmlDocPtr, rootNode: xmlNodePtr){
	      self.document = document
	      self.rootNode = rootNode
	    }
    
	    func dispose() {
		   if self.rootNode != nil {
        	xmlFreeDoc(self.document)
      	   }
	    }
	  }
	}

Again, that's about all we need to implement the ```XMLParsableType``` interface. The creation of the Swift Strings for the Text of an Element are deferred until they are actually needed. If nearly all of the Text Elements are consumed when mapping to a model it won't have much (if any) impact on performance relative to the first attempt.

## Attempt 3: Navigate with a Reader

Our restricted representation of what XML can be is based purely on the need to access Text Nodes within a Tree. Depending on what we are ignoring in the XML Document, the parser could potentially read a large number of Attributes and ignored Elements into memory that never need to be pulled out.

The DOM parser for ```libxml2``` will have parsed the whole document, elements, attributes and all by the time it is created. Depending on the how much of the document is ignored, we may get significant performance benefit. This will also prove that the cost for parsing an XML document is heavily influenced by the XML Parser costs themselves, not the ```XMLParseableType``` abstraction itself and any composition on top.

## Testing

One of the great aspects of building an XML Parser in this way is that the Implementations are trivial to swap out and can be tested against for correctness against the same Parsing Code. Additionally, we can use the new Performance Testing features of XCode 6 to see what the performance is like for these varying methods.
 
[^dom-performance]: I cut my programming teeth on Java. When consuming an XML Document of greater than a MB or so the JVM Heap could get hammered when using a DOM style parser. This could be hugely problematic when intertwined with a Garbage Collector. I wonder if the ubiquity of JSON Parsers that output to a fully reified Data Structure, rather than Event Based parsers has anything to do with the increased availability of resources since JSON came into vogue.

For this reason the Event-Based Streaming SAX parser could be used instead, the whole document need not be read into memory at the expense of a more troublesome interface. Eventually a compromise was found with the StAX parser, buffering the input document with a Cursorable navigation mechanism.
 
[^class-vs-struct]: This is the perfect candidate for usage of a ```struct``` instead of a ```class```. We don't need for this to be subclassed, or any disposal semantics in ```dealloc```. Reference counting is totally uncessary as this structure represents an Immutable value. I'd love to take a look at the performance of passing structs around, [it looks like Array and Dictionary struct types use copy-on-write](https://devforums.apple.com/message/990361#990361) but I have no idea if this applies to User defined structs. 

[^string-nsstring]: Ok I lied, there I use a lot of Bridging functions that use ```NSString```, but an ```NSString``` is instantly converted to a Swift ```String```
