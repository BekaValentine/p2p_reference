@title  Bootstrapping
@date   2023-09-02
@author beka
@tags   bootstrapping, bootstrap node

# Bootstrapping

For any given p2p network that is open to the public, there has to be some process by which new peers can join the network. For a private network, too, there need to be processes, but of course private networks are fully controlled by the people running them, so many methods can be used in that setting which presuppose a lot of knowledge and coordination and control. In a public network, on the other hand, the act of joining is controlled by the person who sets up the new peer, and by the conventions of the system as encoded in software and other knowledge that is shared prior to connecting to the network. But no real actual coordination of the sort that's in principle possible in a private network can happen. No one is your controller, you decide where you are, what your computer is like, where you are on the broader internet, etc, and yet still somehow you ought to be able to connect to this one, specific p2p network that you've chosen. That process is called bootstrapping. How does that happen?

## Bootstrap Nodes

Bootstrap nodes are probably the most popular approach to bootstrapping because it's so wildly available and reliable at creating a single connected network. The basic idea of a bootstrap node is that some peers, or peer-like computers with a pared down function set, are set up and maintained by trusted parties, such as the core developers, and their IP addresses are baked into the software that's distributed to the world. Then, when that software is launched, it contacts the fixed set of bootstrap nodes and asks for information about the network. In a DHT, for instance, a peer would contact a bootstrap node and ask it for the peers nearest to its own peer ID, which would tell it who's "nearby".

Because of the potential changing nature of IP addresses as infrastructure is shuffled around, upgraded, etc., a common variant of this is to instead use a domain name and rely on DNS for the IP lookup. This also makes it possible for the DNS records to contain information such as the public key of the bootstrap nodes, permitting the joining peer to engage in a cryptographic challenge with the bootstrap to ensure that it's not been taken over by malicious parties, or anything to that effect. Other information can be put into the DNS records as well, such as lists of trusted other bootstrap domain names, which can be used in case the one currently being contacted is offline for some reason. This places a greater role on the DNS system, of course, but DNS tends to be very reliable, so it's not too bad.

One downside to the bootstrap node approach, however, is that it requires a trusted set of bootstrappers. In practice this is fine, because bootstrapping is a very lightweight process. Bootstrap nodes exist pretty much only to make introductions, so any introduction to someone on the network will suffice. The only risk of this is that a node will be malicious and not introduce people to the "true" network but to some shadow network, but this isn't a huge problem. If there are multiple bootstrap nodes, some of which are malicious but others are not, then asking all of them will connect a peer to *all* the different networks, and as soon as that peer joins all the networks, they're fused into one big network through that peer. So malicious network splits (netsplits) cannot last very long in the presence of even one non-malicious node. Hence, more bootstraps means better network connectivity.

## Ask A Friend

An alternative to the bootstrap node method is to just ask a friend. This is actually just the same approach as bootstrapping, except there isn't a single fixed list of bootstrap nodes. Instead, you perform the same process, but instead of the fixed list, your peer communicates with a computer that you designate to be trusted for this purpose. This is especially useful in a setting where the social dimension is very present. For instance, in social media oriented networks, getting onto the network via an invite from a friend, which contains the equivalent of what a bootstrap node would provide, is a pretty good technique. The main drawback of this, however, is that netsplits are far more likely, because invites propagate out through social networks, and it's entirely plausible to have many disconnected ones throughout the world.

## Random Probing

Another method, arguably the worst option in some cases, is to randomly probe computers. For local networks, this is not that bad, and this is what Secure Scuttlebutt can do. In fact, local networks make it possible to broadcast presence with UDP, obviating the need for an actual scan, because nodes can simply respond with an "I See You" message. But for the wider internet, scans have been used and are somewhat reliable at finding nodes. They just tend to be slow, unreliable at finding all nodes due to node downtime, and ISPs generally won't like it. Some proxies for actually scanning yourself do exist, however. For instance, the search engine Shodan does this at scale regularly, and provides a useful method for searching the scan data for computers with certain network capabilities. This kind of proxied scan is kind of a blend between scanning and relying on bootstrap nodes, because you are still here relying on a single source of failure -- Shodan -- but so long as that still exists, the information will be available. [Here](https://www.shodan.io/search?query=bittorrent) is an example of using Shodan's publicly available search to find computers running BitTorrent. One downside of relying on sites like Shodan, however, is that if Shodan doesn't know about your protocol, then it won't necessarily have any useful results.

## Meta-Networks

A four option is to use some other system, usually a DHT of some sort, to provide the same functionality as bootstrap nodes. Before connecting to the target network, the peer software would connect to the meta-network using it's own internal bootstrapping process, and then use that network to ask for peers on the target network. At first, this doesn't seem like we've gotten any benefit. In fact, it seems worse: to connect this network, first connect to some other network! Ok, so do you need a meta-meta-network? Well usually the meta-network would use one of the *other* bootstrap techniques, typically the bootstrap nodes. So we seem to gain little. But the meta-network is intended to act as a meta-network for not just the target network, but for *all* p2p networks that need to be publicly accessible. Since there are many p2p networks doing all sorts of weird stuff, a single meta-network would have a comparably larger set of participants, and thus be more robust than any one network could be in isolation. The downsid is this adds a whole other layer of p2p complexity, and has a much larger burden on the meta-bootstrap nodes.

## Remember

The last approach that I'm aware of is not intended for first-time connection, but rather reconnection to the network after a disconnection. That answer is to simply remember the peers that were previously seen. Many of those peers will probably have gone off line or changed IP addresses, of course, and the longer your peer spends offline the more that will happen, but for short-duration disconnections it's not going to be as bad. Since people use laptops and even smart phones as their primary devices these days, disconnections from wifi and cell service are frequent, and simply remembering who was online and then asking them for bootstrap information is going to be far less intensive on the actual bootstrap nodes.