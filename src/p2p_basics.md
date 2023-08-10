@title  Peer-to-peer Basics
@date   2023-08-10
@author beka

# Peer-to-peer Basics

This document is an attempt to provide a ground-up introduce to peer-to-peer
(p2p) technology for people who are familiar with computery things broadly
speaking, but not necessarily familiar with p2p. For people familiar with p2p,
this will hopefully be a nice, concise picture of the high-level issues.

## What does "peer-to-peer" mean?

"Peer-to-peer" is a term referring to a broad class of technologies for building
networked computer systems where all of the participants in the network are more
or less equivalent in terms of what they're able to do. This is in contrast to
centralized systems that involve significant asymmetries in the roles played by
the different participants.

A useful comparison can be made in the problem domain of search. A standard
approach to search, from the earliest web search engines through to today's
dominant search giants, is to have a service provided by a single entity, such
as Google today, or Altavista in the 90s, which people can use to search for
information on the web. All of the work is done by the service's computers, and
all of the data is stored there as well. The user who wishes to search for
something merely visits the search engine and requests some information.

Contrast this with the phenomenon of `#lazyweb` in the later Twitter era.
Rather than requesting information from one centralized data warehouse, users
of Twitter, and other social media sites, post a request to their social media
feed asking their followers if anyone knows anything about the desired subject.
If so, people reply with that information. The information is not stored in a
single centralized repository, nor is it retrieved by that repository's
computers, but rather it's stored in the user's *fellow users*, i.e. their
peers, and retrieved from memory by those peers. And moreover, those peers can
do the exact same thing and get information from anyone who follows them. Each
person can both request information and provide it, so there's symmetry.

Now, there are certainly a lot of confounding factors here, because Twitter and
most other social media systems are themselves centralized, but those aren't
essential to the point, which is that every relevant participant can play the
same roles: everyone can request information and also everyone can provide
information. In a sense, then, `#lazyweb` is a form of peer-to-peer search.

Typically, tho, when we talk about p2p stuff, we generally mean not
person-to-person, per se, but rather some equivalent thing done by computers.
Certainly there are ways in which people do "p2p" research about purely social
phenomena that don't involve computers and that's p2p very broadly construed,
but usually we mean computery things.

We also often have many different kinds of p2p systems, especially historically,
which have more or less p2p-ness. For instance, one of the most famous and
historically impactful p2p systems was Napster, a p2p file sharing system. But
Napster was not, actually, a p2p search system. It's search was entirely
centralized on Napster's servers, which is what lead to its eventual death at
the hands of the Recording Industry Association of America (RIAA)'s copyright
lawyers. What made Napster p2p was that the music and other files being shared
on the system were all distributed around on the computers of the users, and
they were transfered directly from one user to another, without going through
Napster's computers at all. Napster merely acted as a kind of directory for
where to find the files in question. A similar structure exists today in the
world of BitTorrent, arguably the most successful p2p tool ever made. BitTorrent
doesn't have a single search mechanism like Napster did, rather, it has an
entire ecosystem of search websites, but they are themselves centrally
controlled by the site operators, and have similar problems with lawsuits. The
most famous of the torrent search websites -- The Pirate Bay -- has only managed
to avoid permadeath by being located in Sweden, which gives them certain legal
protections (at least until the RIAA and MPAA manage to get Swedish law
changed).

But there are many many kinds of p2p systems that are different than this. A lot
of research over the last 25 or 30 years has gone into finding ways to store,
index, and retrieve information in decentralized, peer-to-peer ways. Lots of
points of variation exist around how much centralization, and what kind, is or
is not needed, and how it can be avoided. Many of the techniques have similar
shapes, and so we can give a fairly high level overview of many many p2p systems
look like, if you squint and ignore the details.