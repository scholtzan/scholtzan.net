+++
title = "Writing Homebridge plugins in Rust"
date = "2019-11-27"
type = "post"
tags = ["Rust", "WebAssembly", "Homebridge", "Spotify", "Open Source"]
+++

[Homebridge](https://homebridge.io/) is an awesome way to integrate smart devices that do not have native HomeKit support and has been running on my Raspberry Pi for months. There are thousands of [plugins](https://www.npmjs.com/search?q=homebridge-plugin) available, however, one that I was missing was a plugin to control Spotify devices. 

Homebridge plugins are Node.js modules which are published through NPM and usually written in JavaScript or TypeScript. While I could have picked up either of these languages for writing a plugin, I wondered if it would be possible to develop plugins in Rust instead. After all, with the tools provided by the [Rust and WebAssembly ecosystem](https://rustwasm.github.io/) it should be possible to [compile plugins to WebAssembly](https://github.com/rustwasm), [publish them to NPM](https://github.com/rustwasm/wasm-pack) and install them with Homebridge.

The short answer is: Yes, it is possible. Although there were a lot of challenges which I'll describe in more detail in the following. All of the source code is available as part of the [homebridge-rusty-spotify repository](https://github.com/scholtzan/homebridge-rusty-spotify).

## Plugin Setup

The project setup for the Spotify plugin is very similar to the typical Rust project setup:
* `Cargo.toml` contains metadata and dependencies. A `[lib]` annotation needs to be added to specify `crate-type = ["cdylib"]`. This signifies that a `.wasm` file should be generated. [`wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen) needs to be added as dependency to allow importing JavaScript code and exporting Rust code. Additionally, [`wasm-bindgen-futures`](https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/) is required to convert between JavaScript `Promise`s and Rust `Future`s. This is necessary when making requests to the Spotify REST API and processing responses in Rust. The [`web-sys`](https://rustwasm.github.io/wasm-bindgen/web-sys/index.html#the-web-sys-crate) crate imports raw binding for the Web's API and the [`js-sys`](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/js-sys) crate imports raw bindings to global JavaScript APIs.
* `src/lib.rs` as the crate root.
* `index.js` is the Node.js entrypoint for the plugin.
* `package.json` [todo]

## Registering the Plugin


Here, the generated JavaScript `homebridge_rusty_spotify.js` is imported and exposes `SpotifyPlatform` which will be used to register the platform plugin.



* Homebridge, plugins
    * Homebridge server also written in Node.js
* Spotify
* why rust
    * rust wasm

* registering the plugin
    * JavaScript
* registering accessories
    * Arrays
* service and characteristics
    * callbacks
* API to make Spotify requests
* Configuration Spotify

* future work, things that don't quite work
* devices disconnecting when inactive

* building the plugin
    * installing