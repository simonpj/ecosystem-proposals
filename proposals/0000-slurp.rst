.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.

.. highlight:: haskell
SLURP: a Single Liberal Unified Registry of Haskell Packages
================================================

Mathieu Boespflug, Manuel Chakravarty, Simon Marlow, Simon Peyton Jones, Alan Zimmerman
January 2018.

1 Introduction
--------------

Hackage has been extraordinarily successful as a single repository
through which to share Haskell packages. It has supported the emergence
of variety of tools to locate Haskell packages, build them and install
them (cabal-install, Stack, Nix, ...). But in recent years there has
been increasing friction over,

-  Hackage’s policies, especially concerning version bounds;
-  Hackage's guarantees, especially around durability of package content
   and metadata;
-  Hackage's features, especially the visual presentation and package
   documentation.

Tools for building packages hosted on Hackage have diverged in their
design philosophies. It is possible that more such tools will appear in
the future exploring yet more points in the design space. Each of these
tools have different needs. Some tools (e.g. cabal-install) require
accurate and conservative version bounds to function correctly. Others
do not. Some tools (e.g. Stack and Nix) require very strong guarantees
that build plans that work today will work as-is tomorrow. Others do
not.

It would be great if a single set of policies, guarantees, features and
technologies could cater for the needs of everyone. But the fact of the
matter is that finding the best practices for building projects made up
of independently developed packages in a language with early binding,
static checking and rich types, **is still a matter of active
research**.

At the same time, fragmentation of the community around segregated
package ecosystems would be a terrible outcome for everybody. We'd end
up with no interop between build tools, conflicting package names and
frustrated developers who'd have to "choose sides" (or choose their
poison) for each new project they undertake, depending on which
dependencies they need most.

If we do not solve this problem, it seems likely that the Haskell
library ecosystem will soon “fork”, with two separate repositories, one
optimised for Cabal and one for Stack. This would be extremely
counter-productive for Haskell users.

This document puts forward a proposal that seeks to remove the friction
that exists today among infrastructure authors (Hackage, Cabal, Stack,
etc) while enabling strong interoperability between tools.

2 Principles
------------

We start from these principles:

-  The diversity of ecosystems is a Good Thing: it allows different
   tools preferred by different users for different purposes.
-  There should be a single unambiguous namespace for all published
   Haskell packages. In this way, all build tools can uniformly refer to
   all packages.
-  Package names should not depend on their provenance:
   ``bytestring-0.10.8.2`` is ``bytestring-0.10.8.2`` no matter where
   you downloaded it from today (which might be a different place from
   where you'll download it tomorrow).
-  Subject to these constraints, we should leave package tools maximum
   flexibility to explore and innovate.

Note: There are really three things that people talk about when they say
“Cabal”:

1. *Cabal-the-spec*: a package metadata format (``.cabal`` files) - at
   one time part of a
   `specification <https://www.haskell.org/cabal/proposal/pkg-spec.pdf>`__.
2. *Cabal-the-library*: an implementation of Cabal-the-spec as a
   framework.
3. *cabal-install*, aka *Cabal-the-tool*, which is a command-line tool
   that builds sets of packages, using Cabal-the-library for each single
   package in the set.

In this document we’ll use “Cabal” to mean cabal-install and its
dependency solver, unless otherwise noted.

3 SLURP: a Single Liberal Unified Registry of Haskell Packages
--------------------------------------------------------------

The core component of this proposal is a new service called SLURP,
designed to enable different groups to offer their own platforms for
hosting packages and supporting build tools. We designed SLURP starting
from the following question:

-  What is the minimum we should consider *shared*, i.e. what should
   people really not fork under any circumstance? In other words, what
   really needs to be centralized, and where can people concurrently
   innovate on their own platforms carrying whatever content they want
   according to whichever policies?

Stack, cabal-install, and other package-management ecosystems such as
Nix, work in different ways, but they all need one thing: a standard way
to:

1. list all available packages,
2. for each package, list all available versions,
3. for each available version, locate and download the corresponding
   source distribution (a ``pkgname-1.2.3.tar.gz`` file).

Each build tool or package manager implements more tool-specific
features on top. For example, here are some additional features
cabal-install provides:

-  download a bundle of all available metadata for all available package
   versions (a ``01-index.tar`` file),
-  resolve a build target into a build plan satisfying the version
   constraints found in the metadata bundle.
-  execute the build plan.

Here are some additional features Stack provides:

-  given an LTS snapshot name, locate and download a package set (an
   ``lts-x.y.yaml`` file),
-  resolve a build target and a package set into a build plan,
-  execute the build plan.

Nix has its own additional features. The point is, all of these use
tool-specific downloadable resources (e.g. cabal-install's
``01-index.tar``, Stack's ``lts-x.y.yaml``) *can be provided by
tool-specific servers and we can keep them entirely separate from this
proposal*.

It's all well and good to locate and download package versions
uniformly. But how do we build them? This supposes a common framework
for specifying how to build a package. We already have such a framework:
Cabal-the-spec. We furthermore already have an implementation of this
specification: Cabal-the-library and the various ``Setup.hs`` build
files that authors have been shipping with their packages.

So to answer our original question, the minimum we need to *share* is:

-  A centralized understanding of who "owns" what package name and where
   to go to find all package versions the maintainer has published under
   this name.
-  A central registry that implements this understanding: this is SLURP.
-  A common format for package metadata (i.e. Cabal) and a common way to
   build packages.
-  A standard API for performing (1), (2) and (3) above.

Given a central registry to locate packages (SLURP), any build tool or
package infrastructure can query which server a package is hosted on.
Then, provided that server implements a standard API, the build tool can
query available versions and download source distributions.

4 The SLURP API
---------------

The central registry, SLURP, is a new service. Its purpose is to allow
clients claim a package name, to list all available packages, and to
find where a package is hosted. It implements the following API:

-  ``GET /packages`` - returns a list of all open source packages in the
   Haskell universe.
-  ``PUT /package/:pkgname`` - register ownership of a new package name.
   Map that name to some URL that you own. Example:
   ``{"name": "mypackage", "location": "https://myserver.com/package/mypackage" }``
-  ``GET /package/:pkgname`` - redirects to URL provided by ``PUT``.

No authentication or user accounts required to make requests to either
of those endpoints.

Example package list returned as the response of a ``GET /packages``
request:
::

    { "packages":
      [ {"name": "bytestring", "location": "https://hackage.haskell.org/package/bytestring"}
      , {"name": "text", "location": "https://hackage.haskell.org/package/text"}
      , {"name": "Cabal", "location": "https://hackage.haskell.org/package/Cabal"}
      , {"name": "conduit", "location": "https://stackage.org/package/conduit" }
      , {"name": "yesod", "location": "https://stackage.org/package/yesod" }
      , {"name": "mypackage", "location": "https://myserver.com/package/mypackage" }
      ...
      ]
    }

You can think of SLURP as an *authoritative* URL-shortening service for
packages. Authoritative means that which package is generally accepted
to have which name depends on the content of the SLURP registry and only
that.

5 The package API
-----------------

The target location for each registered package must support at least
the following subset of the Hackage API:

-  ``GET /package/:pkgname/preferred`` returns a JSON structure listing
   all versions.
-  ``GET /package/:pkgname/:tarball.tar.gz`` where tarball is a
   name-version pair.

We choose to define the package API as a subset of the current Hackage
API for backwards compatibility: it means Hackage is a valid package
hosting server from Day 1, without change.

We put forward no requirement on the guarantees provided by package
hosting services. For example, we do not impose that getting a
particular package tarball for (say) ``pkg-3.5.3`` always yields the
same result each time. (In Hackage, for example, the presence of
revisions may mean it will not.) Immutability, or lack thereof, is an
example of hosting-service specific policy that does not concern SLURP.

6 Key properties
----------------

6.1 Federated infrastructure and GitHub hosting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Separating the registry from storage of the package tarballs themselves
allows package hosting to be *federated* and *decentralized*. 
For example, if you are
the maintainer of ``bytestring``, Hackage lets you "own" the
``https://hackage.haskell.org/package/bytestring`` URL, so it's a legit
place to store all the different versions of ``bytestring``. But as the
maintainer, you could as well have decided to store the package
elsewhere instead. Why do we need to force the whole package universe
into Hackage? Now we don't need to. People can host packages on Hackage
as before, they can host them directly on Stackage, or indeed on their
own publicly available corporate server.

Packages can also be hosted *directly on GitHub*.
Anyone in the community is free to implement a proxy service that maps
calls to the “package API” above to downloads of GitHub-hosted
repository snapshots. For that, all a user needs to do is claim a new
package name on SLURP, once, pointing to the proxy service. Then, the
proxy service can advertise one package version per Git tag found in the
repository. Publishing a new (GPG signed!) package version can be done
with a simple

.. code:: console

    $ git tag -S v1.2.3
    $ git push --tags origin master

Each package repository (Hackage, Stackage, SomeOtherPackageHost etc) is
free to define their own policies. Hackage can choose to stick to
mutable packages as they do now. Stackage can decide that all packages
uploaded there are always immutable. Hackage can decide to reject
packages that don't stick to the PVP. Stackage can impose their own
rules.

6.2 Liberal namespacing
~~~~~~~~~~~~~~~~~~~~~~~

There have been many suggestions made offline in recent months about
introducing a hierarchical package namespace for various purposes. One
such purpose could be to cordon off experimental packages into a separate
``experimental/`` namespace before they make it into the main namespace.
Another might be to identify a set of packages specific to
an as-yet-unreleased version of GHC.

This proposal is compatible with future extensions to the namespace as
served currently by Hackage. But we do not propose to make any changes
to this namespace yet. That is a matter for future proposals.

6.3 Minimal changes to Hackage
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Due to the federation, one small change is necessary to
``hackage-server``, the implementation behind hackage.haskell.org.
Before accepting a new package, ``hackage-server`` must first
successfully register the package name on SLURP. If it cannot, then it
refuses the package upload. This is the only change necessary to
``hackage-server``.

It would be highly desirable that packages available elsewhere are also
available to legacy cabal-install versions. To achieve this,
``hackage-server`` is free to include ``.cabal`` files in the metadata
bundle that correspond to package versions hosted elsewhere but
registered via SLURP. Just like Stackage today imports package versions
available from Hackage.

6.4 No need for extra user accounts
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

No user accounts are needed for making API calls to SLURP. SLURP is
completely platform neutral. Platforms such as Hackage or Stackage can
make SLURP API calls on behalf of users if they wish. They can choose to
do so on the condition that users have an account on said platform, or
not. Users can also make API calls directly.

How can this work? The API does not permit changing any existing
``pkgname->URL`` mapping in SLURP. Only adding new ones. Therefore, no
existing package author can be adversely affected by the actions of any
new package author.

There is a danger of name squatting, i.e. users preemptively claiming
names while not writing any package. This danger already exists for
Hackage itself. The solution for dealing with that is exactly the same
as for Hackage: define a policy for what constitutes a Haskell package,
what constitutes name squatting, and appoint a small group of trustees
to enforce the policy. Initially, the trustees can be the union of
Hackage trustees and Stackage curators. No matter the chosen policy, the
implementation of SLURP and the API remain the same.

More than just name squatting, just like for Hackage, it’s conceivable
that bad actors could run scripts to carpet bomb the package namespace.
It’s easier to do so on SLURP than on Hackage (no user accounts). But in
practice, this hasn’t been a major problem for other package managers
with similar designs (e.g. Bower).

7 Conclusion
------------

With SLURP, no service is *the* privileged place in the Haskell
universe for hosting packages. Different actors in the community are
free to build more package hosting services, without adversely affecting
any of the existing ones or breaking backwards compatibility with
existing build tools, all the while preserving two crucial properties:

-  Package names remain independent of where they are hosted or who is
   currently maintaining them.
-  Package names uniquely and unambiguously map to a single endpoint
   where the available versions can be listed and package tarballs
   downloaded.

This design is hardly new and has been validated on a larger scale than
the Haskell community in the past. Bower for JavaScript (and also used
by PureScript) has grown to about 60k packages, likewise with a registry
of pointers to federated package repositories, and likewise doing so
without requiring authentication. OPAM, the now dominant package manager
in the OCaml community, features similar federation. The registry is
stored in a Git repository and package additions are updates are
uniformly managed via pull requests on GitHub.

8 What will happen if we do nothing?
------------------------------------

If we do nothing it seems likely that the Stack community will create a
separate Haskell package repository. No one wants this, because it
imposes heavy costs on Haskell users:

-  We’d get conflicts when the two repos used the same package name for
   different packages
-  Users would have two repositories to search in
-  Currently a package that builds with stack can, with minor effort, be
   made to build using cabal too. With two completely separate
   repositories, the two will diverge and this cross-ecosystem transfer
   will become much more difficult.
-  The meta-data format (i.e. the .cabal file) syntax might well start
   to diverge, which would make life harder for users.
-  Source-code mining tools (e.g search, discovery, etc) would have two
   places and perhaps two metadata formats to handle.
-  Since each would doubtless seek to mirror packages in the other,
   there would be much duplication and (more damagingly)
   near-duplication, which would make life harder for both users and
   tooling.

Perhaps, after a long time, one ecosystem or another will become
de-facto dominant. But there would be considerable pain to thousands of
Haskell users meanwhile.

That is terrible! We have so much in common.

-  Everyone wants Haskell to succeed and be widely used
-  Everyone wants Haskell users to have a frictionless experience of
   using libraries written by others
-  Everyone is trying hard to do the Right Thing

That makes it tantalising that we are having difficulty finding a
solution. But it also makes us optimistic that we can find one if we
work together.

Appendix A: SLURP governance and terms of use
---------------------------------------------

The below is an example policy document. It should be discussed
independently of the technical proposal above: none of the below imposes
any additional constraints on the implementation.

Governance
~~~~~~~~~~

SLURP will be governed by the *SLURP trustees*.

The SLURP trustees include representatives from the various stakeholders
whose composition will change as the ecosystems around SLURP changes.
Initially, we propose the SLURP trustees to be comprised of (1) one
representative of the Hackage maintainers, (2) one representative of the
Stackage curators, and (3) one member-at-large from the general Haskell
community. The number of trustees can change over time. The
member-at-large will chair the group of trustees.


Changes to the composition of the SLURP trustees are decided by the SLURP trustees and need to be unanimous. All other decisions are made by simple majority vote if there is a dispute among the trustees.
This governance mechanism is, by design, lightweight; for example, the number of trustees is small.  If, in the light of experience, it seems that a more substantial mechanism is needed, we can revisit the issue as a community.

Terms of use
~~~~~~~~~~~~

Users of SLURP agree to abide by the rules stated in the following. IP
addresses or IP ranges of users who repeatedly violate the terms of use
may be blocked. This decision is made by the SLURP trustees.

§1 Valid SLURP entries
^^^^^^^^^^^^^^^^^^^^^^

Any entry added with ``PUT /package/:pkgname`` that maps the new package
name *pkgname* to the URL *url* must abide by the following
restrictions:

-  Both the *pkgname* and the *url* must not already exist in the
   registry.
-  The *pkgname* must not be unlawful, threatening, abusive, libelous,
   defamatory, obscene, offensive, indecent, pornographic, profane, or
   otherwise objectionable.
-  The endpoint identified by *url* must conform to the specification
   given in the section entitled ”The package API”, and must continue to
   do so over time. Endpoints that no longer conform to the
   specification are liable to removal from the SLURP registry after one
   month.
-  The tarballs that can be obtained via that package API need to be
   genuine Haskell packages. This means that the tarballs must include
   package metadata and a build file that conforms to Cabal-the-spec.
   This metadata must (1) identify the packages as *pkgname* and (2)
   include valid license and copyright information.

§2 Squatting
^^^^^^^^^^^^

Clients may not put undue load on the SLURP server. If the package list
needs to be accessed frequently, it ought to be cached locally. 
Spamming SLURP with package entries is prohibited. A package entry is considered
spam if the none of the versions available at the given location have
any genuine function. This judgement is made by a human, and attempts to
"game" squatting by making pseudo-functional packages will increase, not
decrease, the likelihood that the SLURP trustees will transfer the
package to a user who requests it.

§3 Changing SLURP entries
^^^^^^^^^^^^^^^^^^^^^^^^^

Generally, the SLURP registry is append-only; entries do not change.

Exceptionally, they may be changed, but that requires human
intervention. Requests for change of a package entry are made by email
to an address designated by the SLURP trustees. The email needs to
include a reason for the change and the new entry needs to conform to
the restrictions in §1. The final decision of whether to change an entry
lies with the SLURP trustees.
