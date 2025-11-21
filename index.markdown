---
layout: post 
---

<p style="font-size: 72px ; text-align: center">
<img src="images/rust-logo-blk.svg" height="150px" alt="Rust"> + <img src="images/cheri-logo-larger.png" height="150px" alt="CHERI">
</p>

## Latest news

{% assign posts = site.posts %}
{% for item in posts limit:4 %}
  {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
 - [*{{ item.title }}* - {{ item.date | date: date_format }}]({{ item.url }})
{% endfor %}

[More...](news)

## About

Rust and CHERIoT provide complementary guarantees.
Rust provides a rich set of properties that can be enforced at compile time.
These go far beyond memory safety, and can enforce rich invariants.
CHERIoT provides a rich compartmentalisation model and object-granularity memory safety for all objects, from assembly code on up to higher-level languages.

The combination will mean that you can use CHERIoT compartments for supply-chain safety in Rust and to allow Rust code to interoperate with existing C/C++ with dynamic guarantees that the C/C++ cannot violate any of the invariants that the Rust code depends on.

This code is being developed in the [CHERIoT-Platform fork of Rust](https://github.com/CHERIoT-Platform/cheri-rust).
It uses [CHERIoT LLVM](https://github.com/CHERIoT-Platform/llvm-project), which also includes a set of CHERI and CHERIoT-specific [clang static analyser](https://clang-analyzer.llvm.org) analyses that will evolve in tandem with this to make it easier to check C/C++ code that needs to interoperate with Rust in a CHERIoT firmware image.

## Where to ask questions

We use [GitHub Discussions](https://github.com/orgs/CHERIoT-Platform/discussions) for general queries about CHERIoT.
This is persistent and searchable (without an account) and so a good place to ask questions that someone else may want to know the answer to.

We also have a [public Signal chat](https://signal.group/#CjQKIElxAs3t3MUEMOEmQEuMHRK4rErUk2xVeFzjAjFXAShzEhCK9qQwEMFKGLGZnCjrQ7zm).
The Signal chat is intended for live discussions and automatically deletes messages.
We encourage participants to write up the results of any discussions there in documentation, GitHub Discussions, Issues, or somewhere else that's searchable.
You can join the group from your phone by scanning this QR code:

<p style="text-align: center; margin-left: auto; margin-right: auto">
<img src="images/signal-group-qr-code.png" width="100pt" alt="QR Code for joining the CHERIoT Signal public chat">
</p>
