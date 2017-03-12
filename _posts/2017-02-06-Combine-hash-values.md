---
layout: post
title: Combining hash values
tags: swift objc hash boost ios osx
---

TLDR: to combine hashes of 2 objects, use this [cocoapod](https://github.com/myeyesareblind/HashCombine)

Another day, when implementing a cache in iOS/OSX app, I used NSSet with objects of type `Message`.
```obj-c
@interface Message: NSObject
@property (readwrite) NSString *content;
@property (readwrite) NSUInteger msgType;
```

When using NSSet with custom objects, it's important to override
`public var hashValue: Int { get }`
(or `-(NSUInteger)hash` for obj-c)
for faster lookup. You probably know [why](http://nshipster.com/equality/).

Most of the time the object has single unique property, which one can use as `hash`, but `Message` doesn't.
`hash` must take into account both properties: `msgType` and `content`. We need to combine them. The easiest thing to do is:
```obj-c
/// DON'T DO THIS
- (NSUInteger)hash {
  return msgType + content.hash
}
```
which would work almost ok. Well, not really. If fields are somewhat correlated - it's a disaster as I'll show later.

My curiosity led me to make some research. The first thing I did is looked in the Swift sources. All-in-all, this must be solved problem and Apple engineers must have a solution. And they do:

```swift
/// taken from implementation of NSData
public var hashValue: Int {
    switch _backing {
    case .customReference(let d):
        return d.hash
    case .customMutableReference(let d):
        return d.hash
    default:
        let len = _length
        return Int(bitPattern: CFHashBytes(_bytes?.assumingMemoryBound(to: UInt8.self), Swift.min(len, 80)))
    }
}
```

The interesting function in CFHashBytes. Unfortunately it's a private function in CoreFoundation. We can still find it [here]
(https://github.com/opensource-apple/CF/blob/master/CFUtilities.c). This implementation uses ELF hash algorithm.
A great introduction for this hash function and few others can be found [here](http://eternallyconfuzzled.com/tuts/algorithms/jsw_tut_hashing.aspx#elf). Go read it, it's great.

Another source of inspiration is boost. Boost has everything, and [hash combining](http://www.boost.org/doc/libs/1_35_0/doc/html/hash/combine.html) is not an exception. The interesting function is quite simple:
```c++
template <typename SizeT>
inline void hash_combine_impl(SizeT& seed, SizeT value)
{
    seed ^= value + 0x9e3779b9 + (seed<<6) + (seed>>2);
}
```

Boost uses sort of Shift-Add-XOR hash function.

There is an article from [Mike Ash](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html) on this topic as well. Which suggests using rotate and xor:
```objc
#define NSUINT_BIT (CHAR_BIT * sizeof(NSUInteger))
#define NSUINTROTATE(val, howmuch) ((((NSUInteger)val) << howmuch) | (((NSUInteger)val) >> (NSUINT_BIT - howmuch)))

- (NSUInteger)hash
{
    return NSUINTROTATE(self.content.hash, NSUINT_BIT / 2) ^ self.msgType;
}
```


[Apache.Commons.HashCodeBuilder](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/builder/HashCodeBuilder.html) suggests using prime integer summation and multiplication:
```objc
void PrimeIntegerHashCombine(NSUInteger* hashCombine, NSUInteger value) {
    *hashCombine = *hashCombine * 37 + value;
}
- (NSUInteger)hash {
    NSUInteger hash = 17;
    hash = hash * 37 + self.content.hash;
    hash = hash * 37 + self.msgType;
    return hash
}
```
    


Having all this, it's better to run a quick test to see which one is fastest. The ideal function must be calculated very fast and generate no collisions at all.

To keep things simple, I only tried simplest functions:

* `sum`: `hash1 + hash2`
* `xor`: `hash1 ^ hash2`
* [`mike-ash`](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html)
* [`prime-integer-multiply`](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/builder/HashCodeBuilder.html)
* [`boost`](http://www.boost.org/doc/libs/1_35_0/doc/html/hash/combine.html)
* [`oat`](http://eternallyconfuzzled.com/tuts/algorithms/jsw_tut_hashing.aspx)
* [`fnv`](http://eternallyconfuzzled.com/tuts/algorithms/jsw_tut_hashing.aspx)

I picked simple data: composed objects that consist of 
* integer[1..1000000] and rnd(10)
* rnd(1000000) and rnd(1000000)
* UUID and UUID
* String(i) for i = 1...1000000

See a (test project)[https://github.com/myeyesareblind/HashCombine] for more details.

One thing to note: NSNumber has a smart hash method. Look:
```obj-c
[NSNumber numberWithInt:1].hash = 2654435761
[NSNumber numberWithInt:2].hash = 5308871522
```
Combining using `NSNumber.hash` will give much better results.

And these are result, the values are in seconds. The smaller - the better.

Hash-Func     | int, rnd(10) | rnd, rnd | UUID, UUID | Str(i), Str(i) | total
------------- | ------------ | -------- | ---------- | -------------- | -----
sum           | 9.15         | 17.88    | 14.71      | 78.85          | 120.59
xor           | 7.28         | ∞        | 14.20      | ∞              | ∞
mike-ash      | 5.56         | 6.07     | 10.96      | 8.45           | 31.04
prime-int-mul | 4.27         | 6.15     | 10.98      | 7.11           | 28.5
boost         | 4.61         | 6.12     | 10.87      | 7.96           | 29.5
oat           | 5.50         | 6.05     | 11.02      | 8.44           | 31
fnv           | 4.71         | 6.05     | 11.81      | 7.88           | 30.45

Another useful insight is how many collisions did the function generate. The smaller - the better.

Hash-Func     | int, rnd(10) | rnd, rnd | UUID, UUID | Str(i), Str(i) | total
------------- | ------------ | -------- | ---------- | -------------- | -----
sum           | 3487593      | 2640790  | 0          | 0              | 6127498
xor           | 6512749      | 2818959  | 0          | 9999999        | 19331707
mike-ash      | 0            | 0        | 0          | 784989         | 784989
prime-int-mu  | 0            | 132252   | 0          | 0              | 132252
boost         | 13036        | 78221    | 0          | 0              | 91257
oat           | 0            | 0        | 0          | 0              | 0
fnv           | 0            | 0        | 0          | 0              | 0

* `∞` means I couldn't wait so long. `xor` on correlated fields is a bad idea.
* `sum` is 4 times slower then any other function on 2 strings input. I am not sure why that happened, there is really no reason for it. Ignoring this outlier it's still much slower due to high number of collisions.
* `xor` failed on 2 rnd and 2 strings input. Not sure what is wrong with rnd, rnd. 2 strings failed because there is only 1 hash value for all of them - 0, since 2 hashes negate each other.
* `mike-ash` did well, but somehow on 2 strings generated a lot of collision, which affected the result.
* `prime-int-mul` is the fastest on this input. It requires just a few instructions and generated 1.3% of collisions only on 2 random integers input.
* `boost` is like `prime-int-mul`: it requires few instructions, but generates more collisions.
* `oat` and `fnv` don't have any collisions, but are slow to calculate.

Basically all, but trivial functions, have similar performance. `sum` and `xor` are very fast to calculate, but have too many collisions which leads to poor overall performance. 

Since `prime-int-mul` was the fastest, I made a [cocoapod](https://github.com/myeyesareblind/HashCombine) for it.

Happy hashing.
