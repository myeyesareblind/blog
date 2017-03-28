---
layout: post
title: Combining hash values
tags: swift objc hash boost ios osx
---

TLDR: to make a hash of a custom class, that contains few primitive properties with defined `hash` methods - use this
```
- (NSUInteger)hash {
  NSUInteger hash = 0;
  hash = 17 * 37 + self.prop1.hash;
  hash = 17 * 37 + self.prop2.hash;
  return hash;
}
```

## Hashing
`NSDictionary` is a collection that we use everyday. Most of the time we use a native type as a key, something like `NSDictionary<NSString, MyClass>` and everything just works. But sometimes it's required to use a custom object as a key. For example, I needed to store array of events per date-range: `NSDictionary<MyDateRange, NSArray>`, where `MyDateRange` is:
```obj-c
@interface MyDateRange
@property (readwrite) NSDate *start;
@property (readwrite) NSDate *end;
@end
```
When using a custom `NSObject` as a key in dictionary it's required to implement `isEqual:`, `copyWithZone:` and `hash` methods. `isEqual` and `copyWithZone` are easy: we simply compare/copy `NSDate` properties. `hash` is not: we need to combine hashes of 2 `NSDate`.
In this article I want to discuss:
1. The properties that a good hash function must possess.
2. Existing implementations from a few well-known libraries.
3. Performance on artificial test-data.

## 1. Properties of good hash-combine function
`NSDictionary` allows a fast access to an object by a key no matter how many elements are there. To do this it relies on `hash` method. `hash` must be equal for objects that are equal and should be different if they are not equal.
If the `hash` for different objects is equal, it's called a collision and `NSDictionary` will call `isEqual` on all of them to find proper key. More on this topic by [NSHipster](http://nshipster.com/equality/) and [Mike Ash](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html).

That's what documentation says. Unfortunately, it's not that simple in the real world. Internally hash-table stores keys in a so called buckets. The number of buckets is somewhat bigger then number of keys, we can see this in [CF-Foundation sources](https://github.com/opensource-apple/CF/blob/master/CFBasicHash.c). 

| Number of keys | Number of buckets (Actual NSDictionary Size) |
| -------------- | ------------------------ |
| 0              | 0                        |
| 3              | 3                        |
| 6              | 7                        |
| 11             | 13                       |
| 19             | 23                       |
| 32             | 41                       |
| 52             | 71                       |
| 85             | 127                      |
| 118            | 191                      |
| 155            | 251                      |
| 237            | 383                      |
| 390            | 631                      |
| 672            | 1087                     |

and so on ...

We can see few facts: dictionary is filled by about 60% mostly, the actual size is always prime integer.
[There is](http://ciechanowski.me/blog/2014/04/08/exposing-nsdictionary/) a blog that discusses this in more details.
In short: each key must be put into sinlge bucket. We need to calculate index of a bucket based on `hash`. Since `NSUInterMax` - the maximum number of `hash` value, is much bigger than number of buckets, it should be reduced to `[0..number_of_buckets)` somehow. The way it's done is simple: a plain modulo operator
`hash % number_of_buckets` is used.
The more keys have same modulo hash - the more collisions we get. When collision happens, key occupies next empty bucket. This can lead to grouping hashes in some area, which is really bad as we'll see next.

In the next section we will review popular methods to combine hashes.

## 2. Existing implementations

#### Naive way of Hash Building
The first thing that comes to mind is to sum or xor 2 dates: 
```obj-c
- (NSUInteger)hash {
  return start.hash + end.hash
}
```
Well, sum or hash of 2 integers is really single instruction and everything looks good. Except it's not: it produces too many collisions as I'll show later.

#### C++ Boost Hash Combine
Boost is well-known, high quality and efficient c++ library. It also contains [hash combining](http://www.boost.org/doc/libs/1_35_0/doc/html/hash/combine.html). The interesting function is rather simple:
```c++
template <typename SizeT>
inline void hash_combine_impl(SizeT& seed, SizeT value)
{
    seed ^= value + 0x9e3779b9 + (seed<<6) + (seed>>2);
}
```
Boost uses sort of Shift-Add-XOR hash function.

#### Mike Ash
There is an article from [Mike Ash](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html) on this topic as well. He suggests using rotate and xor:
```objc
#define NSUINT_BIT (CHAR_BIT * sizeof(NSUInteger))
#define NSUINTROTATE(val, howmuch) ((((NSUInteger)val) << howmuch) | (((NSUInteger)val) >> (NSUINT_BIT - howmuch)))

- (NSUInteger)hash
{
    return NSUINTROTATE(self.start.hash, NSUINT_BIT / 2) ^ self.end.hash;
}
```

#### Apache.Commons
In Java there is [Apache.Commons.HashCodeBuilder](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/builder/HashCodeBuilder.html). It uses simple prime number multiplication algorithm.
```objc
- (NSUInteger)hash {
    NSUInteger hash = 17;
    hash = hash * 37 + self.start.hash;
    hash = hash * 37 + self.end.hash;
    return hash
}
```

#### CFHashBytes
Most of the Foundation objects override `hash` method. Look:
```obj-c
[NSNumber numberWithInt:1].hash = 2654435761
[NSNumber numberWithInt:2].hash = 5308871522
[[NSDate dateWithTimeIntervalSince1970:0] hash] = 2596853616923779200
[[NSDate dateWithTimeIntervalSince1970:1] hash] = 2596853614269343439
[@"hello_hello_hello_hello" hash] = 12392780783853204868
[@"hello_hello_hello_hello1" hash] = 4243784906404336054
```
Change in 1 bit produces very different result. This all was done to reduce the amount of collisions.

Starting iOS10, there is a `NSDateInterval` class that does just what we need.
Taken from [swift sources](https://github.com/apple/swift/blob/master/stdlib/public/SDK/Foundation/DateInterval.swift):

```swift
public var hashValue: Int {
    var buf: (UInt, UInt) = (UInt(start.timeIntervalSinceReferenceDate), UInt(end.timeIntervalSinceReferenceDate))
    return withUnsafeMutablePointer(to: &buf) {
        $0.withMemoryRebound(to: UInt8.self, capacity: 2 * MemoryLayout<UInt>.size / MemoryLayout<UInt8>.size) {
            return Int(bitPattern: CFHashBytes($0, CFIndex(MemoryLayout<UInt>.size * 2)))
        }
    }
}
```
Under the hood, it uses `CFHashBytes` function. Unfortunately it's a private function in CoreFoundation. We can still find it [open source Apple Core Foundation](https://github.com/opensource-apple/CF/blob/master/CFUtilities.c). This implementation uses ELF hash algorithm.

#### FNV
FNV stands for Fowler/Noll/Vo. Was developed for hashing strings, but can be generalized to hash any byte sequence.
Also used in [Roslyn .NET Compiler](https://github.com/dotnet/roslyn/blob/master/src/Compilers/Core/Portable/InternalUtilities/Hash.cs).
```obj-c
void FNVHashUpdate(NSUInteger* h, NSUInteger value) {
    NSUInteger magic = 1099511628211lu;
    NSUInteger hash = *h;
    unsigned char *p = (unsigned char*)&value;
    for (int i = 0; i < sizeof(NSUInteger); i++) {
        hash = (hash ^ p[i]) * magic;
    }
    *h = hash;
}
```

#### One-at-a-Time Hash
Made by [Bob Jenkins](https://en.wikipedia.org/wiki/Jenkins_hash_function), it's a default hash function in Perl.
I've no idea how did he come up with this, but it's works really good.
```obj-c
void OATHashUpdate(NSUInteger* h, NSUInteger value) {
    unsigned char *p = (unsigned char*)&value;
    NSUInteger hash = *h;

    for (int i = 0; i < sizeof(NSUInteger); i++) {
        hash += p[i];
        hash += (hash << 10);
        hash ^= (hash >> 6);
    }
    *h = hash;
}

void OATHashFinalize(NSUInteger* h) {
    NSUInteger hash = *h;
    hash += (hash << 3);
    hash ^= (hash >> 11);
    hash += (hash << 15);
    *h = hash;
}

- (NSUInteger)hash {
    NSUInteger hash = 0;
    OATHashUpdate(&hash, self.start.hash);
    OATHashUpdate(&hash, self.end.hash);
    OATHashFinalize(&hash);
    return hash
}
```

#### JenkinsHashCombine
The most recent hash function from Bob Jenkins called [SpookyHash](http://burtleburtle.net/bob/hash/spooky.html).
The implementation is really complicated to inline here. [See](https://github.com/andikleen/spooky-c) yourself.


## 3. Performance on artificial test-data
Having all this, it's better to run a quick test to see which one is fastest.
I picked simple and generic data: composed objects that contains 2 random integer values from uniform distribution: 
```
Num_of_samples = 10_000_000
arc4random(num_of_samples)
```

Those objects are put into `NSDictionary` as keys, and we are interested in duration of this operation.

See a [test project](https://github.com/myeyesareblind/HashCombine) for more details.

The values are in seconds. The smaller - the better.
![duration bar-plot]({{ "/assets/img/make_nsdictionary_duration.png" | prepend: site.baseurl }})

`xor` didn't complete in any reasonable time.

Why some functions are better than others? Well, because of collisions. Let's verify this using key distribution plot. 
Since we know the number of buckets, we can simulate construction of `NSDictionary`. For each `hash` we will find an empty index, and for each index we check - increment number of visits. Plot is better then words. We can't plot that much of a data, so I'll use 500 samples. 

![distribution of buckets-plot]({{ "/assets/img/500distributions.png" | prepend: site.baseurl }})

The number of buckets for holding 500 keys is 1087 (according to table of sizes above) - that's X axis on the plot.
Y axes are number of visits in each bucket, which is actual number of `isEqual:` calls.
Good hash function should be evenly distributed in 0..1087 range. 

We can see that `sum` is slightly grouped around center, which makes sense for sum of 2 numbers.
`xor` has maximum at 510 - that's random number upper bound (500) + collisions. `xor` only uses half of all available interval, which leads to huge spikes. Because each collision occupies next empty bucket, number of collisions grow nonlinearly.
Other functions don't have visible peaks and are distributed evenly.

Basically all, but trivial functions, have similar performance. `sum` and `xor` have too many collisions which leads to poor overall performance. Using advanced hash functions as `fnv`, `oat`, `elf` and `jenkins` doesn't really makes sense on typical input. Most of the time functions from Mike Ash, Boost and Apache Commons will suffice. I'll be using Apache version in my current project.

Big thanks [@FalconSer](https://twitter.com/FalconSer) for reading drafts and suggestions.
