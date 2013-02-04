Distrust - Distributed Trust
============================

Distrust is an experiment in implementing distributed trust system for software
authentication. The intent is to allow users to trust certain developers to
distribute certain software and prevent man-in-the-middle attacks, rogue
releases and compromised download servers.

Distrust is not a new idea, it is, essentially, what some people have been
doing all along, manually checking that a source tarball is properly signed by
a person they trust to release it.

All that distrust does is to take that idea and formalize it so that any user
can benefit and be more secure than they currently are. As an additional
feature the Distrust system allows one to have a meaningful transition from the
mostly-unsigned world to the mostly-signed world (see UNSIGNED SOFTWARE below).

The trust model
===============

Distrust models its trust network after GPG (Gnu Privacy Guard) with some
notable exceptions and extensions.

This document does not explain GPG in any way, you are encouraged to read the
Gnu Privacy Handbook: http://www.gnupg.org/gph/en/manual.html.

The basic concept is that a given user (ISSUER) chooses to trust another user
(SUBJECT) to author some named piece of software (SCOPE).

The distrust system is designed with digital software distribution in mind and
as such it wants to be practically applicable to current environments. As such
it deals with both signed and unsigned software. In both cases the user has the
same level of security, based on trust. Unsigned software is simply more
annoying to trust in practice.

The trust model is distributed because each trust record is signed by the
issuer and can be exchanged with other users. In the same way someone can be a
trusted author of software he might be a trusted reviewer (so I trust the
software he trusts). In both cases control is exclusively in the hands of the
user, there is no implicit trust.

SIGNED SOFTWARE
---------------

This is the preferred mode. Here the user expresses trust to a particular
person (as identified by their GPG key signature) to author and release
particular software.

The trust system builds upon GPG for known peer identification (so it's good
to know the developer of the software, verify their identify and sign their
public key) but extends it to specify how the trust should be interpreted
(after all, GPG only provides secure authentication, not trust in the action
of the identified person)

Distrust requires working GPG implementation. In all examples below we assume
that all participants already use GPG and have valid keys. More importantly we
do NOT assume that all users exchanged their public keys.

UNSIGNED SOFTWARE
-----------------

This mode is based on cryptographic hashes that are trusted by the user
themselves, and do not require the software distributor to sign. Those are
more annoying as each single unique file needs to be explicitly trusted,
preferably after a manual inspection.

Tutorial
========

Let's look at a first example. The user Alice starts using distrust.

    $ distrust init
    [distrust] Creating trust database in ~/.config/distrust/trust.json

This is basically the clean slate, Alice does not trust anyone yet. It is
important to distinguish the trust Alice chose to put in GPG keys from the
trust she puts in particular people to author software. Those are explicitly
separate.

    $ distrust list
    [distrust] Listing all trust records signed by "Alice User <alice@example.com>"
    [distrust] There are no trust records yet

Now Alice wants to install a program that her friend Bob wrote, "useful-tool".
Alice already has his public key in her GPG keyring so that part of the problem
is out of the way.

The way the distrust system works, Alice needs to create a trust record that
names Bob as trusted to sign releases of the "useful-tool" program. Since there
is no global naming scheme, Alice needs to namespace the program to a
particular distribution channel. Here Alice will use the "pypi" namespace.

The point of using namespaces is to allow the distrust system to be used by all
developers and downstream consumers (users or developers) regardless of the
language or technology that a particular program is associated with. If
successful, Distrust could be deployed to secure pypi, ruby gems, ubuntu ppa
packages, freedesktop.com source releases and anything else.

Now, going back to the example, Alice will trust Bob to author "useful-tool"

    $ distrust trust person "Bob Developer <bob@example.com>" with project "pypi:useful-tool"
    [distrust] Looking up "Bob Developer <bob@example.com>"
    [distrust] Found 1 personality:
    [distrust] pub   2048R/FA519244 2013-02-03 [expires: 2015-02-03]
    [distrust] uid                  Bob Developer <bob@example.com>
    [distrust] sub   2048R/A63BAB03 2013-02-03 [expires: 2015-02-03]
    [distrust] Are you sure you want to trust:
    [distrust] "Bob Developer <bob@example.com>" with project "pypi:useful-tool"?
    [distrust] [yes or no] > yes
    [distrust] Creating signed trust record...
    // GPG invoked to sign the record
    [distrust] Adding trust record to the database...
    [disturst] Done

Now Alice's database contains a record containing precisely that information.
Using it she can download the program from any website or mirror and install it
knowing it came from Bob.

    Note: in the future distrust will be integrated into pip and setuptools
    directly so that all downloaded files have to be trusted and can be
    properly verified.

Alice can proceed to verify the tarball she has:

    $ distrust check "usefull-tool-0.1.tar.gz" with "useful-tool-0.1.tar.gz.asc" for project "pypi:useful-tool"
    [distrust] Checking signature... ok
    [distrust] Checking signer... ok
    [distrust] Checking trust for scope... ok

    Note: It might make some sense to create a new format that encapsulates those three pieces
    of information: the scope, the signature and the payload. Perhaps an ar record, like Debian?

    $ ar t usefull-tool-0.1.dti

Core Commands
=============

There are just two core commands, trust and check. They have two forms each,
one for each signed and unsigned software.

trust person ... for project ...
--------------------------------

The syntax is:

    $ distrust [--remove|--revoke] trust person <person> for project <namespace:name>

This is the mode for trusting SIGNED SOFTWARE.

This interactively creates a trust record that allows a given person to
issue releases of the specified project.

The optional --revoke keyword can be used to revoke the trust and create an
explicit distrust record. The user can also provide an optional comment.

The optional --remove keyword can be used to remove a trust record without
creating additional revocation records. This is useful for mistakes where no
breach of trust had occurred and the <person> is still trustworthy.

This command requires access to the GPG keyring to discover and confirm the
identify of the <person> and to sign the trust record.

trust file for project
----------------------

The syntax is:

    $ distrust [--remove|--revoke] trust file <file> for project <namespace:name>

This is the mode for trusting UNSIGNED SOFTWARE.

This interactively creates a trust record that allows anyone to distribute
that particular file which is a part of the the specified project.

This mode allows the user to use the trust system with existing, unsigned
software before the upstream chooses to adopt software signing policy.
The trust is based on a computed, selected cryptographic checksum of the
specified file. The trust records explicitly stores that checksum.

This command requires access to the GPG keyring to sign the trust record.

check ... with signature ... for project ...
--------------------------------------------

The syntax is:

    $ distrust check <file> with <signature> for project <namespace:name>

This non-interactively verifies the trust on a specified file, with the
specified signature file for a particular project. The return code can
be used to discover the failure reason. This command should be used
by external tools to verify downloads. The error codes are documented:

    0 - success.

    This is only returned when a valid signature, created by known issuer,
    was previously trusted by the user for the specified scope.

    1 - invalid signature

    The signature is not valid for the specified file, no further checking
    is provided.

    2 - unknown signer

    The signature is valid but the person signing the file is not known

    3 - no trust record

    The signature is valid, the person is known but there is no trust in that
    person for the specified scope.

The "check" command has a second mode that is suitable for UNSIGNED SOFTWARE.

The syntax is similar:

    $ distrust check <file> for project <namespace:name>

The only difference is that the signature part is missing. The return codes
in this case are either 0 (success) or 3 (no trust record).

Auxiliary commands
==================

init
----

The syntax is:

    $ distrust init

This initializes the trust system and keeps a persistent database of trust
records in a platform-specific directory specific to the currently running
user.

list
----

The syntax is:

    $ distrust list [--issuer=<person>] [--subject=<person>] [--scope=<scope>]

    TBD

publish
-------

The syntax is:

    TBD

    Publishes trust records to a server

import
------

The syntax is:

    TBD

    Imports trust records from a server

trust peer
----------

The syntax is:

    TBD

{Creating,removing,revoking} trust records for peer users (I trust that
$PERSON trust records as I would have if I created them myself)
