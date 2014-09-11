---
layout: post
title: "Swift Parsers - Libraries"
date: 2014-09-06 15:00:00 +0100
comments: true
published: false
categories:
- Functional
- Swift
- iOS
- Objective-C
- Parsers
- Backends
---

In the previous posts in this series the Parsers have used a hypothetical ```XMLParsableType``` Protocol for fetching data from an XML Element. I previously mentioned that there are myriad libraries and methods for parsing an XML document for iOS. 

In this post we're going to take a look at two things: 
1. How ```XMLParsableType``` can be implemented using ```libxml2```. 
2. How the various interfaces of ```libxml2``` and abstractions in Swift can result in drastically variant performance.

## Parser Interfaces

The immensely popular Open Source project [libxml2](http://xmlsoft.org) is a C Framework bundled with iOS and Mac OS X for parsing XML, with no external depenedencies. For our interests it has implementations for a number of [XML Reading interfaces](http://en.wikipedia.org/wiki/Java_API_for_XML_Processing). These are the [DOM](http://en.wikipedia.org/wiki/Document_Object_Model), [SAX](http://en.wikipedia.org/wiki/Simple_API_for_XML) and [Reader](http://xmlsoft.org/xmlreader.html) interfaces.

[NSXMLParser](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/nsxmlparser_class/reference/reference.html) has been part of Cocoa for many years now and would also be suitable, however it only implements an event-based ```SAX```-style interface. While it is certainly possible to make a ```XMLParserType``` implementation using NSXMLParser, I won't consider it for now.

If I were to build these interfaces in an Application I'd want to be cautious to not prematurely optimize the XML Document Reading process. Even if there are any substantial gains to XML Parsing performance when building implementations of the ```XMLParsableType```, the gains may not be that significant to the Application as a whole. If the implementations were to be made from scratch it would be worth writing one simple implementation and the swapping it with a more performant implementation if-and-when it is seen to be impacting the performance of the Application at large. However, this is a fun excercise to play with Swift and Xcode 6 as well as getting to grips with the ```libxml2``` library.

### DOM

A DOM or Tree Interface to XML represents all of the Elements within an XML Document as a tree of connected nodes. This often means that the whole of an XML Document is read into memory and is navigible by following relationships in a data strucure. By exposing the XML Document as a fully realized data structure, a DOM Interface very convenient to navigate.

By loading all parts of the XML Document into memory the memory usage is proportional to the size of the XML Document. This will be a low cost for small documents, but for larger documents this can result in a substantial amount of memory to allocate[^dom-performance]. 

The performance of a DOM parser relative to others is largely dependent on the number of times that the same DOM is traversed and the amount of ignored data in the DOM. If the DOM is repeatedly traversed, then the up-front cost of building the tree will amortize later navigation of the data structure. However, if there are only a few Elements and Text Nodes that need to be extracted from a Large Document, the resources used to read the unread parts of the tree are wasted.
	
### SAX

An E

### Reader

## Swift Implementation

With a little context to the types of parsing interfaces that are available, it should be easier to understand how their APIs will operate and how they can be used in Swift.

### C Structures in Swift

[```libxml2```]()  Open Source implementation of is a C library, the C functions can be imported into Swift using an Bridging Header and used as if they are Swift functions with [usual rules of interacting with C API](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-XID_11). As Working with the structures requires quite a bit of unwrapping and manupilating 'Unsafe' containers that wrap all C structures in Swift. Again, a strong reason to move up to the Swift level of abstraction.

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
	  LibXMLReaderTypeNONE = XML_READER_TYPE_NONE,
	  LibXMLReaderTypeELEMENT = XML_READER_TYPE_ELEMENT,
	  LibXMLReaderTypeATTRIBUTE = XML_READER_TYPE_ATTRIBUTE,
	  LibXMLReaderTypeTEXT = XML_READER_TYPE_TEXT,
	  LibXMLReaderTypeCDATA = XML_READER_TYPE_CDATA,
	  LibXMLReaderTypeENTITY_REFERENCE = XML_READER_TYPE_ENTITY_REFERENCE,
	  LibXMLReaderTypeENTITY = XML_READER_TYPE_ENTITY,
	  LibXMLReaderTypePROCESSING_INSTRUCTION = XML_READER_TYPE_PROCESSING_INSTRUCTION,
	  LibXMLReaderTypeCOMMENT = XML_READER_TYPE_COMMENT,
	  LibXMLReaderTypeDOCUMENT = XML_READER_TYPE_DOCUMENT,
	  LibXMLReaderTypeDOCUMENT_TYPE = XML_READER_TYPE_DOCUMENT_TYPE,
	  LibXMLReaderTypeDOCUMENT_FRAGMENT = XML_READER_TYPE_DOCUMENT_FRAGMENT,
	  LibXMLReaderTypeNOTATION = XML_READER_TYPE_NOTATION,
	  LibXMLReaderTypeWHITESPACE = XML_READER_TYPE_WHITESPACE,
	  LibXMLReaderTypeSIGNIFICANT_WHITESPACE = XML_READER_TYPE_SIGNIFICANT_WHITESPACE,
	  LibXMLReaderTypeEND_ELEMENT = XML_READER_TYPE_END_ELEMENT,
	  LibXMLReaderTypeEND_ENTITY = XML_READER_TYPE_END_ENTITY,
	  LibXMLReaderTypeXML_DECLARATION = XML_READER_TYPE_XML_DECLARATION
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

By moving enumeration to a  ```Sequence``` type, consumers of the DOM don't need to be concerned with how the list of Child Nodes is navigated. 

## Attempt 1: Swift Structure from libxml2 DOM

The first thing that springs to mind is that given a DOM in ```libxml```, some of the Data can be pulled out into a pure Swift data structure[^class-vs-struct]. This Swift data structure will contain all of the relationships and Text Nodes, whilst implementing the previously defined ```XMLParsableType```.

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

The ```childrenOfNode``` function is placed in the ```LibXMLDOM``` class to separate the concerns of how the Tree is built into a data structure and how the DOM is navigated. By using this function that returns a  ```Sequence`` and a number of other wrapper functions (prefixed with ```LibXMLDOM```) returning Swift types, the construction of the tree can be defined recusively:

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

## Attempt 2: Swift Structure from libxml2 Buffered Reader

On reflection, this sounds like the XML Document is built twice. Once to read the document into ```libxml2``` and second to extract out the Text Nodes and Element Names into a Tree. This transformation could potentially be costly so there might be a performance benefit to only building the tree once.

The Reader interface for ```libxml2``` is an interface to the XML document that will incrementally read each part of the document in a depth-first fashion. A Node is read by asking the Reader for the next 

## Attempt 3: Navigate the libxml2 DOM with Swift

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

The ```LibXMLDOM.Context``` is just a internal class responsible for cleaning up the manual-memory-managed ```libxml2``` document when the root node is deallocated:

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

Still quite a simple implementation, that's about all we need to implement the ```XMLParsableType``` interface. The creation of the Swift Strings for the Text of an Element are deferred until they are actually needed. If nearly all of the Text Elements are consumed when mapping to a model it won't have much (if any) impact on performance relative to the first attempt.

## Attempt 4: Navigate with a Reader

Our restricted representation of what XML can be is based purely on the need to access Text Nodes within a Tree. Depending on what we are ignoring in the XML Document, the parser could potentially read a large number of Attributes and ignored Elements into memory that never need to be pulled out.

The DOM parser for ```libxml2``` will have parsed the whole document, elements, attributes and all by the time it is created. Depending on the how much of the document is ignored, we may get significant performance benefit. This will also prove that the cost for parsing an XML document is heavily influenced by the XML Parser costs themselves, not the ```XMLParseableType``` abstraction itself and any composition on top.

## Testing

One of the great aspects of building an XML Parser in this way is that the Implementations are trivial to swap out and can be tested against for correctness against the same Parsing Code. Additionally, we can use the new Performance Testing features of XCode 6 to see what the performance is like for these varying methods.

## Conclusion

The results prove that there can be drastically different performance just by using different methodologies. If you'd like to take a look at the code, it is [available on GitHub]().

The majority of this code was written with an Application in mind. I didn't have long at all (in fact I wasn't even close to having the right amount of time). I was working on an iOS App that would show the fastest train home (not necessarily the next train) to help out commuters decide which train would give them the most time at the office and at home [_(I'm about to become one of these people)_](https://www.google.co.uk/maps/dir/Coventry,+United+Kingdom/London+Euston,+United+Kingdom/@51.9631488,-1.3744911,9z/data=!3m1!4b1!4m14!4m13!1m5!1m1!1s0x48774bae1512f593:0x9881aa5f47d741c1!2m2!1d-1.51345!2d52.400828!1m5!1m1!1s0x48761b25a93c9b83:0xdbced150c2761d8f!2m2!1d-0.133898!2d51.528136!3e3). Regardless, the code is [available on GitHub to pick apart and prod]().

--- 
 
[^dom-performance]: I cut my programming teeth on Java. When consuming an XML Document of greater than a MB or so the JVM Heap could get hammered when using a DOM style parser. This could be hugely problematic when intertwined with a Garbage Collector. I wonder if the ubiquity of JSON Parsers that output to a fully reified Data Structure, rather than Event Based parsers has anything to do with the increased availability of resources since JSON came into vogue.

For this reason the Event-Based Streaming SAX parser could be used instead, the whole document need not be read into memory at the expense of a more troublesome interface. Eventually a compromise was found with the StAX parser, buffering the input document with a Cursorable navigation mechanism.
 
[^class-vs-struct]: This is the perfect candidate for usage of a ```struct``` instead of a ```class```. We don't need for this to be subclassed, or any disposal semantics in ```dealloc```. Reference counting is totally uncessary as this structure represents an Immutable value. I'd love to take a look at the performance of passing structs around, [it looks like Array and Dictionary struct types use copy-on-write](https://devforums.apple.com/message/990361#990361) but I have no idea if this applies to User defined structs. 

[^string-nsstring]: Ok I lied, there I use a lot of Bridging functions that use ```NSString```, but an ```NSString``` is instantly converted to a Swift ```String```
