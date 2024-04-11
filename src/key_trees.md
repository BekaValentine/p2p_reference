@title  Key Trees for Private Content
@date   2024-04-10
@author beka
@tags   symmetric keys, cryptography, private content, forums, group chat, access control

# Key Trees

In a setting where content is produced in an ongoing fashion, such as a blog, a web forum, a social media account, an IRC network, etc., it's sometimes necessary to constrain who can access what content. Private accounts on Twitter, private subforums, group chats, and so forth are all well known examples. Managing access to these is therefore something that needs good approach in a p2p setting.

## The Naive Approach: One Key Per Person

Let's assume that we're not concerned with multi-admin settings, and instead we're just dealing with the case of a single person controlling read-only access. A simple approach to controlling this is to simply encrypt the message once for every recipient. Revoking access means simply not encrypting it for that person. But this is terribly inefficient because it duplicates the message as many times as there are people in the allowed group. For a handful of people this might be acceptable, but for hundreds, or thousands, it's unacceptable, and for millions its infeasible.

## Key Groups

An alternative that is slightly better is to _assign_ keys to people, and to assign the same key to multiple people, forming a key group. This way, you reduce the number of duplicates that have to be sent. The cost of this is that when a revocation happens, you need to send out a new key to everyone who was in the revoked key group, except for the revoked individual. This is better, but only proportionally so.

So for example if your group size is 3 people, and there's 3 groups, you'd have a situation that looks like this:

- Group 1 = Key 1: A, B, C
- Group 2 = Key 2: D, E, F
- Group 3 = Key 3: G, H, I

Sending out the message M involves sending out three encrypted versions for each key: Enc(M, Key1), Enc(M, Key2), and Enc(M, Key3).

If E's access needs to be revoked, simply send D and F a new key, Key2v2, and discard the old key 2.

Now, the distinction between assigning people keys and using their pre-existing keys is really minimal, so the extreme case of one key group per person is functionally the same as the naive solution. But the conceptual flip is necessary to have non-trivial key groups, ones that have multiple members. The other extreme of key groups is to have one single big group. That might in fact be the most naive approach when assigning keys.

## Key Trees

The obvious next move is to structure the key groups inside of other key groups in a hierarchical fashion, to form key _trees_. The optimal arrangment is a perfect binary tree, but this isn't always possible. Considering the above example again, we'll have the following tree:


```
                .
         ______/ \______
        .               .
     __/ \__         __/ \__
    .       .       .       .
   / \     / \     / \     / \
  .   C   D   E   F   G   H   I
 / \
A   B
```

We read the group/key number off the nodes as follows: the root key is designated by just `root`, for visibility, and then for any other key, trace the path down: left is 0, right is 1, separating each digit from the previous part of the key with a `.`. For instance, D is left, right, left, so its group/key `root.0.1.0`.

This tree structure indicates who has which keys: everyone has all the keys above them, so D has keys `root`, `root.0`, `root.0.1`, and `root.0.1.0`.

To send out a message to everyone, we simply encrypt using the `root` key. This is nice, because it means there's only a single message that has to be published, not many for many different key groups.

In the case when we need to revoke access, for instance to E again, we need to replace all of the keys that it has access to. Fortunately this is only logarithmically many keys, and more importantly corresponds to the nodes in the path from `root` down to E's key `root.0.1.1`:

```
                X
         =====// \______
        X               .
     __/ \\=         __/ \__
    .       X       .       .
   / \     / \\    / \     / \
  .   C   D   E   F   G   H   I
 / \
A   B
```

So we need to revoke `root`, `root.0`, `root.0.1`, and `root.0.1.1`. The keys that _aren't_ revoked are also tidily aranged: they either _are_ the siblings to one of these keys (i.e. `root.1` for the group FGHI, `root.0.0` for ABC, and `root.0.1.0` for D) or they are keys for descendents of one of those siblings (e.g. the key for C is a descendant of `root.0.0`, namely `root.0.0.1`).

So we need to distribute new keys to the remainder of the key tree after we remove E's access. The new tree, formed by simply removing the node E and tidying up, looks like this:

```
                .
         ______/ \______
        .               .
     __/ \__         __/ \__
    .       D       .       .
   / \             / \     / \
  .   C           F   G   H   I
 / \
A   B
```

The group ABC still has access to the key `root.0.0` so they only need a new key for the ABCD group. Similarly, D still has the old key `root.0.1.0`, which now becomes the key `root.0.1` since there is no more key `root.0.1.1`. The groups under the key `root.1` don't need their keys changed. But everyone also needs the new `root` key. So we can simply send out keys as follows:

1. Generate a new key for the group ABCD, i.e. a new `root.0`, and send it out encrypted to the keys `root.0.0` and `root.0.1` (which is the old `root.0.1.0`, which only D has). Now all of ABCD has the new `root.0` key.
2. Generate a new key for the group ABCDFGHI, i.e. a new `root`, and send it out encrypted to the keys `root.0` (which we just distributed), and `root.1`.

To add a new person J to the key tree, we find the shallowest node, which in this case is D since we removed E, and we split the tree there, which happens to mean that the new person is in the spot that E used to occupy:

```
                .
         ______/ \______
        .               .
     __/ \__         __/ \__
    .       .       .       .
   / \     / \     / \     / \
  .   C   D   J   F   G   H   I
 / \
A   B
```

Person D still has access to the key `root.0.1` (which, you'll observe, used to be `root.0.1.0` until E was removed). So we send a newly generated `root.0.1.0` to D, and we also send a newly generated `root.0.1.1` to J, as well as all of `root`, `root.0`, and `root.0.1`, so that J has all the keys above it. Note that we could also instead generate a new key for `root.0.1` instead of using the key that originally was `root.0.1.0`. How we do this doesn't really matter.

It's worth noticing now that the only time messages have to be sent out to multiple sub-groups is when revocations happen, and moreover only logarithmically many keys have to be sent out as there total members of the group. There are some parasitic cases -- for instance if you have to block a lot of people and it just so happens that the people being blocked are just so arranged as to cause the resulting tree to be very deep, but not very bushy, for instance like so:

```
  .
 / \
A   .
   / \
  B   .
     / \
    C   .
       / \
      D   E
```

This is the worst case scenario, where the number of revocations is equal to the number of people in the group. Fortunately, getting into this situation is going to be very rare, because it requires a lot of revocations with very few access grants happening at the same time, which would counterbalance the structure of the tree. It's also going to be logarithmic in the largest size of the tree, since access grants are defined to expand the key tree at the shallowest parts, thus making it bushier not deeper, so the only way you could ever have gotten a deep branch like this is if at some point you had granted access to enough people to fill the tree down to that point. In this case, D and E are at level 4, which means level 3 had to have been full, i.e. you needed 2^3 = 8 people with access at one point. So even the worst case parasitic situation is actually _fine_, since its indistinguishable from the case where you just have lots of people with access: its always linear in the depth of the tree which is logarithmic in the maximum capacity of trees of that maximum depth.

## Handling High Churn

While this is a very good way to organize keys, it may in fact still be a problem for very large groups with a very high churn rate, lots of revocation happens (compensated for by lots of new access grants). Such a situation might arise, for instance, from extremely popular social media accounts, or from popular discussion groups, with automoderation to prevent spam, etc.

A way of coping with this, which is suboptimal but will definitely work, is to simply batch process revocations at a fixed cadence. For instance, once an hour or once a day, rather than processing revocations immediately. This would reduce the number of messages proportional to the depth of the tree and the number of revocations. For a mass block situation, for instance because of the use of a block list which might block tens of thousands of users at once, batch processing would be extremely useful even in a one off situation.

## Who has the tree?

A very important fact of this design is that the tree is entirely on the controlling side. No one who's been granted access keys has any real notion of a tree, each of them simply has a set of keys that can be used to decrypt messages from the controller. The keys certainly will be used with very different frequencies, the root keys being used for the overwhelming majority of massages and the non-root keys being used with inverse exponential frequency proportional to their depth, but that information is inessential. The grantees merely need to have a set of keys. Only the controller of the tree needs to have the tree, and even then the tree can in principle be purely conceptual, as you can implement everything less efficiently with key identifier prefixes (e.g. `root.0` is a prefix of `root.0.1` so if `root.0.1` is revoked so is `root.0`). Most likely, some kind of prefix trie would be used to store key trees for efficiency anyway, even tho they're not essential.

## Possible Modifications

Some modifications to the key tree approach are possible which might optimize the tree a little better. For example, you could restructure the tree after revocations to keep it as balanced as possible. There is a tension here, however: restructuring means potentially a _lot_ of new keys need to be distributed, because the groupings have changed. This is probably not too good. Some kind of local restructuring might be fine, however.

Another option is to never actually make the tree more shallow, so that intermediate groups continue to exist after revocations. For instance, when we revoked E's access above, the resultant tree would look like

```
                .
         ______/ \______
        .               .
     __/ \__         __/ \__
    .       .       .       .
   / \     /       / \     / \
  .   C   D       F   G   H   I
 / \
A   B
```

This would require D to get a new key for the revoked group, but it would mean that adding a new node later would place it next to D and only require a message to the new node instead of to the new node and also to D. This is a very minor design variation and the differences here are almost certainly incidental, with little to no impact on the overall system behavior.

## Uses of Key Trees

The above description of key trees enables one to many messages to be made private. This functionality replicates private accounts in various systems, as well as things like "circles" in Twitter, which are further limited subsets of people who have access to only certain content.

But because the root key is shared by all members of the group, it can also be used by all group members as a means of posting their own messages privately to that group. This is especially important for replies to private content, which should ideally be private as well, but only to that same group. This makes it easy to implement things such as private chat rooms, private forums, and so forth. Little if any additional infrastructure is needed to have this functionality.