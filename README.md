RoaringBitmap
=============

[![Build Status](https://travis-ci.org/RoaringBitmap/RoaringBitmap.png)](https://travis-ci.org/RoaringBitmap/RoaringBitmap)
[![][maven img]][maven]
[![][license img]][license]
[![docs-badge][]][docs]
[![Coverage Status](https://coveralls.io/repos/github/RoaringBitmap/RoaringBitmap/badge.svg?branch=master)](https://coveralls.io/github/RoaringBitmap/RoaringBitmap?branch=master)

Bitsets, also called bitmaps, are commonly used as fast data structures.
Unfortunately, they can use too much memory. To compensate, we often use
compressed bitmaps.

Roaring bitmaps are compressed bitmaps which tend to outperform conventional
compressed bitmaps such as WAH, EWAH or Concise. In some instances, roaring bitmaps can
be hundreds of times faster and they often offer significantly better compression.
They can even be faster than uncompressed bitmaps.

This library is used by
*   [Apache Spark](http://spark.apache.org/),
*   [Apache Kylin](http://kylin.io),
*   [Netflix Atlas](https://github.com/Netflix/atlas),
*   [LinkedIn Pinot](https://github.com/linkedin/pinot/wiki) and
*   [Druid](http://druid.io/).


Apache Lucene (http://lucene.apache.org/) uses  Roaring bitmaps, though they have their own [independent implementation](https://svn.apache.org/viewvc/lucene/dev/branches/branch_5x/lucene/core/src/java/org/apache/lucene/util/RoaringDocIdSet.java?view=markup&pathrev=1629606). Derivatives of Lucene such as Solr and Elastic also use Roaring bitmaps.
Other platforms such as Whoosh (https://pypi.python.org/pypi/Whoosh/) also use Roaring bitmaps with
their own implementations.


There is a serialized format specification for interoperability between implementations: https://github.com/RoaringBitmap/RoaringFormatSpec/


(c) 2013-2016 the RoaringBitmap authors


This code is licensed under Apache License, Version 2.0 (ASL2.0).



When should you use a bitmap?
===================================


Sets are a fundamental abstraction in
software. They can be implemented in various
ways, as hash sets, as trees, and so forth.
In databases and search engines, sets are often an integral
part of indexes. For example, we may need to maintain a set
of all documents or rows  (represented by numerical identifier)
that satisfy some property. Besides adding or removing
elements from the set, we need fast functions
to compute the intersection, the union, the difference between sets, and so on.


To implement a set
of integers, a particularly appealing strategy is the
bitmap (also called bitset or bit vector). Using n bits,
we can represent any set made of the integers from the range
[0,n): it suffices to set the ith bit is set to one if integer i is present in the set.
Commodity processors use words of W=32 or W=64 bits. By combining many such words, we can
support large values of n. Intersections, unions and differences can then be implemented
 as bitwise AND, OR and ANDNOT operations.
More complicated set functions can also be implemented as bitwise operations.

When the bitset approach is applicable, it can be orders of
magnitude faster than other possible implementation of a set (e.g., as a hash set)
while using several times less memory.


When should you use compressed bitmaps?
===================================

An uncompress BitSet can use a lot of memory. For example, if you take a BitSet
and set the bit at position 1,000,000 to true and you have just over 100kB. That's over 100kB
to store the position of one bit. This is wasteful  even if you do not care about memory:
suppose that you need to compute the intersection between this BitSet and another one
that has a bit at position 1,000,001 to true, then you need to go through all these zeroes,
whether you like it or not. That can become very wasteful.

This being said, there are definitively cases where attempting to use compressed bitmaps is wasteful.
For example, if you have a small universe size. E.g., your bitmaps represent sets of integers
from [0,n) where n is small (e.g., n=64 or n=128). If you are able to uncompressed BitSet and
it does not blow up your memory usage,  then compressed bitmaps are probably not useful
to you. In fact, if you do not need compression, then a BitSet offers remarkable speed.


How does Roaring compares with the alternatives?
==================================================


Most alternatives to Roaring are part of a larger family of compressed bitmaps that are run-length-encoded
bitmaps. They identify long runs of 1s or 0s and they represent them with a marker word.
If you have a local mix of 1s and 0, you use an uncompressed word.

There are many formats in this family:

* Oracle's BBC is an obsolete format at this point: though it may provide good compression,
it is likely much slower than more recent alternatives due to excessive branching.
* WAH is a patented variation on BBC that provides better performance.
* Concise is a variation on the patented WAH. It some specific instances, it can compress
much better than WAH (up to 2x better), but it is generally slower.
* EWAH is both free of patent, and it is faster than all the above. On the downside, it
does not compress quite as well. It is faster because it allows some form of "skipping"
over uncompressed words. So though none of these formats are great at random access, EWAH
is better than the alternatives.



There is a big problem with these formats however that can hurt you badly in some cases: there is no random access. If you want to check whether a given value is present in the set, you have to start from the beginning and "uncompress" the whole thing. This means that if you want to intersect a big set with a large set, you still have to uncompress the whole big set in the worst case...

Roaring solves this problem. It works in the following manner. It divides the data into chunks of 2<sup>16</sup> integers
(e.g., [0, 2<sup>16</sup>), [2<sup>16</sup>, 2 x 2<sup>16</sup>), ...). Within a chunk, it can use an uncompressed bitmap, a simple list of integers,
or a list of runs. Whatever format it uses, they all allow you to check for the present of any one value quickly
(e.g., with a binary search). The net result is that Roaring can compute many operations much faster that run-length-encoded
formats like WAH, EWAH, Concise... Maybe surprisingly, Roaring also generally offers better compression ratios.



API docs
---------

http://www.javadoc.io/doc/org.roaringbitmap/RoaringBitmap/

Scientific Documentation
--------------------------

-  Samy Chambi, Daniel Lemire, Owen Kaser, Robert Godin,
Better bitmap performance with Roaring bitmaps,
Software: Practice and Experience Volume 46, Issue 5, pages 709???719, May 2016
http://arxiv.org/abs/1402.6407 This paper used data from http://lemire.me/data/realroaring2014.html
- Daniel Lemire, Gregory Ssi-Yan-Kai, Owen Kaser, Consistently faster and smaller compressed bitmaps with Roaring, Software: Practice and Experience (accepted in 2016, to appear) http://arxiv.org/abs/1603.06549
- Samy Chambi, Daniel Lemire, Robert Godin, Kamel Boukhalfa, Charles Allen, Fangjin Yang, Optimizing Druid with Roaring bitmaps, IDEAS 2016, 2016. http://r-libre.teluq.ca/950/

Code sample
-------------

```java        
import org.roaringbitmap.RoaringBitmap;

public class Basic {

  public static void main(String[] args) {
        RoaringBitmap rr = RoaringBitmap.bitmapOf(1,2,3,1000);
        RoaringBitmap rr2 = new RoaringBitmap();
        rr2.add(4000L,4255L);

        RoaringBitmap rror = RoaringBitmap.or(rr, rr2);// new bitmap
        rr.or(rr2); //in-place computation
        boolean equals = rror.equals(rr);// true
        if(!equals) throw new RuntimeException("bug");
        // number of values stored?
        long cardinality = rr.getLongCardinality();
        System.out.println(cardinality);
        // a "forEach" is faster than this loop, but a loop is possible:
        for(int i : rr) {
          System.out.println(i);
        }
  }
}
```

Please see the examples folder for more examples.


Unsigned integers
------------------

Java lacks native unsigned integers, but integers are still considered to be unsigned within Roaring, and ordered  according to ??``Integer.compareUnsigned``. This means that Java will order the numbers like so 0, 1, ..., 2147483647, -2147483648, -2147483647,..., -1. To intepret correctly, you can use ``Integer.toUnsignedLong`` and ``Integer.toUnsignedString``.


Working with memory-mapped bitmaps
---------------------------------------

If you want to have your bitmaps lie in memory-mapped files, you can
use the org.roaringbitmap.buffer package instead.

The following code sample illustrates how to create an ImmutableRoaringBitmap
from a ByteBuffer. In such instances, the constructor only loads the meta-data
in RAM while the actual data is accessed from the ByteBuffer on demand.

```java
        import org.roaringbitmap.buffer.*;

        //...

        MutableRoaringBitmap rr1 = MutableRoaringBitmap.bitmapOf(1, 2, 3, 1000);
        MutableRoaringBitmap rr2 = MutableRoaringBitmap.bitmapOf( 2, 3, 1010);
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(bos);
        // If there were runs of consecutive values, you could
        // call rr1.runOptimize(); or rr2.runOptimize(); to improve compression
        rr1.serialize(dos);
        rr2.serialize(dos);
        dos.close();
        ByteBuffer bb = ByteBuffer.wrap(bos.toByteArray());
        ImmutableRoaringBitmap rrback1 = new ImmutableRoaringBitmap(bb);
        bb.position(bb.position() + rrback1.serializedSizeInBytes());
        ImmutableRoaringBitmap rrback2 = new ImmutableRoaringBitmap(bb);
```

Operations on an ImmutableRoaringBitmap such as and, or, xor, flip, will
generate a RoaringBitmap which lies in RAM. As the name suggest, the
ImmutableRoaringBitmap itself cannot be modified.

This design was inspired by Druid.

One can find a complete working example in the test file TestMemoryMapping.java.

Note that you should not mix the classes from the org.roaringbitmap package with the classes
from the org.roaringbitmap.buffer package. They are incompatible. They serialize to the same output however.




Prerequisites
-------------

 - Version 0.6.x requires JDK 7 or better
 - Version 0.5.x requires JDK 6 or better

To build the project you need maven (version 3).


Download
---------

You can download releases from the Maven repository:
http://central.maven.org/maven2/org/roaringbitmap/RoaringBitmap/

or from github:
https://github.com/RoaringBitmap/RoaringBitmap/releases

Maven repository
----------------
If your project depends on roaring, you  can  specify the dependency in the Maven "pom.xml" file:

```xml
        <dependencies>
          <dependency>
            <groupId>org.roaringbitmap</groupId>
            <artifactId>RoaringBitmap</artifactId>
            <version>[0.6,)</version>
          </dependency>
        </dependencies>
```

where you should replace the version number by the version you require.

Usage
------

* Get java
* Get maven 3

* ``mvn compile`` will compile
* ``mvn test`` will run the basic unit tests
* ``mvn package`` will package in a jar (found in target)
* ``mvn checkstyle:check`` will check that you abide by the code style
*  To run our complete testing routine (it takes a long time), execute ``mvn clean test && mvn clean install -DskipTests -Dgpg.skip=true && mvn -f real-roaring-dataset/pom.xml clean install && mvn -f ./jmh/pom.xml test``. Be warned that our testing is very thorough.

A convenient command to build the code is :

             mvn clean install -DskipTests -Dgpg.skip=true

Contributing
------------

Contributions are invited. We enforce the Google Java style.
Please run  ``mvn checkstyle:check`` on your code before submitting
a patch.

FAQ
----


* I am getting an error about a bad cookie. What is this about?

In the serialized files, part of the first 4 bytes are dedicated to a "cookie"
which serves to indicate the file format.

If you try to deserialize or map a bitmap from data that has an
unrecognized "cookie", the code will abort the process and report
an error.

This problem will occur to all users who serialized Roaring bitmaps
using versions prior to 0.4.x as they upgrade to version 0.4.x or better.
These users need to refresh their serialized bitmaps.



* How big can a Roaring bitmap get?


Given N integers in [0,x), then the serialized size in bytes of
a Roaring bitmap should never exceed this bound:

`` 8 + 9 * ((long)x+65535)/65536 + 2 * N ``

That is, given a fixed overhead for the universe size (x), Roaring
bitmaps never use more than 2 bytes per integer. You can call
``RoaringBitmap.maximumSerializedSize`` for a more precise estimate.

Benchmark
-----------

To run JMH benchmarks, use the following command:

         $ ./jmh/run.sh

You can also run specific benchmarks...

         $ ./jmh/run.sh org.roaringbitmap.aggregation.and.identical.*

To run memory benchmarks, use the following command:

         $ ./memory/run.sh


Mailing list/discussion group
-----------------------------

https://groups.google.com/forum/#!forum/roaring-bitmaps

Funding
----------

This work was supported by NSERC grant number 26143.



[maven img]:https://maven-badges.herokuapp.com/maven-central/org.roaringbitmap/RoaringBitmap/badge.svg
[maven]:http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.roaringbitmap%22%20

[license]:LICENSE-2.0.txt
[license img]:https://img.shields.io/badge/License-Apache%202-blue.svg

[docs-badge]:https://img.shields.io/badge/API-docs-blue.svg?style=flat-square
[docs]:http://www.javadoc.io/doc/org.roaringbitmap/RoaringBitmap/
