@title  Distributed Hash Tables
@date   2023-08-10
@author beka
@tags   distributed hash table, dht, storage, search, query

# Distributed Hash Tables

Distributed Hash Tables are arguably the most important and most widely used p2p tool around. They form the basis of a huge number of widely used systems, most famous being BitTorrent, but less famous is IPFS, less so still is Freenet. The basic idea of a DHT is to be a hash table that is distributed among many different peers on the network in some way so that no one peer is responsible for the entirety of the hash table, and possible also so that no one peer is the only source of some data.

## Undistributed Hash Tables

Before we look at DHTs, however, it's useful to actually review the plain old undistributed sort of hash table, because it actually provides a lot of useful context for why DHTs are the way they are. So then, what is a a hash table?

Hash tables are a specific implementation choice for a key-value map or dictionary data structure. Many people are familiar with these abstractions from languages like Python, where they show up as `dict`s, or from JavaScript, where they show up as `object`s. The idea of a map is that they somehow associate a key with a value in a unique way. For any given key in the map, requesting the key returns the unique value associated with it. These can be implemented in many different ways. For instance, the most naive but straight-forward implementation is probably to treat a map as a list of key-value pairs, and looking up a key is done by simply iterating over the list until the first matching key is found, and then returning the associated value. But this kind of implementation is inefficient, because the lookup time is linear in the number of key-value pairs in the list. Each pair has to be inspected before you know if you've found the right one, so you have to possibly traverse the whole list.

There are many different ways to address this problem, for instance by sorting the list to enable logarithmic search over its elements, but the one that is most relevant here is the hash table. We know that if we have a list with constant time look up, such as an array, then for any index in the bounds of the array, we can look up the associated element in a single step, without iterating over the whole list. In a sense, then, an array is a dictionary that has integers as keys. The idea of a hash table is to take advantage of this fact and generalize it beyond just integer keys.

To do this generalization, we make use of something called a hash function or hashing function. Suppose that we want to use strings as keys in our implementation of the dictionary. If we had some magic function `h` which could turn an arbitrary string into an integer, then we could associate any key `k` with the _array index_ `h(k)`, and store the associated value in that position of the array. In fact, there are many ways to do this. We could for instance take the length of the string. Or we could sum up the ASCII character codes of the characters in the string. Or treat the string as a bit string and then treat that as an integer. There are many options. If the array didn't have a key `k`, it's associated position  `h(k)` would be empty, unless `h(k)` is larger than the largest key that _is_ in the array, in which case we would just not allocate any memory to it at all.

Here's an example. Suppose that we have three keys `"foo"`, `"bar"`, and `"baz"`, which hash to `2`, `5`, and `7`, respectively. No particular hash function is being used here, it's just for demonstration purposes. Now let's suppose that the array stores the value `"x"` for the key `"foo"`, the value `42` for the key `"bar"`, and the value `True` for the key `"baz"`. The memory representation of the proto hash table would look like thus:

| Index | Value       |
|-------|-------------|
| 0     | `undefined` |
| 1     | `undefined` |
| 2     | `"x"`       |
| 3     | `undefined` |
| 4     | `undefined` |
| 5     | `42`        |
| 6     | `undefined` |
| 7     | `True`      |

Now clearly this is suboptimal, because we've only got three elements and most of the array is empty. Depending on the hashing function we choose, this might be extremely common. For instance, let's consider the last one above as a reference point: `h(k)` is the value of `k` seen as a bit-string seen as an integer. In general, this is going to be a very large number. Any single character could be as high as a hundred and some, and that's ignoring fancier encodings beyond ASCII. Two characters will be in the tens of thousands, and so on. Each additional character multiplies the typical scale of the number by around 256. That would mean that the array would have to have at least that many elements to store values for keys that long, and since there are generally going to be quite few keys compared to the total _possible_ keys that the array is defined over, the vast vast majority of the array is going to be empty.

Ideally, we'd like to keep the size of the array as small as possible, and the number of undefined elements as small as possible too. How we handle resizing the array is neither here nor there, let's just suppose for now that the array is a fixed size of 8, as it is above. As we add elements to the proto hash table, eventually, and indeed very quickly, we'd insert something outside the 0-7 range of the indices. In fact, with the string-as-bitstring-as-integer approach, pretty much every key will be at an index well outside the range. And for any relatively small size, this will generally be true. We need something better than just string-as-bitstring-as-integer. So here's one idea: mod the number by 8, that way every key is in the array bounds. Ok, so now we're assuming instead that in the above example, `h("foo") = 2` is true not because `"foo"` seen as a bitstring as a integer is itself 2, but rather because when we mod that integer with 8, _then_ we get 2. I don't know if that's exactly right but there's a 1/8 chance it is so we'll go with it.

But now we have another problem, which is that there's a fairly high chance that two keys will collide. That is to say, since `"foo"` hashes to 2, any other string might hash to 2 as well, with a 1/8 probability. So how do we deal with that? The usual answer is to not store the values directly in the array at their hash index, but rather to store a plain old list of key-value pairs there, called a bin or bucket, where the keys for that list are all and only the keys that hash to that hash index. So the above hash table would look like

| Index | Value               |
|-------|---------------------|
| 0     | `[]`                |
| 1     | `]]`                |
| 2     | `[ ("foo", "x") ]`  |
| 3     | `[]`                |
| 4     | `[]`                |
| 5     | `[ ("bar", 42) ]`   |
| 6     | `[]`                |
| 7     | `[ ("baz", True) ]` |

If we then suppose that `"quux"` also hashes to 2, and we stored `"blorp"` at that key, the resulting hash table would be

| Index | Value                                 |
|-------|---------------------------------------|
| 0     | `[]`                                  |
| 1     | `]]`                                  |
| 2     | `[ ("foo", "x"), ("quux", "blorp") ]` |
| 3     | `[]`                                  |
| 4     | `[]`                                  |
| 5     | `[ ("bar", 42) ]`                     |
| 6     | `[]`                                  |
| 7     | `[ ("baz", True) ]`                   |

Further refinements can be performed here as well. For instance, we could use a fancier structure than a list of key-value pairs in each of the slots, or we could use another hash table with a different hashing mechanism, or all sorts of things. Eventually we have to bottom out to something like a list of key-value pairs tho, typically sorted to again allow logarithmic search. The final crucial aspect of this, to maintain constant time lookup, is that the bins have some maximum possible size. If any bin needs to have an element inserted into it and it's full, then we resize the _whole_ array, and re-insert items into the new larger array putting them in their new bins.

Now, the hashing function used above isn't a very good one probably, because there are likely to be regularities in the input. For instance, if the keys are typically words of English, there are very different statistical distributions of words in across the alphabet, and that might lead to hotspots in the keyspace. So what we really want for a hash table is a hashing function that has good randomization properties. A great candidate for this is cryptographic hashing functions such as SHA1 or SHA2.

## Distributed Hash Tables

With the undistributed hash table out of the way, we can now turn to the distributed sort. Suppose that we have a p2p network where every peer has some sort of identity. For instance, it could be their IP address, or a public key, or something to that effect. All of these can be inputs to a hash function, possibly with some trivial conversions to make it compatible depending on the hash function, and so both the peer IDs and the potential hash table keys could, in fact, be mapped to the same value space consisting of whatever the hash function returns. Let's pick SHA-2 256 (aka SHA-256), which hashes byte strings of arbitrary length into byte strings of 256 bits. SHA-2 is a good hashing function that has all of the desired properties, including vanishingly small likelihood of collision, and currently there are no known collisions at all.

Then, we can play the same game we did with undistributed hash tables, but rather than using an array, we can store the values at the peers who's hashed IDs match the hashed keys. Except that by itself won't work, because even if there are billions of peers on the network, that's only billions of hashed peer IDs, out of a total possible number of 2<sup>256</sup>. The keyspace is vanishing sparse in terms of peers, and so most keys won't correspond to any actual peers. So instead, we don't store the key-value pairs at the peer who's hashed ID is _exactly_ equal to the hashed key, but rather at the peer who's hashed ID is _closest_ according to some distance metric. Also, since SHA-256 has such a low collision probability, it's also a common move to not bother storing the unhashed key there as well, because hashed and unhashed keys are effectively in 1-to-1 correspondence. Instead, the target peer simply stores the hashed key together with its value.

## Distance Metrics

The biggest, most obvious variaton among different DHTs is in the distance metrics they use. A distance metric is just a function that takes two inputs of whatever sort is being measured, and maps them to some value, typically a number of some sort, which is itself totally ordered in a familiar way. The standard definition of a metric function is that the value returned is a real number, but of course computers don't really have those, so floats might be used, but for the purposes of DHTs, it can even suffice to use integers of an appropriate size.

A distance metrics `d`, to be a proper mathematical metric, must have the following properties:

1. The distance from an element to itself is 0, i.e. for all `x`, `d(x,x) = 0`
2. Distances are positive, i.e. for all `x` and `y`, `d(x,y) >= 0`
3. Distances are non-directional, i.e. for all `x` and `y`, `d(x,y) = d(y,x)`
4. Going from one point to another and then a third is never shorter than going directly from first to third, i.e. for all `x`, `y`, and `z`, `d(x,y) + d(x,z) >= d(x,z)`

This isn't always strictly true for the distance metrics used in DHTs, however. For example, Chord's distance metric is asymmetric, so the direction matters. In practice, this is fine, we just need the distance metric to let us have some notion of "closer", so we can sort peers.

### The Linear Metric

The simplest distance metric we can give for the output of SHA-256 is to treat the resulting bitstrings as 256-bit integers and do standard numerical subtraction, taking the absolute value of the difference. This is a terrible choice in practice, but it's simple and convenient for explanations, so let's look at an example.

Suppose we have three peers, who's respective IDs are `peer1`, `peer2`, and `peer3`, or at least their ASCII hex representations as `7065657231`, `7065657232`, and `7065657233`, ho I'll refer to them by the textual names. Their hashed IDs using SHA-256 are therefore

| Peer ID | Hashed Peer ID                                                     |
|---------|--------------------------------------------------------------------|
| `peer1` | `698750a09b934337746f0973448167f364cae132e2f8b327ae4913e5b5445029` | 
| `peer2` | `3b213ced003e89b35a26c22cbd011c9bfab29578415b2069f7fc8b01998b903d` |
| `peer3` | `e42bbf8533f4f0b1d44e7fc1c9ac54a6ac368642dd1b8a10a1775255eed0c31a` |

If we now consider the keys `foo`, `bar`, and `baz` from before, they have the ASCII hex representations `666F6F`, `626172`, and `62617A`, which hash as follows:

| Key   | Hashed Key                                                         |
|-------|--------------------------------------------------------------------|
| `foo` | `2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae` |
| `bar` | `fcde2b2edba56bf408601fb721fe9b5c338d10ee429ea04fae5511b68fbf8fb9` |
| `baz` | `baa5a0964d3320fbc0c6a922140453c8513ea24ab8fd0577034804a967248096` |

For each key, we can now take the absolute value of the difference of the hashed key and the hashed peer ID. I used Python for this, as it supports arbitrary precision integers and hex notation, and the `hex` function returns hex notation for numbers in general. So then, for the key `foo`, the distances are



```
|h(foo) - h(peer1)| =
    3d609c3532937ca77ad3c437275126bf5188b3c27e74f386b4beb55d52dd687b

|h(foo) - h(peer2)| =
    0efa8881973ec323608b7cf09fd0db67e7706807dcd760c8fe722c793724a88f

|h(foo) - h(peer3)| =
    b8050b19caf52a21dab33a85ac7c137298f458d27897ca6fa7ecf3cd8c69db6c
```

Quick visual inspection shows us that `peer2` is closest to `foo`, and so the value associated with `foo`, which was the string `"x"` earlier, should be stored at Peer 2.

Running through this same process for `bar` and `baz` reveals that their respective values be stored at Peer 3. None of these key-value pairs should be stored at Peer 1. Seeing all of this is even easier with a good visualization:

```
       ^
       |
       |
       |
       |
 .-->  o-- h(foo) =
|      |   2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae
|      |
 '---- o-- h(peer2) =
       |   3b213ced003e89b35a26c22cbd011c9bfab29578415b2069f7fc8b01998b903d
       |
       |
       |
       |
       |
       o-- h(peer1) =
       |   698750a09b934337746f0973448167f364cae132e2f8b327ae4913e5b5445029
       |
       |
       |
       |
       |
       |
       |
       |
       |
 .---- o-- h(baz) =
|      |   baa5a0964d3320fbc0c6a922140453c8513ea24ab8fd0577034804a967248096
|      |
|      |
|      |
|      |
 \     |
  +->  o-- h(peer3) =
 /     |   e42bbf8533f4f0b1d44e7fc1c9ac54a6ac368642dd1b8a10a1775255eed0c31a
|      |
 '---- o-- h(bar) =
       |   fcde2b2edba56bf408601fb721fe9b5c338d10ee429ea04fae5511b68fbf8fb9
       |
       v
```

One thing we can observe from the visualization that's quite important, in fact, is that the peers are somewhat randomly distributed. The fact that similar peer IDs produce very dissimilar hashes is quite useful, as it means clumps of peer IDs won't produce clumps in the metric space. But notice that we can't avoid some clumping, since there's only a finite number of peers and there's no mechanism currently for controlling where they go in the metric space. For this reason, we want our hash function to be fairly uniform in the distribution of its output, so that no matter what the input, there is no _statistical_ clumping in the output. That way we minimize the clumps and get good fairly even coverage of the metric space. Choice of hash function is therefore important, and SHA-256 is a pretty good choice for this.

Considering the allocation of the hash space to peers, we can divide the hash space as follows. The bubbles on the left represent the regions of the hash space that is stored at the peer marked inside the bubbled region. Notice that this forms a kind of 1-dimensional Voronoi diagram. Voronoi structures end up being used in a variety of places in DHTs.

```
 ,---- ^
|      |
|      |
|      |
|      |
|      |
|      |
|      |
|      o-- h(peer2) =
|      |   3b213ced003e89b35a26c22cbd011c9bfab29578415b2069f7fc8b01998b903d
|      |
|      |
 :---- |
|      |
|      |
|      |
|      o-- h(peer1) =
|      |   698750a09b934337746f0973448167f364cae132e2f8b327ae4913e5b5445029
|      |
|      |
|      |
|      |
|      |
|      |
|      |
 :---- |
|      |
|      |
|      |
|      |
|      |
|      |
|      |
|      |
|      o-- h(peer3) =
|      |   e42bbf8533f4f0b1d44e7fc1c9ac54a6ac368642dd1b8a10a1775255eed0c31a
|      |
|      |
|      |
|      |
 '---- v
```

Another thing we can observe is that with this choice of metric space, there's an asymmetry between the middle and the ends of the metric space. The closer a peer is to the middle, the more space there is around them that counts as "nearby". The closer to the ends, the less space nearby space, because the ends have less room. This means that peers toward the middle will tend to have more data assigned to them than peers toward the ends. Ideally, we want a space that doesn't have this property.

To my knowledge, the linear metric space is never actually used in any real world DHTs, but it's conceptually simple and illustrative. A related 2-dimensional space is used in the CAN system, but it's not strictly a hash space, but rather a different, higher order concept.

### The Circular Metric

The easiest way to deal with the asymmetry of the linear space is to simply glue the ends together. Conceptually this is quite nice, but it also has the unfortunate consequence of making the distance metric somewhat more convoluted to express precisely. Instead, we'll just rely on the visual representations to understand what the circular space looks like. We orient numbers starting with 0 at the top, and increase clockwise, through numbers starting at 4 at the right, then numbers starting at 8 at the bottom, numbers starting at c at the left, and continuing up through numbers that start d, e, and f, until we wrap around to numbers starting at 0 again.

```
        bar (fc..)
                \
  peer  (e4..)   \     0  
            \     \    |          foo (2c..)
             \  _,-o'''''''--,_    /
              o'               ', /   peer2 (3b..)
            ,'                   o,   /
           /                       \ /
          :                         o
         |                           |
     c --|                           |-- 4
         |                           |
          o                         :
         / \                       /
        /   ',                   ,'
baz (ba..)    ',_             _,o
                 '--_______--'   \
                       |          \
                       8         peer1 (69..)
```

The circular metric space is indeed a metric space when using the topological nearness metric, but this also can make it so that the nearest peer might be counterclockwise "behind" a given key. The allocation of the hash space to peers looks like this:

```
                    __________
                -'''          \___
  peer3 (e4..)                /   '-,
            \                /       '-,
     .'      \  _,--'''''''-/,_
    /         o'           /   ',     peer2 (3b..)
   /        ,'                   ',   /
  :        /                       \ /      :
 |        :                         o        |
 |       |                           |       |
 |       |                           |       |
 |       |                           |       |
 |        :                      '''--,_    :
  :        \ _,                    /    ',-'
   \     _,-',                   ,'      |
    '--.'     ',_             _,o        |
       |         '--_______--'   \      /
        \                         \
         ',                      peer1 (69..)
           '-,_                 _
               ''--,_______,--''              
```

Assignment of key-value pairs to peers here turns out to be the same, just by accident of what the peer IDs and keys are, but suppose we had some key which hashed to all 0s. In the linear key space, it would be assigned to Peer 2. But in this circular key space, Peer 3 is actually closer. In fact, everything from about 1.. and below will be mapped to Peer 3. If we had exactly two peers in this system, then they would each always store exactly *half* the keys, no matter where they were located, while in the linear key space, a peer at the middle would have a massive burden compared to a peer right at either end. They divide the keys in half right at the middle point between them, and on a circle this means there's two directions that they halve, but on the linear space, there's only one direction they halve and the other two directions are exclusively controlled, leading to the burden. As the network grows, this burden shrinks, of course, but it's still an asymmetry that we can avoid relatively easily.

### Directional Circular Not-Quite-Metric

To my knowledge, the circular metric space is also not used, but circular spaces broadly speaking are. Chord, for example, uses a distance function which computes distance in the clockwise direction only. This isn't a metric function, because the distance from, say, `foo` to `peer2` is quite small, only about `09..`, but the distance from `peer2` to `foo` is quite large, around `ff.. - 09.. = f6..`. For DHTs, this is perfectly fine, because the keys lost in the space above a peer are made up by giving those keys to the next peer alone, and so on.

The allocation of the hash space to the peers, using the above circular space but using the directional not-quite-metric function looks like this:


```
                   ________
                  /        '''--__
  peer3 (e4..)   /                ''-,
            \   /                     ',
     .'      \ /_,--'''''''--,_
    /         o'               ',     peer2 (3b..)
   /        ,' \                 ',   /
  :        /                       \ /      
 |        :                         o------,,
 |       |                         / |       |
 |       |                           |       |
 |       |                           |       |
 |        :                         :        |
  :        \                       /        :
   \        ',                 \ ,'        /
    \         ',_             _,o         /
     ',          '--_______--' / \       /
       ',                     /   \
         '_                  /   peer1 (69..)
           '-,_             /  
               ''--,______.'              
```

You can see that in a sparse Chord-like network, you sometimes get cluster, as Peer 3 here is responsing for almost half the hash space. But as the network grows, the clustring rapidly goes away.

### Fancier Metrics and Not-Quite-Metrics

Many other distance functions can be defined over the hash space. Pretty much any function that produces a number will suffice, even if its not a metric, but often people like to use things as close to metrics as possible to ensure properties related to how routing works, which we haven't talked about yet. Another very popular metric function is the Kademlia metric, which defines the distance to be the bitwise XOR seen as a number, i.e. `d(x,y) = x^y`. This produces a metric space that's based on a tree structure rather than on a linear or circular space.

Hash values, when viewed as bit strings, can be seen as paths into a complete binary tree of depth 256, with `0` indicating "go left" and `1` indicating "go right", and so a hash value picks out a specific leaf node in the tree. The bitwise XOR of two hashes zeroes out all of the shared `1`'s, which entails that the longest shared prefix between the two hashes gets zeroed out. For example, `001011010111` and `001011011100` share the prefix `00101101` and so their distance looks like `00000000...`, namely, `000000001011`. The distance metric therefore says that nodes in the tree are "closer" to one another whenever their lowest common ancestor node is deeper in the tree/further from the root. The rest of the XOR distance after the 0'ed out prefix is also, of course, XORed, and we can understand it as measuring the similarity of the remainder of the paths.

Overall, tho, we can think of the XOR distance metric as being just about the similarity of the two paths, with a bias towards "preferring" rootward similarity over leafward similarity. When two hashes both say "go left" or both say "go right" (i.e. are `0` or `1`) near the root, then the two leaf nodes are "closer together" in the distance metric than two nodes that say opposite directions toward the root, but might agree lower down.

Let's look at a simplified example with hash space that's 4 bits long, and let's suppose we have peers at the keys `0101`, `0111`, and `1100`. The tree looks like this, using solid lines to indicate inhabited branches:

```
          /'\
         /   \
        /     \
       /       \
      /         \
    ,'\         ,'\
   ,   \       ,   \
 ,',   /'\   ,',   /',
,, ,, ,\ ,\ ,, ,, /, ,,
       o  o       o
    0101  0111    1100
```

If we look at the paths for `0101` and `0111`, they're visually quite similar:

```
/       /    <<< same
\   vs  \    <<< same
/        \
\        /
```

Or compare `0101` and `1100`:

```
/       \
\   vs   \   <<< same
/        /   <<< same
\       /
```

This isn't really much more informative than the binary, but I find that these visual representations really help understand the broader context. Kademlia, for instance, makes explicit further reference to this tree structure.

## DHT Routing and Topologies

Having defined a distance metric, it's now possible to ask how the peers are actually organized in relation to one another in terms of who knows who. In the simplest possible topology, every peer knows about every other peer on the network, which is to say they know the peer ID and their IP address or some other identifier that enables communication. The collection of this information is usually called a routing table. So then, when a peer needs to store or look up some key, they can just hash it, and find the peer who's closest to it according to the distance metric, and talk to that peer. But this is extremely inefficient in terms of space as the network grows. Not only does every peer need to store some information about every other peer, which is often something like a public key plus an IP address and therefore we could be talking a kilobyte per peer, but also all of that information has to be distributed to every peer, meaning there's lots and lots of traffic just to distribute the network metadata. We need to do better.

The usual way to address this is to require that each peer only know about a handful of other peers, and ensuring that the selection of peers is sufficient to connect every peer to every other through intermediate peer routing information.

Returning to the simple linear metric space let's imagine we have 9 peers. Let's draw these in a line, ignoring how far apart they are:

```
<--o-----o-----o-----o-----o-----o-----o-----o-----o-->
```

As long as each peer knows about its immediate neighbors, depicted below by arrows, then every peer can find every other peer through the routing knowledge of its neighbors.

```
     ___   ___   ___   ___   ___   ___   ___   ___
    /   v v   \ /   v v   \ /   v v   \ /   v v   \
<--o-----o-----o-----o-----o-----o-----o-----o-----o-->
    ^.__/ \__.^ ^.__/ \__.^ ^.__/ \__.^ ^.__/ \__.^
```

With this information present at each peer, any peer can communicate to another peer. One option is to have neighbors forward messages to remote peers. Another option is for the sending peer to ask its neighbors for more information about more distant peers, and then once that information is received, use that to ask for information about yet further peers, and so on, until the destination is found.

In the linear metric space, and also the circular spaces, knowing about only ones closest neighbors can be quite inefficient. Consider the case where the leftmost peer needs to talk to the rightmost peer. Then either the full message has to hop across all the intermediate peers, or the leftmost peer needs to learn about all the peers to its right, and thereby has to reconstruct the full map of the network. This is still better than the total knowledge routing approach, but it quickly approaches that as time passes.

A better option would be to require each peer to know a little bit more about the network than just its neighbors. For instance, if we stipulate that each peer knows exponentially further peers -- 1 hop, 2 hops, 4 hops, 8 hops, etc. -- then a message can very quickly make its way to the target with only logarithmically many steps. Here's what the leftmost peer would have to know in the above network with exponential distance included:

```
         _______________________________________
        /________________                       \
       /______           \                       \
      /__     \           \                       \
     /   |     |           |                       |
    /    v     v           v                       v
<--o-----o-----o-----o-----o-----o-----o-----o-----o-->
```

And here's the middle peer:

```
      _________________         _________________
     /           ______\       /______           \
    /           /     __\     /__     \           \
   |           |     |   \   /   |     |           |
   v           v     v    \ /    v     v           v
<--o-----o-----o-----o-----o-----o-----o-----o-----o-->
```

By having this information about exponentially more distant peers, any given peer can forward messages quite far in a single hop. A fairly conventional method is to forward to the peer that's closest to the target without going past the target, which usually does something roughly like halving the remaining distance. This way, we cut the distance in half and then half again and so on, making the total trip take logarithmically many forwarding steps. We can apply this approach to pretty much any semi-linear distance function, including the circular functions. Chord, for instance, makes use of an exponential distance increase like this.

The memory burden on each peer is also not bad, as it is also logarithmic, this time in size of the network. Doubling the network size requires at most two more entries in the peer's routing table. A network with 2<sup>n</sup> peers requires each peer to have around n entries in their routing table, so 1024 peers is a mere 10 entries, a million or so peers is a mere 20 entries, and a billion peers is just 30 entries, so one peer per human being is in principle entirely feasible while still having quite small routing tables. In practice, you can bump this number up dramatically without it being an extreme burden. Instead of a doubling of the distance, you can for instance pick a smaller multiply, say `sqrt(2)` instead, and cut your route lengths in half at the price of merely doubling the size of your routing table. This can be a welcome tradeoff when network speeds are the primary bottleneck.

Some routing topologies are somewhat more nuanced than this, however. In Kademlia, which uses an XOR distance metric, for instance, a tree structure is used to specify the routing topology. Let's consider again the tree from earlier:

```
          /'\
         /   \
        /     \
       /       \
      /         \
    ,'\         ,'\
   ,   \       ,   \
 ,',   /'\   ,',   /',
,, ,, ,\ ,\ ,, ,, /, ,,
       o  o       o
    0101  0111    1100
```

If we focus just on the leaf node marked `0101`, we can observe that for any other leaf node, there is a minimum subtree containing both, such that `0101` is in one half of that subtree, say the left half, and the other leaf node is in the other half of the subtree, the right have. And, moreover, any *other* leaf node in that *other* half subtree will necessarily be *closer* to that other leaf node than `0101` is. For instance, if there was a noce at `1111`, then because `0101` and `1111` are in different halves of the root tree, `1111` is closer to `1100` that `0101` is. Therefore, it suffices that each leaf node knows a single other leaf node in each of the "other halves" of all of the subtrees it appears in. For `0101` this means the right subtree of the whole tree, the left subtree of the left subtree of the whole tree, and so on. Here are the portions of the tree that `0101` needs to know about, marked off as `@` in place of the internal structure of those parts of the tree:

```
          /'\
         /   \
        /     \
       /       \
      /         \
    /'\          @
   @   \
       /'\
      /\  @
     @  o
         0101
```

You'll notice that these are the same nodes that show up in a merkle tree membership proof. This is coincidental, but interesting.

If we think about this in terms of prefixes, the node `0101` needs to know about one leaf node that is at a path of the form `1xxx`, one of the form `00xx`, one of the form `011x`, one of the form `0101`, which is to say the increasing larger prefixes that are different at the last digit. Extending this to the full hash space, with hashes of 256 bits, each peer needs to know around 255 other peers. Of course, this is all provisional on there being peers with those prefixes in the first place, and there will always be gaps, but that's ok. If the peers have this information, they can again route in such a way as to get logarithmically many hops relative to the total distance. In this case, the hops are getting closer and closer by single bits of the 256-bit hash value, but each such bit is some power of two as a number in the distance function, and so each successive hop is at least half the remaining distance, possibly more.

Another further optimization that can take place in DHTs that use recursive lookup of target peers, rather than message forwarding -- that is to say, DHTs where a sender doesn't delegate transmission of a message to another peer, but rather asks that peer for someone even closer to the target, and so on, until finding the target -- is that those new peers that are discovered during this lookup process can be cached. So for instance, in a linear network like this, with peers numbered 0-8:

```
   0     1     2     3     4     5     6     7     8
<--o-----o-----o-----o-----o-----o-----o-----o-----o-->
```

We know the routing table for Peer 0 will contain information on Peer 1, Peer 2, Peer 4, and Peer 8. And as it tries to route to Peer 7, it will ask Peer 4 for a closer peer to 7 than 4 is. Peer 4 of course knows about Peer 5 and Peer 6, and some others to its left, so Peer 4 tells Peer 0 about Peer 6, and now Peer 0 can simply cache that information about Peer 6 for future use, without re-asking Peer 4.

On the other hand, if messages are forward, then this kind of routing table caching can't happen. But other things *can* happen. For instance, if the responses are returned along the same path just in reverse, the intermediate nodes can *cache the data*, which increases storage redundancy and potentially makes it possible to get data from very nearby peers thereby reducing the burden on the target peer by spreading it to the caching peers instead. This approach is taken by Freenet and many other p2p systems.

## Example Uses of DHTs

Many interesting p2p tools are built on top of DHTs. They're used, for instance, in Freenet, in order to store information, and in IPFS to share information. Both Freenet and IPFS use a form of content addressable DHTs, where the key is the content itself. At first this seems paradoxical, why use a DHT if you need the content first, then you hash it and talk to some peer on the network who's hashed peer ID is closest to the hashed content? But usually you *don't* have the data, only it's hash. The hash of content can be used as the "name", so to speak, for that content and only that content, and so you can retrieve the full content by looking up that hash in the DHT. Freenet and IPFS different in the details of their hash identifiers, and also in the details of where data is stored: in Freenet, the data is stored at the peer with the closest hashed ID, while in IPFS it's stored at peers that opt in to host it, and instead what's stored at the other peer is metadata about who's hosting the data.

BitTorrent also uses a DHT, in a way very similar to IPFS, because it stores metadata about hosts, but instead of directly hashing the content, BitTorrent instead hashes a portion of a metadata file, which itself happens to contain the hashes of many pieces of content. For BitTorrent, those embedded hashes are primarily used not for identifying content, because that's the role of a torrent file, but rather for data verification after it's acquired to check that you've actually received the data that the torrent file refers to and not some other random junk that isn't mentioned in the torrent file.

IPFS and BitTorrent also have components, either in actual deployment or in proposed protocol improvements, for using a second DHT to permit mutable values. In IPFS, there is a component called IPNS which is a DHT that permits each peer to store a small chunk of a data on other peers, namely on some peers who's hashed peer IDs are close to their own hashed peer ID (so they store it themselves, but so do other nearby peers). The chunk of data is itself allowed to be only the hash of some data, which is to say the identifier of some data in the IPFS data DHT. This lets an individual IPFS peer to have some fixed unchanging identifier that can point to different actual pieces of data at different times. You could, for instance, host a website this way and add new content over time, by simply pointing the fixed IPNS name to different actual content addresses. BitTorrent uses this for so-called mutable torrents, where the specific content of a torrent file is seen as changing over time. This is very convenient for things like TV shows, where you can have a single torrent for the current season, and its contents get grow larger with new episodes as those episodes come out, rather than replacing the entire torrent file.

Another example use of a DHT, but orthogonal to data retrieval, is as a distributed search index. One could imagine constructing keys from words found in websites, perhaps all the single words, as well as all the bigrams (pairs of adjacent words in the order they show up in) or all the pairs of two words that appear anywhere (sorted by alphabetical order), and same for trigrams and triples of words, or something like this. And then you can hash these collections of words, and at the corresponding peer, you store a set of URLs for all the websites that contain that word or set of n-grams or set of words. Then when someone needs to search the web for some word, they hash it and ask the DHT for the set of URLs. To search for an n-gram you hash it and do the same. To search for a set of words you sort, hash, and ask the DHT. In this way we could build a very primitive kind of distributed search index that has no site ranking or anything, just word indexing.

## Layering Structures On Top of DHTs

One interesting thing that can be done with a DHT is to embed tree-like data structures into the DHT. Suppose for instance that we have a DHT like IPFS or Freenet, where the hash of the data is used as an identifier of the data. Then we can take any tree-like structure (or technically any directed acyclic graph) and recursively construct hashes and representations of data to embed it into the DHT as follows:

1. For leaf nodes with no outward edges, just hash the data and that's the identifier on the DHT at which the data is stored
2. For non-leaf nodes, which have edges out to other nodes, combine the node's data with the hashes of the nodes on the far sides of the edges, and hash and store all of that

This produces something known as a Merkle DAG, named after Ralph Merkle who invented them, or sometimes just a Merkle Tree, since it's indistinguishable from the perspective of the hashes. Git uses Merkle trees to produce hashes identifiers for commits, and IPFS uses Merkle DAGs for providing a convenient file-system-like structure of hierarchical "directories". This approach works more broadly for *any* kind of tree-like data structure, and so any such structure can be embedded directly into a DHT.

One very interesting use case of this is to embed a quadtree into a DHT. In that setting, rather than hashing the *content* of the quadtree nodes in a content addressed fashion, the keys being hashed are the center points (or equivalently, the bounding rectangles) that define the nodes, and the data stored at each node is the inhabitance information of the for quadrants (i.e. for each quadrant, is there data there or not). This then permits the user of the DHT to query for values that exist in a two-dimensional subregion of the entire region. This of course generalizes to k-d trees as well.

Another interesting embedded structure is the Merkle Search Tree, which is a particular variation on the B-tree tuned for ensuring that data is fairly evenly distributed across all the nodes in the tree without any active re-balancing.

## Limitations of DHTs

While DHTs are amazing, interesting structures will many important uses, they are definitely limited. The keys in a DHT, when hashed, usually produce bitstrings that are wildly disconnected from the keys themselves, in unpredictable ways. As a result, two keys that are very similar, even identical except for one bit, will hash to completely different bitstrings. This is a useful property of hash functions like SHA-2, but it means that fundamentally, DHTs are about retrieving for *specific* keys. Even the embedded quadtree example which permits looking up data in a range of 2-d point valued keys, rather than one specific key, is still actually using the DHT to store only a single item for each actual key: the quadtree node.

This can be a very significant limitation. So-called range queries, including the general higher-dimensional case of the quadtree queries above, are often better suited to a different kind of distributed data structure. Queries on strings using similarity measures, such as edit distance, which is very useful for fuzzy search indexes, are very difficult to implement in a DHT in a decent way, as well.

Let's take a look at one particular example of something we might want to do: an index of torrent files analogous to The Pirate Bay or RARBG, but distributed over BitTorrent itself and run collaboratively by everyone who cares to contribute just like data hosting on BitTorrent. This is currently relevant because RARBG shut down recently, at the time of writing, and its information is now only available as an archive someone made that's being hosted entirely on GitHub. The avid fan of RARBG might wonder if it's possible to use BitTorrent itself to provide the same functionality that RARBG's site provided, and the answer is yes, sort of.

RARBG, like so many other sites, provided many different ways of performing searches. It had a variety of taxonomies that can be used to categorize things, and it provides fuzzy string search. Now, obviously you could just bundle all of the relevant information together into a single torrent and just download that and search locall. But a potentially more useful structure would be a hierarchy of torrents and meta-torrents so you only have to download the information necessary to find what you want. So ok, structure the torrents into various taxonomic groups for TV shows, movies, etc. and inside those have subtorrents with alphabetically grouped information. Have also a subtorrent for your keyword database index, etc. etc. You can do all of this. But what if you want to add a new torrent file to this?

Well, that means building up a new tree of torrents, reusing all the parts you didn't change. That's not so bad, that's just a standard immutable datastructure problem. But... how do people *find* that new meta-torrent? You need some way to advertise its existence to the world. And what if someone else, meanwhile, added their *own* new entries, and now you have the moral equivalent of a fork in your Git branches? Well in IPFS and the various mutable torrent implementations, you could just have a single mutable reference, like an IPNS name, that gets updated with the actual torrent information as things change. What's wrong with that? Well, that is only possible if you have a *single* editor of the torrents. After a fashion, that's a perfectly good replica of RARBG, because only the RARBG crew and approved users could edit their site. But that's *very* different than a truly collaborative p2p equivalent of RARBG's index, where everyone on the network is contributing. To achieve that sort of behavior, we need to go beyond DHTs into entirely different structures.