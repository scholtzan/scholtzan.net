+++
title = "Writing Homebridge plugins in Rust"
date = "2020-11-29"
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
* `index.js` as the Node.js entry point for the plugin.
* `package.json` specifies `homebridge` and `npm` as `engines` and `node-fetch` as dependency required for making HTTP requests to the Spotify API.

## Registering the Plugin

The `index.js` file serves as the plugin entry point. Here, the generated JavaScript `homebridge_rusty_spotify.js` is imported and exposes `SpotifyPlatform` which contains the platform plugin core logic that is written in Rust and compiled to WebAssembly. 

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

A few Homebridge specific types, such as `Accessory`, `Service`, `UUIDGen` and `Characteristic` need to be exported for them to be imported and used in Rust. `createSwitch` needed to be provided to initialize a new `Lightbulb` service which represents a Spotify device accessory. This was necessary as a workaround since nested JavaScript classes are not supported with `wasm-bindgen` at the moment. 

It is also noteworthy that individual Spotify devices are instantiated as a `Lightbulb` accessory and not as, for example, a `Speaker` accessory. When trying to change the accessory type to `Speaker`, HomeKit issues a warning: _"This accessory is not certified and may not work reliably with HomeKit"_ and the accessory becomes unresponsive. The `Lightbulb` accessory type also has the advantage that a `Brightness` characteristic can be added which can be used to control the playback volume on the device.

`SpotifyPlatform` is a class that is written in Rust and compiled to WebAssembly. It has an additional `homebridge` parameter which is used to interface with the Homebridge API. `SpotifyPlatform` needs to be partially applied to pass `homebridge` as a parameter and then use the partially applied class constructor when registering the platform plugin to Homebridge using [`registerPlatform`](https://developers.homebridge.io/#/api/platform-plugins#apiregisterplatform).


## Platform Plugin - `SpotifyPlatform`

The platform plugin is written in Rust in `src/spotify_platform.rs`. At the top of the file, Homebridge API function used for registering plugin accessories and configuring plugins are imported as well as the [`setInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setInterval) method which is used for checking for new Spotify devices periodically.

The `configureAccessory()` method is invoked whenever Homebridge tries to restore accessories that are cached in `~/.homebridge/accessories/cachedAccessories`. Restoring happens, for example, after restarting Homebridge. The plugin will determine these cached accessories and remove them once the plugin has been initialized. This ensures that Spotify devices that are no longer available do not get registered as accessories.

To ensure Spotify device accessories are as up-to-date as possible, `setInterval` is used to periodically call the `refresh_devices` function. This function defines a closure that will retrieve all active Spotify devices from the Spotify API, store them and remove those that are no longer available.

```rust
fn refresh_devices(&mut self) {
    // [...]
    let refresh_closure = Closure::wrap(Box::new(move || {
        spawn_local(async move {
            // [...]
            // make API request
            let available_devices: SpotifyDevices =
                match JsFuture::from(api.get_devices()).await {
                    Ok(state) => state.into_serde().unwrap(),
                    Err(_) => SpotifyDevices {
                        devices: Vec::new(),
                    },
                };

            // check if devices still exist
            devices.borrow_mut().retain(|registered_device| {
                if !available_devices
                    .devices
                    .iter()
                    .any(|d| &d.id == registered_device.get_device_id())
                {
                    // Array.of()
                    let accessories =
                        PlatformAccessories::of(registered_device.get_accessory());

                    homebridge.unregister_platform_accessories(
                        PLUGIN_IDENTIFIER,
                        PLATFORM_NAME,
                        accessories,
                    );

                    return false;
                }
                true
            });

            // check if device already exists, otherwise add
            for available_device in available_devices.devices {
                if !devices
                    .borrow()
                    .iter()
                    .any(|d| d.get_device_id() == &available_device.id)
                {
                    let accessory = SpotifyAccessory::new(
                        available_device.name,
                        available_device.id,
                        api.clone(),
                    );

                    // Array.of()
                    let accessories = PlatformAccessories::of(accessory.get_accessory());

                    homebridge.register_platform_accessories(
                        PLUGIN_IDENTIFIER,
                        PLATFORM_NAME,
                        accessories.clone(),
                    );

                    devices.borrow_mut().push(accessory);
                }
            }
            // [...]
        });
    }) as Box<dyn FnMut()>);

    // define setInterval()
    let _ = set_interval(
        refresh_closure.as_ref().unchecked_ref(),
        self.config.refresh_rate.unwrap_or(REFRESH_RATE),
    );
    refresh_closure.forget();
}
```
*`src/spotify_platform.rs` `refresh_devices()` excerpt*

The Homebridge API function [`registerPlatformAccessories()`](https://developers.homebridge.io/#/api/platform-plugins#apiregisterplatformaccessories) for registering new accessories and [`unregisterPlatformAccessories()`](https://developers.homebridge.io/#/api/platform-plugins#apiunregisterplatformaccessories) for unregistering accessories expects a list of accessories as parameter. However, currently `wasm-bindgen` [does not support returning `Vec<T>` or having parameters of type `Vec<T>`](https://github.com/rustwasm/wasm-bindgen/issues/111). So, importing `registerPlatformAccessories()` as was not an option:

```rust
    #[wasm_bindgen(method, js_name = registerPlatformAccessories)]
    fn register_platform_accessories(
        this: &Homebridge,
        plugin_identifier: &str,
        platform_name: &str,
        accessories: Vec<SpotifyAccessory>,     // not supported
    );
```

Also, using [`js_sys::Array`](https://rustwasm.github.io/wasm-bindgen/api/js_sys/struct.Array.html) as parameter type did not work, because array elements get converted to `JsValue`s.

The workaround I ended up doing was to import the JavaScript `Array` type and its constructor under a different name and use this one instead for creating an array of `SpotifyAccessory`s:

```rust
    #[derive(Clone, Debug)]
    #[wasm_bindgen(js_name = Array)]
    pub type PlatformAccessories;

    #[wasm_bindgen(constructor, js_class = "Array")]
    fn of(accessory: &Accessory) -> PlatformAccessories;

    #[wasm_bindgen(method, js_name = registerPlatformAccessories)]
    fn register_platform_accessories(
        this: &Homebridge,
        plugin_identifier: &str,
        platform_name: &str,
        accessories: PlatformAccessories,
    );
```

## Platform Accessories - `SpotifyAccessory`

Each available Spotify device is represented as a `SpotifyAccessory` which is defined in `src/spotify_accessory.rs`. Similarly to `spotify_platform.rs`, first all JavaScript function that are used at some point are imported. When a new accessory gets instantiated, first a new Homebridge service is created using the `createSwitch()` function that is defined in `index.js` and invoked here. Next, characteristics, such as `On` for starting and stopping Spotify playback and `Brightness` for changing the volume, are assigned to the service. Also, a name gets assigned that corresponds to the name of the Spotify device.

```rust
fn apply_characteristics(&self) {
    let get_on = self.get_on();
    let set_on = self.set_on();

    self.service
        .get_characteristic("On")
        .on("set", set_on.as_ref().unchecked_ref())
        .on("get", get_on.as_ref().unchecked_ref());

    let get_volume = self.get_volume();
    let set_volume = self.set_volume();

    self.service
        .get_characteristic("Brightness")
        .on("set", set_volume.as_ref().unchecked_ref())
        .on("get", get_volume.as_ref().unchecked_ref());

    self.service
        .get_characteristic("Name")
        .set_value(&self.name);

    get_on.forget();
    set_on.forget();
    set_volume.forget();
    get_volume.forget();
}
```
*Defining accessory characteristics*

The methods invoked when switching the accessory on or off or when changing the volume are defined as closures: 

``` rust
fn set_on(&self) -> Closure<dyn FnMut(bool, Function)> {
    let api = Rc::clone(&self.api);
    let device_id = self.device_id.clone();

    Closure::wrap(Box::new(move |new_on: bool, callback: Function| {
        if new_on {
            let _ = api.play(&device_id);
        } else {
            let _ = api.pause(&device_id);
        }

        callback
            .apply(
                &JsValue::null(),
                &Array::of2(&JsValue::null(), &JsValue::from(new_on)),
            )
            .unwrap();
    }) as Box<dyn FnMut(bool, Function)>)
}
```
*Closure for starting/pausing Spotify that is part of `SpotifyAccessory`*

Each of these methods make requests to the Spotify API and use `SpotifyApi` for this.


## Making Requests to the Spotify API - `SpotifyApi`

`SpotifyApi` is defined in `src/spotify_api.rs` and contains methods for making requests to the [Spotify API](https://developer.spotify.com/documentation/web-api/). For executing HTTP requests, [node-fetch](https://github.com/node-fetch/node-fetch) is used. It also needs to be installed for the plugin to run. `src/node_fetch.rs` defines a `fetch` method which is used to issue requests using node-fetch and to return API responses to the caller as JSON.

To access private information through the Spotify Web API and to control the music playback, the plugin needs to be authorized via the [Spotify Accounts service](https://accounts.spotify.com/). This also requires users to have a Spotify Premium account and makes the plugin installation and configuration a bit more complicated. Users have to generate a client ID and client secret that needs to be provided in the plugin configuration file. These credentials are used in `authorize()` to authenticate to the Spotify API every time a request is made. `authorize()` returns an access token which needs to be included as bearer token for every request.

`SpotifyApi` provides separate functions for each request. All of them look quite similar:

```rust
pub fn get_devices(&self) -> Promise {
    let authorize_request = self.authorize();

    future_to_promise(async move {
        match JsFuture::from(authorize_request).await {
            Ok(authorize_request) => {
                let access_token: String = authorize_request.as_string().unwrap();
                let url = "https://api.spotify.com/v1/me/player/devices";

                let authorization_header = format!("Bearer {}", access_token);
                let mut headers = HashMap::new();
                headers.insert("Authorization".to_owned(), authorization_header);

                match fetch(url, FetchMethod::Get, "", headers, false).await {
                    Err(e) => {
                        console::log_1(&format!("Error getting devices: {:?}", e).into())
                    }
                    Ok(result) => {
                        return Ok(result);
                    }
                }
            }
            Err(e) => console::log_1(
                &format!("Error while authenticating to Spotify API: {:?}", e).into(),
            ),
        }

        Ok(JsValue::null())
    })
}
```
*Function requesting available Spotify devices.*

## Configuring the Plugin

To configure the plugin and to make it run in Homebridge, it needs to be registered as an app in the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard/login) where the following steps need to be executed:
+ Select "Create a client ID"
+ Provide a name and description in the pop-up; click "Next"
+ Copy the "Client ID" and "Client Secret" which will be required in the following configuration step
+ Click "Edit Settings"
+ Add `http://localhost/callback` as "Redirect URI" and save

Next, the plugin configuration needs to be generated. The `generate_config` script can be used to create this configuration. It requires for the client_id, client_secret and Spotify username to be set since those are required to authenticate to the Spotify Web API. Running the script will open a web browser asking to authenticate to Spotify which is required to retrieve the `refresh_token`.

```bash
$ ./generate_config --help
usage: generate_config [-h] [--client_id CLIENT_ID]
                       [--client_secret CLIENT_SECRET]
                       [--redirect_uri REDIRECT_URI] [--username USERNAME]

Script to retrieve the access and refresh token for using the Spotify API

optional arguments:
  -h, --help            show this help message and exit
  --client_id CLIENT_ID, --client-id CLIENT_ID
                        Spotify client ID
  --client_secret CLIENT_SECRET, --client-secret CLIENT_SECRET
                        Spotify client secret
  --redirect_uri REDIRECT_URI, --redirect-uri REDIRECT_URI
                        Redirect URI
  --username USERNAME   Spotify username


$ ./generate_config --client_id=<client_id> --client_secret=<client_secret> --username=<username>
  {
    "platform": "Spotify",
    "name": "Spotify",
    "client_id": "<client_id>",
    "client_secret": "<client_secret>",
    "refresh_token": "<refresh_token>"
  }
```

The generated config needs to be copied to the Homebridge config file (e.g. `~/.homebridge/config.json`).

## Building, Publishing and Installing the Plugin

For building the plugin, the `Makefile` can simply be executed which performs the following steps:

```Makefile
build:
    wasm-pack build --target nodejs
    cp package.json pkg/package.json
    cp index.js pkg/index.js
    cp generate_config pkg/generate_config
```

All files required to be part of the plugin Node package are copied to `pkg/`. 

To publish the package to [npm](https://www.npmjs.com/), [`wasm-pack`](https://github.com/rustwasm/wasm-pack) is used by simply running `wasm-pack publish `.

The plugin can be installed to be used by Homebridge by running `sudo npm install -g homebridge-rusty-spotify`.

## Issues, Future Work and Final Thoughts

Originally, I wanted to have a Spotify plugin so that I can set up an alarm in the morning that consisted of playing some music on Spotify using the iOS Home app. However, when I started writing this plugin I noticed that Spotify devices become inactive after a while or in the case of iPhone and iPad Spotify devices they become inactive when the app is closed. So, overnight all Spotify devices get disconnected and none are available in the morning anymore. I have some future ideas of setting up a Spotify client that does not get disconnected and stays active at all times, however, this is not an issue that Homebridge or a plugin can solve.

So while the plugin does not solve my initial use case yet, it does show that it is possible to write Homebridge plugins using Rust. Although I had to resort to a few workarounds which also resulted in developing the plugin taking much longer than if I had just used JavaScript or TypeScript, I am quite pleased with the outcome and did learn a lot in the process of writing. And it is also satisfying to see that other people are interested in using the plugin. 
