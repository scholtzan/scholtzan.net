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
* `index.js` as the Node.js entrypoint for the plugin.
* `package.json` specifies `homebridge` and `npm` as `engines` and `node-fetch` as dependency required for making HTTP requests to the Spotify API.

## Registering the Plugin

The `index.js` file serves as the plugin entrypoint. Here, the generated JavaScript `homebridge_rusty_spotify.js` is imported and exposes `SpotifyPlatform` which contains the platform plugin core logic that is written in Rust and compiled to WebAssembly. 

```javascript
// import generated WebAssembly
const SpotifyPlatform = require('./homebridge_rusty_spotify.js').SpotifyPlatform;

function partial(fn /*, rest args */){
  return fn.bind.apply(fn, Array.apply(null, arguments));
}

module.exports = function(homebridge) {
  // expose types that will be used in the Rust code  
  Accessory = homebridge.platformAccessory;
  Service = homebridge.hap.Service;
  UUIDGen = homebridge.hap.uuid;
  Characteristic = homebridge.hap.Characteristic;

  // global function for creating a new switch accessory
  createSwitch = function(name) {
    let newSwitch = new Service.Lightbulb(name);
    // we'll use brightness to control the volume
    newSwitch.addCharacteristic(Characteristic.Brightness);
    return newSwitch;
  }

  // register platform plugin
  constructor = partial(SpotifyPlatform, homebridge);
  homebridge.registerPlatform("homebridge-rusty-spotify", "Spotify", constructor, true);
}
```
*`index.js`*

A few Homebridge specific types, such as `Accessory`, `Service`, `UUIDGen` and `Characteristic` need to be exported in order for them to be imported and used in Rust. `createSwitch` needed to be provided to initialize a new `Lightbulb` service which represents a Spotify device accessory. This was necessary as a work around since nested JavaScript classes are not supported with `wasm-bindgen` at the moment. 

It is also noteworthy that individual Spotify devices are instantiated as a `Lightbulb` accessory and not as, for example, a `Speaker` accessory. When trying to change the accessory type to `Speaker`, HomeKit issues a warning: _"This accessory is not certified and may not work reliably with HomeKit"_ and the accessory becomes unresponsive. The `Lightbulb` accessory type also has the advantage the a `Brightness` characteristic can be added which can be used to control the playback volume on the device.

`SpotifyPlatform` is a class that is written in Rust and compiled to WebAssembly. It has an additional `homebridge` parameter which is used to interface with the Homebridge API. `SpotifyPlatform` needs to be partially applied to pass `homebridge` as a parameter and then use the partially applied class constructor when registering the platform plugin to Homebridge using [`registerPlatform`](https://developers.homebridge.io/#/api/platform-plugins#apiregisterplatform).


## Platform Plugin - `SpotifyPlatform`

The platform plugin is written in Rust in `src/spotify_platform.rs`. 


## Platform Accessories - `SpotifyAccessory`


## Making Requests to the Spotify API - `SpotifyApi`


## Configuring the Plugin


## Building, Publishing and Installing the Plugin


## Issues and Future Work


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