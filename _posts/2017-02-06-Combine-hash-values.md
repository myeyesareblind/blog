---
layout: post
title: Combining hash values
tags: swift objc hash boost ios osx
---

TLDR: to combine hashes of 2 objects, use this [cocoapod](https://github.com/myeyesareblind/HashCombine)

Another day, when implementing a cache in iOS/OSX app, I used NSSet with objects of type `Message`.
```swift
class Message {
    let msgType: Int
    let content: String
}
```

When using NSSet with custom objects, it's important to override
`public var hashValue: Int { get }`
(or `-(NSUInteger)hash` for obj-c)
for faster lookup. You probably know [why](http://nshipster.com/equality/).

Most of the time the object has single unique property, which one can use as `hashValue`, but `Message` doesn't.
`hashValue` must take into account both properties: `msgType` and `content`. We need to combine them. The easiest thing to do is:
```swift
/// DON'T DO THIS
public var hashValue {
    return msgType.hashValue + content.hashValue
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

Also, search on cocoapods revealed these 2 projects:
https://github.com/irace/BRYHashCodeBuilder
https://github.com/levigroker/HashBuilder
There is an article from Mike Ash on this topic as well:
https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html

Having all this, it's better to run a quick test to see which one works best.
I picked simple data: combination of integers, integer and string and 2 string with various combinations.
The number of data samples is `1000000`.


See a (test project)[https://github.com/myeyesareblind/HashCombine] for more details.
To keep things simple, I only tried simplest functions:

* `sum`
* `xor`
* `string-concat`
* `mike-ash`
* `prime-integer-multiply`
* `boost`
* `oat`
* `fnv`

And results are curious:

Hash-Func     | 2 same int    | 2 reverse int | int and small rnd |   2 rnd   | 2 strings | time
------------- | ------------- | ------------- | ----------------- | --------- | --------- | ----
sum           | 1000000       | 1             | 651062            | 734929    | 1000000   | 8.01646101474762
xor           | 1             | 482054        | 999970            | 999999    | 1         | 3.582207024097443
string-concat | 1000000       | 999998        | 1000000           | 1000000   | 1000000   | 29.69862800836563
mike-ash      | 999973        | 1000000       | 1000000           | 1000000   | 952245    | 9.646197021007538
prime-int-mul | 1000000       | 1000000       | 1000000           | 986641    | 1000000   | 7.852792024612427
boost         | 1000000       | 1000000       | 1000000           | 1000000   | 1000000   | 9.370872020721436
oat           | 1000000       | 1000000       | 1000000           | 1000000   | 1000000   | 11.10435301065445
fnv           | 1000000       | 1000000       | 1000000           | 1000000   | 1000000   | 11.40823596715927

One thing to notice is that `sum` and `xor` work much better then expected. This is because NSNumber has a smart hash method. Look:
```obj-c
[NSNumber numberWithInt:1].hash = 2654435761
[NSNumber numberWithInt:2].hash = 5308871522
```
Even a small change produces completely different hash, which leads to good distribution. But on a corelated data, they fall miserably.

`String-concat` works great, but is 3x time slowly than any other.

The next functions are more complicated, but they produce much better result.
`mike-ash` and `prime-int-mul` don't provide perfect hash for this input and collision number is noticable. They are still much-much better then trivial `sum` or `xor` and should be fine for most data.

`boost`, `oat` and `fnv` provide perfect hash on this input. This comes with no surprise - those algorithms are proven and wildly used. `boost` is the most effective, but it's missing 64bit constant and it's also said to fail avalanche test.
`oat` or `fnv` is a coin-flip, both are perfect. Since I have to choose one - I pick `oat`, because it's implementation is simpler.

I've made a [cocoapod](https://github.com/myeyesareblind/HashCombine) for it and a pleasant swift version.

See you next time and happy hashing.
