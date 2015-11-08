---
layout:     post
title:      URL URN URI
date:       2015-05-15 16:11
summary:    URL URN URI
categories: Web-Basic
---

这个问题虽然不是什么大的问题但是一直困惑着我： URL URN URI 到底有什么鬼关系？

像追溯这个问题，发现在网上盲目地找已经没有什么意义了。 所以干脆直接去 W3C 看原文。

原来这么长时间来大家嘴里的 URL URI URN 中说纷纭的主要原因是20年的发展中大家对这几个名词的看法发生了一些转变

## 1 经典分类 (Classic View)

> During the early years of discussion of web identifiers (early to mid 90s), people assumed that an identifer type would be cast into one of two (or possibly more) classes. An identifier might specify the location of a resource (a URL) or its name (a URN) independent of location. Thus a URI was either a URL or a URN. There was discussion about generalizing this by addition of a discrete number of additional classes; for example, a URI might point to metadata rather than the resource itself, in which case the URI would be a URC (citation). URI space was thus viewed as partitioned into subspaces: URL and URN, and additional subspaces, to be defined. The only such additional space ever proposed was URC and there never was any buy-in; so without loss of generality it's reasonable to say that URI space was thought to be partitioned into two classes: URL and URN. Thus for example, "http:" was a URL scheme, and "isbn:" would (someday) be a URN scheme. Any new scheme would be cast into one or the other of these two classes.

上面是 W3C 原文 是最最开始的时候提出这三个概念的时候的初衷 在那个 协议没有这么多的年代，人们曾经想过将 URI 分成两个大类： URL 和 URN

举个栗子，当时他们曾经提出过 

* `http:` 开头的 uri (表示某个资源在某个地址中) 就应该是 URL 
* `isbn:` 开头的 uri （表示某个资源的名称) 就应给被归类为 URN

只不过这样的定义会印出来一些问题，那就是这两个大类不够用了：所以曾经想要引用出来 URC （Citation）这种 uri 分类。

再简单点说，其实就是根据 URL 我们可以找到一个资源的位置，而 URN 只是告诉我们资源的名字但是没有告诉我们去哪里找这个资源

## 2 现时分类 (Contemporary View)

> Over time, the importance of this additional level of hierarchy seemed to lessen; the view became that an individual scheme does not need to be cast into one of a discrete set of URI types such as "URL", "URN", "URC", etc. Web-identifer schemes are in general URI schemes; a given URI scheme may define subspaces. Thus "http:" is a URI scheme. "urn:" is also a URI scheme; it defines subspaces, called "namespaces". For example, the set of URNs of the form "urn:isbn:n-nn-nnnnnn-n" is a URN namespace. ("isbn" is an URN namespace identifier. It is not a "URN scheme" nor a "URI scheme").

> Further according to the contemporary view, the term "URL" does not refer to a formal partition of URI space; rather, URL is a useful but informal concept: a URL is a type of URI that identifies a resource via a representation of its primary access mechanism (e.g., its network "location"), rather than by some other attributes it may have. Thus as we noted, "http:" is a URI scheme. An http URI is a URL. The phrase "URL scheme" is now used infrequently, usually to refer to some subclass of URI schemes which exclude URNs.

很长的一段引用 不过大概的意思是这样的： 就是说这个世界上按照常理来说只应该存在两种东西： URI 和 URN，URL是一个对于 `http` 作为 URI scheme 的 URI 的非正式简称

对于 URN 来说 `urn:` 作为一个 uri scheme 之后可以对于这个 scheme 做一个 子命名空间的声明 `urn:isbn:nn-nnnn-nn` 就是一个例子。 这条 uri 中 isbn 不作为一个 URI scheme 出现，而是作为了一个 `urn:` shceme 下的 namespace identifyer。 

总结一下就是现在的看法就是不把 uri 严格分类成 location 或者 name，而是统一地将他们看成是 URI

## 3 RFC 解释

> A URI can be further classified as a locator, a name, or both.  The term "Uniform Resource Locator" (URL) refers to the subset of URIs that, in addition to identifying a resource, provide a means of locating the resource by describing its primary access mechanism (e.g., its network "location").  The term "Uniform Resource Name"(URN) has been used historically to refer to both URIs under the "urn" scheme [RFC2141], which are required to remain globally unique and persistent even when the resource ceases to exist or becomes unavailable, and to any other URI with the properties of a name.

> An individual scheme does not have to be classified as being just one of "name" or "locator".  Instances of URIs from any given scheme may have the characteristics of names or locators or both, often depending on the persistence and care in the assignment of identifiers by the naming authority, rather than on any quality of the scheme.  Future specifications and related documentation should use the general term "URI" rather than the more restrictive terms "URL" and "URN" [RFC3305].

实际上也是如此， rfc3986 已经明确声明之后的所有技术文档中应该尽量避免 `url` 的声明 而是使用 `uri`

## 4 URI 语法

``` text
foo://example.com:8042/over/there?name=ferret#nose
 \_/  \______________/\_________/ \_________/ \__/
  |           |            |            |        |
scheme     authority       path        query   fragment
  |   _____________________|__
 / \ /                        \
 urn:example:animal:ferret:nose
```

可以看到一条 uri 传统意义上的 url 还是跟 urn 有很大的区别的

对于一条 urn 来说，它的语法是这样的

``` text
urn:<NID>:<NSS>
```

nid: namespace identifier
nss: namespace specific stirng

也就是说一般的 URN 是不包括 query string 和 fragement 和这两个东西的

而对于 `authority` 这个属性，对于 url 来说它一般代表一个域名，对于 urn 来说它代表一系列的 namespace identifier 并且用 `:` 隔开

举个栗子就是

``` text
urn:oasis:names:specification:docbook:dtd:xml:4.1.2 // urn
```

## 5 总结

1. URL 都是 uri
2. URI 还包括了 URN
3. URN 是资源的名字，命民空间用 `:` 隔开
4. URN 不足以让我们找到这个资源
5. URL 是可以让我们去某一个地方找到某一条资源的地址

参考: 

* [W3C-uri-clarification](http://www.w3.org/TR/uri-clarification/)
* [RFC3986](http://tools.ietf.org/html/rfc3986)