- Feature Name: Packages as Namespaces
- Start Date: 2018-10-25
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

A single flat namespace on `crates.io` reduces the friction associated
with using and promoting packages, helping the rust ecosystem to flourish.
Unfortunately, the flat namespace makes package names a precious resource,
and therefore a source of conflict. This document proposes to add optional
package based namespaces to rust and its registry infrastructure to alleviate
the potential for future conflict while retaining the advantages of the
current system.

# Motivation
[motivation]: #motivation

Package name squatting has become a point of contention within the community,
cropping up repeatedly on forums like `/r/rust` and `internals.rust-lang.org`.
While namespaces are not the same issue as squatting, they are often proposed
as a solution to the specific case where one project wishes to publish multiple
related packages without fear of them being squatted on. A particular complaint
with the current system is that project authors who wish to be good citizens and
avoid squatting are incentivized to squat on names related to their project.

This proposal aims to be backwards compatible with the existing
system. Beyond just technical backwards compatibility, this proposal attempts
to provide aesthetic backwards compatibility. Subcrates are a natural extension
of the existing system. The pattern of prefixing helper crates
with `<main crate>-` that has emerged demonstrates that the creation of
subcrates is already a common pattern.

TODO(ethan): The forums lied to me? How?
For an example of such "good faith squatting" see the 

Protection from squatting may be the primary motivation for package
maintainers, but there is another reason to adopt it from
the perspective of package consumers. An important part of developing
high quality software is ensuring that the libraries you depend on
are of equally high quality. Allowing packages to define sub-packages
provides a way to signal shared branding to users. A user may wish to
spend more effort auditing `foo-bar`, a crate developed to work with
`foo` but not written by the authors of `foo`, than they would like
to spend auditing `foo::>baz`, a crate developed by the already
trusted authors of `foo`. This will reduce the research burden associated
with pulling in crates that are part of a larger ecosystem. Note that this
motivation is the weakest of the motivations presented here because it
is already possible to determine crate authorship.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Crate Names

A crate name consists of 1 or more rust identifiers separated
by the crate path separator, `::>`.
Any crate may define itself to be a subcrate of 0 or more crates by
giving itself a name of the form
`parent1::>parent2::>...::>parentn::>crate_name`.
Each prefix of the name which begins and ends with an identifier
should name another crate. Rust the language does not make an
effort to enforce this, but `crates.io` does.
A crate name containing the crate path separator
can be used in all the same places that a crate name without
the crate path separator can. For example, to define a crate
`foo` which is a subcrate of `bar`, we would place this in the
`Cargo.toml`

```
name = "bar::>foo"
```

In order to import `bar::>foo`, we would write

```
extern crate bar::>foo;
```

in `lib.rs` or `main.rs` just as we would for a crate without the crate
path separator in its name.

Crate names can also appear as part of expressions. Again, the rules for
where a crate name with a crate patch separator can appear are exactly the
same as the rules for where a crate name without a separator can appear.
If the crate `bar::>foo` exports the struct `Button` which defines a `new`
associated function, we could write `bar::>foo::Button::new()` to create
a new `Button`. Similarly, we could first `use bar::>foo;`, then refer to
`Button` directly with its unqualified name.

## Subcrates

A crate, `foo::>bar`, is called a subcrate of the crate `foo`.

## Supercrates

A crate, `foo`, is called the supercrate of the crate `foo::>bar`.

## Uploading a crate to crates.io

In order to upload a subcrate to `crates.io`, you must own its parent crate.
For example, if you already maintain the `button` crate in the top level
namespace, you can upload `button::>click` to `crates.io`. If you don't
own the `button` crate you cannot upload any subcrates of `button`. Attempting
to do so will result in a permission error. If you don't own the top level
`button` crate, but are one of the owners of its subcrate `button::>click`,
you may upload the crate `button::>click::>mobile`. This allows package
ecosystems to delegate responsibility for the maintenance of different
subsystems.

TODO: sample error messages

# Teaching packages-as-namespaces to existing rust developers

This proposal aims to be backwards compatible with the existing
system, so existing rust developers can continue using their current
workflow. The biggest change for a rust package consumer is that
some of the crates they depend on will now include the crate path
separator. The biggest change for a rust package maintainer is that
they will now have the option of uploading helper crates as subcrates
instead of following the established convention of using
`<main crate>-` as a prefix. It is worth noting that some crate ecosystem
maintainers see the fact that their own helper crates and those published
by third parties look the same as a good thing. These ecosystems will
be able to continue on as before.

# Teaching packages-as-namespaces to new rust developers

Other language ecosystems have namespaces for their package systems.
Namespaced crates will probably not be the most common case, so they
should show up 
TODO

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Lexer

The lexer will gain a new token representing the crate path
separator

```
::> { return CRATE_SEP; }
```

This will mean that the string "::>" will no longer lex as
the token sequence `MOD_SEP, '>'`, which will cause some compiler
error messages to change. This will not be a breaking change because
`:: >` was not previously syntactically valid.

## Grammar

The `path_expr` production will gain a new alternative (shown at the end)
so that it looks like

```
path_expr
: path_generic_args_with_colons
| MOD_SEP path_generic_args_with_colons
| SELF MOD_SEP path_generic_args_with_colons
| crate_path MOD_SEP path_expr
;
```

where `crate_path` is defined as

```
crate_path
: ident
| crate_path CRATE_SEP ident
```

## Name Mangling

`:` and `>` are both valid symbols in a rust symbol before it is mangled today,
so mangling would proceed in a similar fashion.

## Crates.io Ownership

When a subcrate is created it will start off with all the same owners
as the supercrate. The owners of the supercrate may then add or remove
owners to the list of subcrate owners. For example, if Alice owns
`foo` she could create the crate `foo::>bar` and add Bob to the list
of owners. At this point `foo::>bar` would be owned by both Alice
and Bob. Alice could choose to remove herself from the owners list
of `foo::>bar` to signal that Bob is the subcrate's primary maintainer
(though nothing would stop her from adding herself back to
the owners list at a later date).

# Drawbacks
[drawbacks]: #drawbacks

The biggest drawback of this proposal is that it increases the complexity of
the module system, which is already one of the areas that newcomers to the
language struggle with.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are a number of different approaches to adding namespaces to
a package ecosystem.

## Domains as Namespaces

One approach to adding namespaces to the package ecosystem is to allow
domains to be used as namespaces. A big advantage of this approach
is the ability to offload a lot of work to an existing ecosystem.
We would be able to rely on ICANN to handle issuing new namespaces
and resolving disputes over their ownership. Unfortunately,
the use of a pre-existing system of namespace registration would
also come with significant downsides.

Domains under programmers control rarely directly relate to the packages
they have published. Prominent counterexamples like `diesel.rs` do exist,
but projects without a dedicated website would end up with names such as
`burntsushi.net/regex-syntax` or `burntsushi.github.io/regex-syntax`
instead of `regex::>syntax`. Only the second half of the first two
names tells us much about what the package *actually does*.

Domains must be paid for in order to retain ownership, and people do
allow their domain registrations to laps, sometimes without meaning
to. This would be inconvenient for developers who lost control of a
domain, but wished to continue maintaining their packages published
under that domain or publish new packages in the same namespace.

There are a number of top level domains, and it is difficult to own
a project-related subdomain on all TLDs. As a user it is not
necessarily easy to determine whether `diesel.rs/diesel` or
`diesel.io/diesel` is the official package. As the rust ecosystem
grows domains as namespaces will make choosing the right package
to depend on tricky.

## A single flat namespace without an arbitration policy

This is the situation as it stands now. The solution can scale
to a large package ecosystem as evidenced by the success of
rubygems.org. The main benefit is that most packages end up
with shorter, more memorable names. This is especially beneficial
when a community is new and still trying to build up steam.
As the package ecosystem grows, names in the single flat
namespace become a precious commodity. Disputes over who ought
to control a name will arise, and without an arbitration policy
the name will stay with whoever got there first.

Larger rust projects like `actix`, `diesel`, `piston` and
`amethyst` often end up publishing a host of supporting
crates. In a single flat namespace, these projects are forced
to take up a larger footprint, even if they would prefer to
occupy a single name.

## A single flat namespace with an arbitration policy

This is the approach taken by npm and pypi. The biggest drawback
of this approach is the amount of effort that is required in order
to administer the arbitration policy.

## The impact of not doing this

TODO


# Prior art
[prior-art]: #prior-art

rubygems.org uses a single flat namespace, pypi uses a flat
namespace with an arbitration policy, the jvm ecosystem has support
for namespaces and a convention of using domains as namespaces, and
npm has a top level flat namespace plus scopes which allow some packages
to be namespaced. npm's scope system is the closest to the system proposed
in this document. The pros and cons of the different systems are discussed
in more detail in the [rationale and alternatives][rationale-and-alternatives]
section.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

An arbitration policy for package name disputes is out of scope for
this RFC.

The decision to introduce a crate path separator into the rust naming
scheme is not the only possible path for this proposal. One alternative
design is to keep rust-the-language exactly the same as it is today.
A crate such as `foo::>bar` would be referred to with the name `bar`
in any rust source code. This has the nice benefit that it is easy to
swap out implementations of `bar` with only configuration changes.
The main drawbacks are reduced explicitness in the code, and the fact
that it is not possible to use both `router` and `actix::>router` in
the same projects without aliasing one of them. Additionally, including
the namespace in rust identifiers opens the door to relaxing the restriction
on where traits can be implemented. Even with these drawbacks, the ability
to swap out implementations and the fact that such an approach would be much
less invasive may be more desirable.

I would like help making the section on name mangling more precise before this
gets merged.
