# fxa_unordered
Proof of concept of closed- and open-addressing unordered associative containers.
* [Development Plan for Boost.Unordered](https://pdimov.github.io/articles/unordered_dev_plan.html)
* [Closed addressing](#closed-addressing)
  * [`fca_unordered_set`, `fca_unordered_map`](#fca_unordered)
  * [`fca_simple_unordered_set`, `fca_simple_unordered_map`](#fca_simple_unordered)
* [Open addressing](#open-addressing)
  * [`foa_unordered_coalesced_set`, `foa_unordered_coalesced_map`](#foa_unordered_coalesced)
  * [`foa_unordered_nway_set`, `foa_unordered_nway_map`](#foa_unordered_nway)
  * [`foa_unordered_nwayplus_set`, `foa_unordered_nwayplus_map`](#foa_unordered_nwayplus)
  * [`foa_unordered_hopscotch_set`, `foa_unordered_hopscotch_map`](#foa_unordered_hopscotch)
  * [`foa_unordered_longhop_set`, `foa_unordered_longhop_map`](#foa_unordered_longhop)
* [Benchmark results](https://github.com/joaquintides/fca_unordered/actions) for this PoC

## Closed addressing
<a name="fca_unordered"></a>

```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>,
  typename SizePolicy=prime_size,typename BucketArrayPolicy=grouped_buckets,
  typename NodeAllocationPolicy=dynamic_node_allocation
>
class fca_unordered_set;

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator</* equivalent to std::pair<const Key,Value> */>,
  typename SizePolicy=prime_size,typename BucketArrayPolicy=grouped_buckets,
  typename NodeAllocationPolicy=dynamic_node_allocation
>
class fca_unordered_map;
```

**`SizePolicy`**

Specifies how the bucket array grows and the algorithm used for determining the position
of an element with hash value `h` in the array.
* `prime_size`: Sizes are prime numbers in an approximately doubling sequence. `position(h) = h % size`,
modulo operations are sped up by keeping a function pointer table with `fpt[i](h) == h % size(i)`,
where `i` ranges over the size sequence.
* `prime_switch_size`: Same as before, but instead of a table a `switch` statement over `i` is used.
* `prime_fmod_size`: `position(h) = fastmod(high32bits(h) + low32bits(h), size)`.
[fastmod](https://github.com/lemire/fastmod) is a fast implementation of modulo for 32-bit numbers
by Daniel Lemire.
* `prime_frng_size`: `position(h) = fastrange(h, size)`. Daniel Lemire's [fastrange](https://github.com/lemire/fastrange)
maps a uniform distribution of values in the range `[0, size)`. This policy is ill behaved for
low quality hash functions because it ignores the low bits of the hash value.
* `prime_frng_fib_size`: Fixes pathological situations of `prime_frng_size` by doing
`positon(h) = fastrange(h * F, size)`, where `F` is the
[Fibonacci hashing](https://en.wikipedia.org/wiki/Hash_function#Fibonacci_hashing) constant.
* `pow2_size`: Sizes are consecutive powers of two. `position(h)` returns the higher bits of the
hash value, which, as it happens with `prime_frng_size`, works poorly for low quality hash functions.
* `pow2_fib_size`: `h` is Fibonacci hashed before calculating the position.

**`BucketArrayPolicy`**
* `simple_buckets`: The bucket array is a plain vector of node pointers without additional metadata.
The resulting container deviates from the C++ standard requirements for unordered associative
containers in three aspects:
  * Iterator increment is not constant but gets slower as the number of empty buckets grow;
    see [N2023](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2023.pdf) for details.
  * Because of the former, `erase(iterator)` returns `void` instead of an iterator to the next
    element.
  * `begin()` is not constant time (hopping to the first occupied bucket is required).
<div style="list-style-type: none;margin-left: 40px;">
  <ul>This policy is used to simulate <code>unordered_bucket_map</code> as specified in the
    <a href="https://pdimov.github.io/articles/unordered_dev_plan.html">Development Plan for Boost.Unordered</a>.
  </ul>
</div>

* `bcached_simple_buckets`: Same as `simple_buckets`, but a reference to the first occupied bucket
is kept and updated so as to provide constant-time `begin()`.
* `grouped_buckets`: The resulting container is fully standards-compliant, including constant
iteration and average constant `erasure(iterator)`. Besides the usual bucket array, a vector
of *bucket group* metadata is kept. Buckets are logically grouped in 32/64 (depending on the
architecture) consecutive buckets: the associated bucket group metadata consists of a pointer
to the first bucket of the group, a bitmask signalling which buckets are occupied,
and `prev` and `next` pointers to link non-empty bucket groups in a bidirectional list.
Going from a given bucket to the next occupied one is implemented as follows:
  * Use [bit counting](https://www.boost.org/libs/core/doc/html/core/bit.html) operations to
    determine, in constant time, the following occupied bucket within the group.
  * If there are no further occupied buckets in the group, go the `next` group (which is
    guaranteed to be non-empty).
<div style="list-style-type: none;margin-left: 40px;">
  <ul>The memory overhead added by bucket groups is 4 bits per bucket.</ul>
</div>

**`NodeAllocationPolicy`**
* `dynamic_node_allocation`: Nodes are allocated individually.
* `hybrid_node_allocation`: Buckets are extended to hold space for a node. When inserting
a new value in a bucket, that bucket and its three neighbors to the right are checked for
available embedded space to hold the node; if no space is found, dynamic allocation is used.
The resulting container deviates in a number of important aspects from the C++ standard
requirements for unordered associative containers:
  * Pointer stability is not mantained on rehashing.
  * The elements of the container must be movable.
  * It is not possible to provide [node extraction](https://en.cppreference.com/w/cpp/container/node_handle)
  capabilities.
* `linear_node_allocation`: Nodes are preallocated in a linear array and selected with
quadratic probing using an occupancy bitmask. Same deviations from the C++ standard as
`hybrid_node_allocation`.
* `pool_node_allocation`: Nodes are preallocated in a linear array and selected
incrementally as requested (with recycling of erased nodes). Same deviations from the
C++ standard as `hybrid_node_allocation`.
* `embedded_node_allocation`: Nodes are embedded into the buckets like in
`hybrid_node_allocation`, but no dynamic allocation happens ever: selection is done
through quadratic probing using the same technique as `linear_node_allocation`.

<a name="fca_simple_unordered"></a>

```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>
>
class fca_simple_unordered_set;

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator</* equivalent to std::pair<const Key,Value> */>
>
class fca_simple_unordered_map;
```

Abandoned experiment where individual occupied buckets where linked in a bidirectional
list. Outperformed by `fca_unordered_[set|map]` with `grouped_buckets`.

## Open addressing
These containers, in general, deviate in a number of important aspects from the C++ standard
requirements for unordered associative containers:
* Pointer stability is not mantained on rehashing.
* The elements of the container must be movable.
* It is not possible to provide [node extraction](https://en.cppreference.com/w/cpp/container/node_handle)
capabilities.
* Iterator increment is not constant but gets slower as the number of non-occupied nodes grow.
* Because of the former, `erase(iterator)` returns `void` instead of an iterator to the next
  element.
* `begin()` is not constant time (hopping to the first occupied node is required).

<a name="foa_unordered_coalesced"></a>
```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>,
  typename SizePolicy=prime_size,typename NodePolicy=simple_coalesced_set_nodes
>
class foa_unordered_coalesced_set;

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator</* equivalent to std::pair<const Key,Value> */>,
  typename SizePolicy=prime_size,typename NodePolicy=simple_coalesced_set_nodes
>
class foa_unordered_coalesced_map;
```

Containers based on [coalesced hashing](https://en.wikipedia.org/wiki/Coalesced_hashing)
The implementation follows [Vitter's original formulation](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.119.6552&rep=rep1&type=pdf)
with some innovations in the area of node recycling:
* A *cellar* (extra node reservoir) is provided with an address factor
(nodes reachable by hash divided by total number of nodes allocated) *β* = 0.86.
* Insertion into a chain happens after the initial element or, if cellar nodes are traversed,
after the last cellar node in the chain (Vitter's VICH Varied-Insertion Coalesced Hashing
algorithm).
* Nodes of erased elements are unlinked and recycled if they are not at the beginning
of a (sub)chain. In particular, erased cellar nodes are always recycled.

In addition to the general deviations from the standard incurred by open-addressing
containers in this PoC, `foa_unordered_coalesced_set`/`foa_unordered_coalesced_map`
have the following:
* `erase(iterator)` is implemented as `erase(*iterator)`: this will throw if `Hash` or `Pred` do,
but allows for node recycling.
* Rehashing does not happen when the number of elements in the container hits the maximum load,
but when the number of *used* nodes do; for instance, erasing an element at the beginning of
its chain won't get its node recycled, so in this case the number of used nodes does not
decrease. This behavior makes it hard to predict exactly when rehashing will occur in the
presence of erasures.
  
**`SizePolicy`**

As with [`fca_unordered_set`/`fca_unordered_map`](#fca_unordered).

**`NodePolicy`**
* `simple_coalesced_set_nodes`: Regular nodes are used with storage for the value and a `next` pointer.
* `hcached_coalesced_set_nodes`: Nodes are extended to keep the hash value of the element; hash
comparison is used to rule out non-matches on lookup without invoking the equality predicate,
which can potentially speed up the process.

<a name="foa_unordered_nway"></a>
```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>,
  typename SizePolicy=prime_size
>
class foa_unordered_nway_set;

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator</* equivalent to std::pair<const Key,Value> */>,
  typename SizePolicy=prime_size
class foa_unordered_nway_map;
```

Slots are logically divided in groups of size 16. Slots are probed *only* within
the target group looking for a reduced 7-bit hash value with vectorized
byte operations (using [SSE2](https://en.wikipedia.org/wiki/SSE2) when available):
if the group is full, extra nodes are allocated and kept in a singly linked list,
where each group has its own list.

**`SizePolicy`**

As with [`fca_unordered_set`/`fca_unordered_map`](#fca_unordered).

<a name="foa_unordered_nwayplus"></a>
```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>,
  typename SizePolicy=prime_size,
  typename GroupAllocationPolicy=regular_allocation
>
class foa_unordered_nwayplus_set 

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator<map_value_adaptor<Key,Value>>,
  typename SizePolicy=prime_size,
  typename GroupAllocationPolicy=regular_allocation
>
class foa_unordered_nwayplus_map;
```

Slots are logically divided in groups of size 16. On first attempt, slots
are probed within the target group looking for a reduced 7-bit hash value
with vectorized byte operations
(using [SSE2](https://en.wikipedia.org/wiki/SSE2) when available); if this
first group is unsuccessful (key not found, group full), additional groups
are tried according to the rules encoded by `GroupAllocationPolicy`.

**`SizePolicy`**

As with [`fca_unordered_set`/`fca_unordered_map`](#fca_unordered).
N-way<sup>+</sup> maps are very sensitive to the hash function
[uniformity](https://en.wikipedia.org/wiki/Hash_function#Uniformity), which
makes policies using Fibonacci hashing (or other bit spreading techniques)
the only viable alternatives (unless a really good hash function is provided
in the first place).

**`GroupAllocationPolicy`**
* `regular_allocation`: N-groups are probed quadratically.
*  `soa_allocation`: N-group metadata and the associated slots
are kept in separate arrays to improve cache locality. Quadratic probing
is used.
* `coalesced_allocation`: Borrowing ideas from coalesced hashing, a portion
of the array (the cellar) is kept out of hash-based positioning and
used when a regular N-group is full; N-groups are then linked via a `next`
pointer. When the cellar is exhausted, quadratic probing is resorted to.
`coalesced_allocation` depends on the following parameters:
  * *β* = 0.86, ratio of regular N-groups to total groups.
  * maximum saturation = 4: When a new cellar N-group is requested, the
  previously used one is re-used as long as its occupancy level (saturation)
  is below a predetermined threshold, which implies that several regular
  N-groups can be linked to the same cellar location. This approach strikes
  a balance between average chain length and cellar efficiency (the occupancy
  level of cellar N-groups).
* `soa_coalesced_allocation`: As `coalesced_allocation`, but group metadata and
slots are kept in separate arrays.

<a name="foa_unordered_hopscotch"></a>
```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>,
  typename SizePolicy=prime_size
>
class foa_unordered_hopscotch_set;

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator<map_value_adaptor<Key,Value>>,
  typename SizePolicy=prime_size
>
class foa_unordered_hopscotch_map;
```

Containers based on [hopscotch hashing](http://mcg.cs.tau.ac.il/papers/disc2008-hopscotch.pdf).
We introduce the following modifications with respect to the approach described
in chapter 2 of the referenced paper:
* The original formulation assumed that element slots can be checked for
occupancy/emptiness, which requires that a special value of `Key` is
designated as meaning "empty". To avoid imposing this requirement, we
maintain a separate array of control metadata bytes keeping an occupancy
bit and a reduced 7-bit hash value.
* As checking for matching elements for a bucket can be done very
efficiently using [SSE2](https://en.wikipedia.org/wiki/SSE2) on the control array,
bucket *hop-information bitmaps*, as defined in the original paper, are not strictly
needed and have been replaced by an array of *delta values*
(distances from the elements to their base buckets), which perform better for
the hopscotch algorithm and occupy less memory (4 bits per slot for a
bucket maximum length of 16).

In addition to the general deviations from the standard incurred by
open-addressing containers in this PoC,
`foa_unordered_hopscotch_set`/`foa_unordered_hopscotch_map` have the following:
* Rehashing happens when the number of elements in the container hits the
maximum load, or, before that, if the hopscotch algorithm is blocked
(no empty slot can be produced in the vicinity of a bucket).
* If hopscotch block happens *during or immediately after* rehashing,
a `hopscotch_failure` exception is thrown.

**`SizePolicy`**

As with [`fca_unordered_set`/`fca_unordered_map`](#fca_unordered).
Hopscotch maps are very sensitive to the hash function
[uniformity](https://en.wikipedia.org/wiki/Hash_function#Uniformity), which
makes policies using Fibonacci hashing (or other bit spreading techniques)
the only viable alternatives (unless a really good hash function is provided
in the first place).

<a name="foa_unordered_longhop"></a>
```cpp
template<
  typename T,typename Hash=boost::hash<T>,typename Pred=std::equal_to<T>,
  typename Allocator=std::allocator<T>,
  typename SizePolicy=prime_size
>
class foa_unordered_longhop_set;

template<
  typename Key,typename Value,
  typename Hash=boost::hash<Key>,typename Pred=std::equal_to<Key>,
  typename Allocator=std::allocator<map_value_adaptor<Key,Value>>,
  typename SizePolicy=prime_size
>
class foa_unordered_longhop_map;
```

Further variation on [hopscotch hashing](http://mcg.cs.tau.ac.il/papers/disc2008-hopscotch.pdf):
* Instead of using hop-information bitmaps or deltas to the beginning
of the bucket, slots are linked via 4-bit values `first` and `next` acting
as "short" pointers to the first and next element of the bucket, respectively.
This allows for buckets to be of indefinite length, as opposed to
[`foa_unordered_hopscotch_set`/`foa_unordered_hopscotch_map`](#foa_unordered_hopscotch),
where the maximum size is 16.
* `first`/`next` info is conflated with an occupancy bit and a 7-bit reduced
hash value in a 2-byte metadata word per slot kept in a separate array.
This layout prevents the usage of [SSE2](https://en.wikipedia.org/wiki/SSE2)
for quick match checking.

In addition to the general deviations from the standard incurred by
open-addressing containers in this PoC,
`foa_unordered_longhop_set`/`foa_unordered_longhop_map` have the following:
* Rehashing happens when the number of elements in the container hits the
maximum load, or, before that, if the hopscotch algorithm is blocked
(no empty slot can be produced in the vicinity of the last bucket element).
* If hopscotch block happens *during or immediately after* rehashing,
a `hopscotch_failure` exception is thrown.
* On erasure, elements in the same bucket beyond that being erased are
moved towards the beginning of the bucket to occupy the slot left
available. This means that the `erase(it++)` idiom won't work for
erasure on container traversal: instead, one has to write `it=erase(it)`
as would be done with, say, a `std::vector`. As a consequence of
`erase` returning an iterator, its complexity is not average constant
but suffers from the problem described in
[N2023](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2023.pdf).

**`SizePolicy`**

As with [`fca_unordered_set`/`fca_unordered_map`](#fca_unordered).
Hopscotch maps are very sensitive to the hash function
[uniformity](https://en.wikipedia.org/wiki/Hash_function#Uniformity), which
makes policies using Fibonacci hashing (or other bit spreading techniques)
the only viable alternatives (unless a really good hash function is provided
in the first place).
