+++
template = "page.html"
+++


## Social

* [GitHub](https://github.com/luryus)
* [Mastodon](https://mastodon.social/@luryus)
* [LinkedIn](https://www.linkedin.com/in/laurikoskela/)


## My projects

{% project_card() %}
#### [wden](https://github.com/luryus/wden)

A password manager TUI for Bitwarden-compatible servers. A lot more lightweight than official Bitwarden client apps. Written in Rust using the [Cursive](https://github.com/gyscos/cursive) framework.
{% end %}


{% project_card() %}
#### [Wattiviisari](@/wattiviisari/index.md)

A Wear OS (Android Wear) app for viewing Finnish electricity spot prices.
{% end %}


{% project_card() %}
#### [RP-FC Footswitch](https://github.com/luryus/rp-fc)

A GA-FC footswitch clone for Boss Katana amps, built around a Raspberry Pi Pico using embedded Rust.
{% end %}


{% project_card() %}
#### Raspberry Pi Pico Macropad

I built a Raspberry Pi Pico based macropad for my own use. It has a completely [custom firmware](https://github.com/luryus/pico-macropad) that handles the keyboard, rotary encoders, the user interface, and USB communication with the host. It works together with a [Python-based server program](https://github.com/luryus/macropadd) that handles application-specific shortcuts and macros.
{% end %}


{% project_card() %}
#### [udp-bcast-relay-rs](https://github.com/luryus/udp-bcast-relay-rs)

A Rust reimplementation of the [udp-broadcast-relay](https://github.com/nomeata/udp-broadcast-relay) project. Used in [my fork of the ubnt-bcast-relay project](https://github.com/luryus/ubnt-bcast-relay) for Ubiquiti EdgeRouters. The upstream project ships prebuilt binaries that I'm not keen on installing to my routers. And cross-compiling the C project to the MIPS architecture on EdgeRouter X is non-trivial. So I ended up rewriting it in Rust just because that was more fun than setting up the build environment for the upstream project.
{% end %}


{% project_card() %}
#### More projects

[https://github.com/luryus](https://github.com/luryus)
{% end %}