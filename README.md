# p2p_reference

A reference for p2p tech

## Tooling Note

This repo is meant to be used with my [php site generator](https://github.com/BekaValentine/python_site_generator).

Some documents contain attributes that can be used for other fun things. Here
are some of them:

`parent`
: a document/concept that is in some sense a supertype or hypernym of this one.

`related`
: a related document/concept that is related in some generic unspecified way.

`related.<relname>`
: a related document/concept that is related by the relation `<relname>`. for
example, a particular p2p technology might be inspired by some other technology,
and the document for the former might contain `@related.inspired_by <latter>`.