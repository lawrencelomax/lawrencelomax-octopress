---
layout: post
title: "Swift Parsers - Libraries"
date: 2014-09-12 22:30:03 +0100
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

In the previous posts in this series the Parsers have used a hypothetical ```XMLParsableType``` Protocol for fetching data from an XML Element. I previously mentioned that there are myriad libraries and methods for parsing an XML document for iOS. All of the code in this article and covered in previous articles [is available on GitHub](http://github.com/lawrencelomax/XMLParsable).

In this post we're going to take a look at two things: 
1. How ```XMLParsableType``` can be implemented using ```libxml2```. 
2. How the various interfaces of ```libxml2``` and abstractions in Swift can result in drastically variant performance.

## Parser Interfaces

The immensely popular Open Source project [libxml2](http://xmlsoft.org) is a C Framework bundled with iOS and Mac OS X for parsing XML[^nsxmlparser], with no external dependencies. For our interests it has implementations for a number of [XML Reading interfaces](http://en.wikipedia.org/wiki/Java_API_for_XML_Processing). These are the [DOM](http://en.wikipedia.org/wiki/Document_Object_Model), [SAX](http://en.wikipedia.org/wiki/Simple_API_for_XML) and [Reader](http://xmlsoft.org/xmlreader.html) interfaces.

If I were to build these interfaces in an Application I'd want to be cautious to not prematurely optimise the XML Document reading process. Even if there are any substantial gains to XML Parsing performance when building implementations of the ```XMLParsableType```, the gains may not be that significant to the Application as a whole. If the implementations were to be made from scratch it would be worth writing one simple implementation and the swapping it with a more performant implementation if-and-when it is seen to be impacting the performance of the Application at large. However, this is a fun exercise to play with Swift and Xcode 6 as well as getting to grips with the ```libxml2``` library.

### DOM

A [DOM (Document Object Model) or Tree Interface](http://en.wikipedia.org/wiki/Document_Object_Model) to XML represents all of the Elements within an XML Document as a tree of connected nodes. This often means that the whole of an XML Document is read into memory and is navigable by following relationships in a data structure. By exposing the XML Document as a fully realized data structure, a DOM Interface very convenient to navigate.

By loading all parts of the XML Document into memory the memory usage is proportional to the size of the XML Document. This will be a low cost for small documents, but for larger documents this can result in a substantial amount of memory to allocate[^dom-performance]. 

The performance of a DOM parser relative to others is largely dependent on the number of times that the same DOM is traversed and the amount of ignored data in the DOM. If the DOM is repeatedly traversed, then the up-front cost of building the tree will amortise later navigation of the data structure. However, if there are only a few Elements and Text nodes that need to be extracted from a large document, the resources used to read the unread parts of the tree are wasted.
	
### SAX

As a DOM interface requires that the whole Tree of Elements is in memory an interface to XML exists that streams data out of the document on a per-node basis. The [SAX](http://en.wikipedia.org/wiki/Simple_API_for_XML) interface traverses all of the nodes in sequence, allowing data to be extracted from a stream of callbacks. This means that the entirety of the document doesn't have to be in memory in order to read a document. This can be very desirable in resource constrained environments, or when the document to be parsed is very large.

Unfortunately, this comes with drawbacks. The immediate concern for this article is that a callback based interface can be inconvenient as it results in temporary state being created for book-keeping purposes. A data-structure approach may be able to store all of the bookkeeping information in local variables and function arguments, resulting in simpler implementations that use the recursion and the call stack instead of allocated data structures.

### Reader

The [Reader Interface](http://xmlsoft.org/xmlreader.html) is very similar to the SAX interface, with a cursor based way of traversing the tree, rather than callbacks. This can be thought of as the difference between a push and pull driven stream of data. This makes the interface significantly easier to work with than SAX.

## Swift Implementation

With a little context to the types of parsing interfaces that are available in ```libxml2```, it help us to understand how they can be used in Swift. All of the implementations of the interface will return a [```Result<XMLParsableType>```](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsable/XML/XMLNavigatingParser.swift#L12) of the root node in the XML Document, with the XML Document read from either a [File or an ```NSData``` representation of document String](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsable/XML/XMLNavigatingParser.swift#L13).

### C Structures in Swift

[```libxml2```](http://xmlsoft.org)  Open Source implementation of is a C library, the C functions can be imported into Swift using an Bridging Header and used as if they are Swift functions with [usual rules of interacting with C API](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-XID_11). As Working with the structures requires quite a bit of unwrapping and manipulating 'Unsafe' containers that wrap all C structures in Swift. Again, a strong reason to move up to the Swift level of abstraction.

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

These functions can then be used in Swift when they are exposed in an [Umbrella or Bridging Header](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html).

### GOTO Swift

I like to follow a principle when developing software, regardless of the languages and frameworks used:

> _"Get to the highest level of Abstraction that makes sense, as early as possible. Don't think about the overhead of the abstraction until you can prove that it is detrimental to performance"_

```libxml2``` sits very far down the ladder of Abstraction, so I consider it to be ideal to pull data from ```libxml2``` into a higher level of abstraction as soon as possible. This makes manipulating and reading XML data in Swift significantly easier. Additionally, this means skipping Objective-C[^string-nsstring] and using elements of the [Swift Standard Library](http://swifter.natecook.com) along with bridged Cocoa libraries to get stuff done. Swift provides us with some abstractions that are superior to the Objective-C equivalents, there's no need for an intermediate stage unless it is needed. The Strong guarantees that Swift provides us are also present in the components of the Standard Library.

Conveniently, Xcode will parse headers with enumerations declared with the ```NS_ENUM``` macro:

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

This is a redeclaration of libxml enum values, [with a naming convention that will play nice with Swift](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-XID_13):

	switch LibXMLDOMGetElementType(child) {
	case .ELEMENT_NODE:
		children.append(LibXMLNodeParserDOM.createTreeRecursive(child))
	case .TEXT_NODE:
		text = LibXMLDOMGetText(child)
	default:
		break
	}

Swift provides the [```SequenceType```](http://swifter.natecook.com/protocol/SequenceType/) and [```GeneratorType```](http://swifter.natecook.com/protocol/GeneratorType/) protocols to allow us to use the ```for ... in ...``` syntax, or use the higher-level [```map```](http://swifter.natecook.com/func/map/), [```find```](http://swifter.natecook.com/func/find/) and [```filter```](http://swifter.natecook.com/func/filter/) functions. We can define a ```Sequence``` type that exposes the Child nodes of an XML Node:

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

## Implementation 1: Swift Structure from libxml2 DOM

The first thing that springs to mind is that given a DOM in ```libxml```, some of the data can be pulled out into a pure Swift data structure[^class-vs-struct]. This Swift data structure will contain all of the relationships and Text Nodes, whilst implementing the previously defined ```XMLParsableType```.

	public struct XMLNode: XMLParsableType {
		public let name: String
		public let text: String?
		public let children: [XMLNode]
	
		public func parseChildren(elementName: String) -> [XMLNode] {
			return self.children.filter { node in
				return node.name == elementName
			}
		}
		
		public func parseText() -> String? {
			return self.text
		}
	}

The ```childrenOfNode``` function is placed in the ```LibXMLDOM``` class to separate the concerns of how the Tree is built into a data structure and how the DOM is navigated. By using this function that returns a  ```Sequence`` and a number of other wrapper functions (prefixed with ```LibXMLDOM```) returning Swift types, the construction of the tree can be defined recursively:

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

## Implementation 2: Swift Structure from libxml2 Buffered Reader

Looking at the first this means that the XML Document is built twice. Once to read the document into ```libxml2``` and the second to extract out the Text Nodes and Element Names into a Tree of Swift data structures. The costs of allocating a bunch of Swift structures and objects could be significant so there will likely be a performance benefit to only building the tree once.

[The Reader interface for ```libxml2```](http://xmlsoft.org/xmlreader.html) is an interface to the XML document that will incrementally read each component of the document in a depth-first fashion. The Current Node is repeatedly advanced by calling the [```xmlTextReaderRead```](http://xmlsoft.org/html/libxml-xmlreader.html#xmlTextReaderRead) function until the end of the file has been reached, or an Error has occurred in parsing. Using the Reader, a Swift Data structure can be built without having to create a DOM in ```libxml2```. It is possible however, that the overhead associated by advancing through the ```Reader``` buffer could be greater than the cost of allocating the DOM in ```libxml2```

A Lazy ```Sequence``` of nodes can be made for the [Reader interface](http://xmlsoft.org/xmlreader.html) in the same way as the [Tree interface](http://xmlsoft.org/html/libxml-tree.html). The ```Sequence``` is made from the more fundamental ```Generator```:

	internal typealias ReaderGenerator = GeneratorOf<(xmlTextReaderPtr, LibXMLReaderMode)>
	
	private class func makeRecursiveGenerator(reader: xmlTextReaderPtr) -> ReaderGenerator {
		return GeneratorOf {
			if reader == nil {
				return nil
			}
		
			let result = LibXMLReaderMode.forceMake(xmlTextReaderRead(reader))
			switch result {
			case .CLOSED: return nil
			case .EOF: return nil
			case .ERROR: return nil
			case .INITIAL: return nil
			default: return (reader, result)
			}
		}
	}

With the Reader approach, an error can occur at any time and therefore needs to be handled during the reading process. This is passed back in a tuple of both the pointer to the current node and the status of the read operation. It is then wrapped in a Sequence for the Higher-Order operations and the pointer must be advanced one time on the first read:

	internal typealias ReaderSequence = SequenceOf<(xmlTextReaderPtr, LibXMLReaderMode)>
	private class func wrapInSequence(reader: xmlTextReaderPtr) -> Result<ReaderSequence> {
		if (reader == nil) {
			return Result.Error(self.error("Could Not Parse Document, no root node"))
		}
	
		var generator = LibXMLReader.makeRecursiveGenerator(reader)
		if generator.next() == nil {
			xmlFreeTextReader(reader)
			return Result.Error(self.error("Could Not Parse Document, no first node"))
		}
	
		return Result.value(SequenceOf(generator))
	}

In the ```LibXMLReader``` class, a ```Context``` class is used as a container to hide the details of the interface between Swift and ```libxml```. This is all wrapped up in a structure[^struct-multiple-return] containing a Reader pointer and a Sequence which will advance the Reader as it is consumed. It also has a ```dispose``` method for resource cleanup of the manually memory managed ```libxml``` structures. Note that there should only be one ```Context``` per reader to avoid multiple freeing of the Reader:

	internal final class LibXMLReader {
		internal struct Context {
			let reader: xmlTextReaderPtr
			let sequence: ReaderSequence
			var hasFreed: Bool
			
			init (reader: xmlTextReaderPtr, sequence: ReaderSequence) {
				self.reader = reader
				self.sequence = sequence
				self.hasFreed = false
			}
			
			func dispose() {
				if !self.hasFreed {
					xmlFreeTextReader(self.reader)
				}
			}
	}

The Tree can be built up using a Recursive function, passing through the ```ReaderSequence```

	private class func parseRecursive(reader: xmlTextReaderPtr, _ sequence: ReaderSequence) -> Result<XMLNode> {
		if LibXMLReaderType.forceMake(reader) != .ELEMENT {
			let realType = LibXMLReaderGetElementTypeString(LibXMLReaderType.forceMake(reader))
			return Result.Error(LibXMLReader.error("Recursive Node Parse Requires an ELEMENT, got \(realType)"))
		}
		
		var name = LibXMLReaderGetName(reader)
		var text: String?
		var children: [XMLNode] = []
		
		if LibXMLReaderIsEmpty(reader) {
			return Result.value(XMLNode(name: name, text: text, children: children))
		}
		
		for (reader, mode) in sequence {
			let type = LibXMLReaderType.forceMake(xmlTextReaderNodeType(reader))
			let typeString = LibXMLReaderGetElementTypeString(type)
			let modeString = LibXMLReaderGetReaderMode(mode)
		
			switch type {
				case .TEXT:
					text = LibXMLReaderGetText(reader);
				case .ELEMENT:
					let childResult = self.parseRecursive(reader, sequence)
					switch childResult {
						case .Error(let error): return childResult
						case .Value(let box): children.append(childResult.toOptional()!)
					}
				case .END_ELEMENT:
					assert(name == LibXMLReaderGetName(reader), "BEGIN AND END NOT MATCHED")
					return Result.value(XMLNode(name: name, text: text, children: children))
				default:
					break
			}
		}
		
		return Result.Error(LibXMLReader.error("I don't know how this became exhausted, unbalanced begin and end of document"))
	}

There's a little more in terms of book-keeping as the End and the Beginning of a nested element need to be matched up, but it should look very similar to the DOM based approach.

## Implementation 3: Navigate the libxml2 DOM with Swift

The Building of the tree as a fully realized Swift data structure is a simple implementation and all the work is done up front, but it certainly isn't the least resource intensive. In the previous implementation ```String```s for the Text and Element Name of an XML Element are created regardless of whether they are used or not. Even if the use of the ```Reader``` interface results in a faster implementation than the dual-tree interface of the first Implementation, redundancy in data that is read could mean a lot of wasted effort. 

There's no reason that we can't just wrap a whole ```libxml2``` Tree structure in a Class implementing ```XMLParsableType``` that knows how to fetch the Text and Children of an Element from a current Node Pointer:

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

The ```LibXMLDOM.Context``` is just a internal class responsible for cleaning up the manual-memory-managed ```libxml2``` DOM document when the root node is deallocated. Again, there is only one ```Context``` per Reader:

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

Still very simple and that's about all that is required to implement ```XMLParsableType``` protocol. The creation of the Swift ```String```s for the Text of an Element are deferred until they are actually needed.

## Implementation 4: Navigate with a Reader

In Implementation 2, a ```Reader``` was used to navigate the XML Document and extract all of the Elements and Text Nodes as a Swift Data structure. Instead of building a ```Swift``` data structure, it should be possible to extract out Text from ```Reader``` without needing a secondary data structure at all. This should work well in circumstances where the amount of data extracted from the whole document is minimal.

This will result in a substantially different implementation to the previous three and will be worth implementing at a later date. It is currently unimplemented in the [```XMLParsable```](http://github.com/lawrencelomax/XMLParsable) project.

## Testing

One of the great aspects of building an XML Parser in this way is that the implementations are trivial to swap out and can be tested against for correctness against the same Parsing Code. Additionally, we can use the new Performance Testing features of Xcode 6 to see how the performance varies in these differing implementations. The code is all available on [GitHub for your viewing pleasure](http://github.com/lawrencelomax/XMLParsable).

The correctness testing is relatively simple, just run the same suite of tests over a fixed XML resource and verify that all of the properties in the Model are set to the values in the resource.

{% img right /images/posts/swift_parsers/unit_results.png "Unit Test Results: Great Success" "Unit Test Results: Great Success" %}

Xcode 6 makes performance testing easy too. The same XML parse to Model decode task can be stuck inside a [measurement block](https://developer.apple.com/library/prerelease/ios/documentation/DeveloperTools/Conceptual/testing_with_xcode/testing_3_writing_test_classes/testing_3_writing_test_classes.html#//apple_ref/doc/uid/TP40014132-CH4-SW8) and repeated a number of times to eliminate any performance fluctuations of the test host. The Performance Tests should be designed in such a way that they can expose some the performance characteristics of each implementation with a different data set. One of these characteristics is that the amount of unused or redundant data in a document can massively impact performance.

The small suite of performance tests in [```XMLParsable```](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsableTests/Performance/XMLPerformanceTests.swift) test the same source data on each of the implementations on a number of axis, using [```NSData```](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsableTests/Fixtures/Fixtures.swift#L25) vs. [using Files](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsableTests/Fixtures/Fixtures.swift#L21) and a [small XML file](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsableTests/Fixtures/zoo.xml) vs. an [XML file with a lot of redundant data](https://github.com/lawrencelomax/XMLParsable/blob/master/XMLParsableTests/Fixtures/zoo_highredundancy.xml):

{% img right /images/posts/swift_parsers/performance_results.png "Performance Results" "Performance Results" %}

Some observations from the results:

- There isn't a huge difference between ```NSData``` and reading from a file. This suggests that File I/O isn't much of a bottleneck. There won't be any noticeable difference between decoding XML from a File, or from ```NSData``` returned from a HTTP request.  
- When a small XML document is used, there's not much of a performance difference between all of the implementations. There may be some other bottlenecks worth identifying in the system, but the time taken to run 10 iterations is very small. 
- The two implementations that build a Tree of Nodes in Swift have order-of-magnitude worse performance than the DOM-wrapper in the XML document with lots of redundant data. This highlights the differences in performance between building a pure Swift data structure and keeping the data in ```libxml``` itself. This doesn't make Swift slow by any means, it just highlights the tradeoff of abstractions & safety against raw performance. Moving pointers around a region of purpose-allocated memory will always be faster than a runtime that checks types, reference counts and dereferences pointers across discretely allocated data structures.

## Conclusion

The results prove that the performance characteristics of different implementations of an XML Parser can be exposed when using different data sets. The small gap in performance between implementations widens as the size of the document increases. This is likely due to the cost of object and structure allocations. Reducing allocations in ```libxml2``` or Swift will squeezes out more performance.

I'd probably opt to use 'Implementation 3', which wraps the original ```libxml``` DOM in the ```XMLParsableType``` protocol. It is by far the simplest and has very good performance. Even when resources are at a premium it can be better to opt for the simpler solution, even if it means sacrificing a little performance. Maintainability and understandability are important qualities in code, so that the code can be revisited and revised by others and understandable for years to come. As 'Implementation 3' is the most performant and simplest to implement, hard to argue that it is the one to go for!

I hope you've enjoyed this series of posts, I'd love to hear your thoughts and comments! Have a look at the [```XMLParsable``` project and run the tests for yourself!](https://github.com/lawrencelomax/XMLParsable) I'm [@insertjokehere on Twitter](https://twitter.com/insertjokehere/) & [lawrencelomax on GitHub](https://github.com/lawrencelomax).

--- 

[^nsxmlparser]: [NSXMLParser](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/nsxmlparser_class/reference/reference.html) has been part of Cocoa for many years now and would also be suitable, however it only implements an event-based ```SAX```-style interface. While it is certainly possible to make a ```XMLParserType``` implementation using NSXMLParser, I won't consider it for now.

[^dom-performance]: I remember all of this from my early programming days in Java. When consuming an XML Document of greater than a MB or so the JVM Heap could get hammered when using a DOM style parser. This could be hugely problematic when intertwined with a Garbage Collector. I wonder if the ubiquity of JSON Parsers that output to a fully reified Data Structure, rather than Event Based parsers has anything to do with the increased availability of resources since JSON came into vogue.
 
[^class-vs-struct]: This is the perfect candidate for usage of a ```struct``` instead of a ```class```. We don't need for this to be subclassed, or any disposal semantics in ```dealloc```. Reference counting is totally unnecessary as this structure represents an Immutable value. I'd love to take a look at the performance of passing structs around, [it looks like Array and Dictionary struct types use copy-on-write](https://devforums.apple.com/message/990361#990361) but I have no idea if this applies to User defined structs. [Andy Matuschak](http://www.objc.io/issue-16/swift-classes-vs-structs.html) has a great overview of the differences and ideal Use Cases for ```structs```.

[^struct-multiple-return]: This could also be put in a tuple as it is essentially just used for multiple-return in a method. However, creating inner or standalone classes/structures in Swift is so easy to do that it can give some information about the relationship between the returned data.

[^string-nsstring]: Ok I lied, there I use a lot of Bridging functions that use ```NSString```, but an ```NSString``` is instantly converted to a Swift ```String```
