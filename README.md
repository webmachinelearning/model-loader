# Web ML Inference Explained

**ML Inference** is a proposed web API to take a custom, pre-trained machine learning model in a standard format, and apply 
it to example data in JavaScript in order to perform inference, like classification, regression, or ranking. The idea is to 
make it as easy as possible for web developers to use a custom, pre-built machine learning model in their web app, across 
devices and browsers. 

Performing inference locally can:

*   Preserve privacy, by not shipping user data across the network
*   Improve performance, by eliminating network latency
*   Provide a fallback if network access is unavailable, possibly using a smaller and lower quality model

The API is modeled after existing inference APIs like TensorFlow Serving, Clipper, TensorRT, and MXNet Model Server, which 
are already widely used by many products and organizations for large volumes of requests. Using an API that’s similar to a 
server-based API makes it easier to switch between server-based and local-based inference.

Unlike the Shape Detection API, ML Inference APIs are generic. Application-specific libraries and APIs could be built on 
top.

ML Inference does not support training a model, modifying a model, federated learning, or other functionality. Maybe future 
APIs could address those, if useful.

The underlying implementation can use any hardware acceleration available on the device, as an internal implementation 
detail.

# Sample code

```
const modelUrl = 'url/to/ml/model';
var exampleList = [{
  'Feature1': value1,
  'Feature2': value2
}];
var options = { 
  maxResults = 5
};

const mli = new MLInference();
const model = await mli.load(modelUrl)
model.infer(exampleList, options)
  .then(inferences => inferences.forEach(result => console.log(result)))
  .catch(e => {
    console.error("Inference failed: " + e);
  });
```


# Frequently asked questions

## How do we know this level of API will be useful?

This API proposal is modeled after existing services like TensorFlow Serving, which provide an API at the same level of 
abstraction. These existing APIs are already used by thousands of organizations large and small, serving billions of 
inference requests per day. 

## Why not just use a machine learning library like brain.js or tensorflow.js?

You totally can. In fact, those libraries could function as a polyfill for browsers that don’t yet support a standard 
inference API.

It’s possible that in the relatively near future, the operating system on every new phone, laptop, server, and other device 
will ship with a built-in ML inference engine that takes advantage of the underlying hardware. That ML inference software 
will have the support of the OS authors, will provide versioning, and will be heavily optimized. In that hypothetical future 
world, it wouldn’t make much sense to ship a duplicate copy of the same functionality, likely in a worse-performing 
implementation. 

The benefits of a built-in Inference API are:

*   Friendlier developer experience.
*   Operating system-level code can implement the API in C++, and the web can provide an API to access it. By contrast, 
    JavaScript is slower than native code. Realistically, WASM is probably the alternative.
*   Full access to hardware acceleration is available without the developer doing extra work or relying on a library that’s 
    limited to WASM, WebGPU, and WebGL.
*   Fewer bytes to ship.
*   A standardized ML model format. ML models become as easy to use as images or media files. 


## Why not just add more web standard APIs that are application-specific, like Shape Detection API?

Application-specific APIs like face detection and barcode detection have some issues:

*   Developers want to run custom models too. Application-specific APIs can cover the top use cases only, and with limited 
    flexibility 
*   Organizations want to differentiate based on their data and their own trained models. They can’t with canned models.
*   The APIs may give different results on different browsers, if the models themselves are provided with the browser and 
    operating system.

A generic inference API addresses these concerns. Something like the Shape Detection APIs could be built on top of ML 
Inference, with the benefit that developers could swap in their own models, and use those same models across browsers and 
devices.


## Why not use a graph API like the Neural Network API (NN API) on Android?

There are a few reasons:

*   Most web developers just want to perform inference on a pre-trained model. They don’t need to construct a model in
    JavaScript, so the web platform API surface doesn’t need to include all of those operations, and web developers wouldn’t use them directly.
*   Only a small number of ML library authors would actually use a graph API directly. The library authors would in turn 
    create the inference API that web developers want.
*   A graph API would have a very large API surface, and the operations in it are rapidly evolving. It’s easier to let a 
    model format evolve than to codify it in web platform APIs.
*   Building the complete graph representation into a JavaScript API means it can’t be used for proprietary models. The 
    entire model can be accessed by JavaScript. That said, a DRM-like solution for protected models would be very hard.
*   Standardizing at the graph level will further fragment the web, at the level where web developers write their application 
    code, because they’re forced to rely on higher level ML libraries and abstractions. They’ll end up not as web developers 
    writing with web standards, but as TensorFlow.js developers, or WinML developers, writing with framework-specific APIs. 
*   The Google team that has worked very closely with the NN API, and knows its history in great detail, does not recommend 
    moving forward at this level of abstraction, and is actively working on alternatives like MLIR to address the pitfalls and 
    shortcomings.


## What kinds of machine learning could this support?

The idea is to support any type, such as image classification, binary classification, logistic regression, sequence models, 
or ranking.

The API signature, inputs, and outputs would need to be carefully reviewed to make sure they work for all supported ML model 
types.


## What model formats are supported?

The model format must be:

*   An open standard, without IP restrictions
*   Versioned
*   Backwards compatible (or else translatable)

The most widely used open-source formats are PyTorch’s and TensorFlow’s. There’s also Keras, which is supported by 
TensorFlow, Theano, and CNTK. And there’s ONNX, which is intended to be convertible to and from other frameworks like 
PyTorch and TensorFlow. For all of these model formats, there are issues around backwards compatibility, and a translation 
layer may be required to ensure models continue to run in the future.

The web could standardize on:

1. A single canonical model format, used on all devices by all browsers.
2. A single canonical model format, supported by all devices, and optional additional formats, that may not be supported on 
   any given device.
3. A set of permissible model formats, with at least one format guaranteed to be available, but that format might be 
   different per device. In this option, all browsers running on a given device could use the same format, accessing the 
   same underlying inference engine.
4. A set of permissible model formats, all optional. Each device supports zero or more formats.

If a single canonical model format is adopted, browser vendors could optionally support additional formats. There are 
multiple reasons they might want to do this:

*   As a way to try out new functionality not yet supported in the standard
*   For performance, to avoid a conversion step from the interop format to the internally used format, if the conversion is 
    expensive.
*   As a hedge in case the standard interoperable format doesn’t evolve quickly enough

If multiple formats are supported, web developers will have a more complicated time, but maybe it’s not totally horrible. 
Developers already serve different images for different screen resolutions. It might not be such a burden to ship different 
ML models for different devices, if the tooling exists to create those different models. In practice, developers may need 
to ship different models per device anyway due to differences in memory and CPU. For example, they might ship a smaller 
model on a mobile device, and a larger, more accurate model for desktop. As long as the model format is browser-independent, 
it might be ok. In other words, it might be ok if a web site needs to serve WinML for Windows desktop, CoreML for iOS, and 
TensorFlow for Android, as long as those models work with the API in Edge, Firefox, Safari, and Chrome, and there’s any easy 
enough way to create the models in the necessary formats.


## How can web developers protect their proprietary ML models?

Short answer: they can’t, under this initial proposal.

But eventually this may need to be solved somehow. Data is often proprietary, and companies and organizations consider the ML 
model to be IP. For server-based inference, the model need never leave the remote server, which provides at least some level 
of protection.

For OS vendors, there could be a way to specify local URLs that cannot be read from JavaScript. Eg, a URL scheme like ml://.
There may well be other solutions.

For web applications, maybe there’s a more DRM-like solution. Maybe the browser could have an encrypted model download space 
that’s write-only, and it could fetch the remotely stored model over a secure connection. Once downloaded, the model could be 
referenced, but no JavaScript library could read or access the contents. Only the internal implementation code could access 
the model directly. 


## How would a browser vendor actually implement the API?

It’s anticipated that the implementation would be a thin wrapper over native code that runs on the operating system. For 
example, the underlying ML implementation could be provided by WinML on Windows, the NN API on Android, CoreML on iOS, and 
ML Service in Chrome OS. The browser development team would not need much ML expertise in order to implement the API.

Maybe the simplest implementation in a browser would be to use a host message handler to return a URL to a locally running 
service on the host OS. For convenience, a simple promise-based API on top of that could make it friendlier to use. Minimal 
code would need to change in the browser in order to pass through access to the native service.

Rough steps:

*   Install an ML framework like TensorFlow Serving locally.
*   Modify the browser to pass a host message handler for a new URL prefix, like ml://, which is a handle to the service 
    provided by the ML framework.
*   Write a client-side JavaScript promise-based API to provide better ergonomics.


## How will backwards compatibility be maintained?

In this proposal, it's up to the binary model format to ensure compatibility.

Maybe backwards compatibility isn't as important as we generall assume? ML is evolving quickly, and newer ML models tend to 
render older ones obsolete very quickly. 
Also, retraining often is important for some applications in order to maintain model freshness. For example, any rankings 
that consider recency or trends will need to be based on fresh models. If developers update their models often, there 
wouldn’t be a need to support old model formats, beyond the most recent version or two. It could even be built into the spec 
that no model older than [1 year] is guaranteed to work.

Even with frequent retraining, the model format needs to be versioned and there needs to be a way to maintain compatibility 
across subsequent releases. Implementations have to deal with parsing the model format and converting as needed. 

If the model format isn’t stable, and conversion between model formats is impossible or slow, backwards compatibility could 
require bundling multiple ML runtime versions. That could lead to bloat. Of course, with client-side solutions, the ML 
runtime is loaded separately for every site, so there’s even more potential bloat.


## ML is evolving quickly. How will this stay up to date?

Device hardware is diverse. Operating systems are varied. Machine learning implementations are changing. ML models are 
changing even faster, because new data is constantly becoming available. It’s a reality that things will change, and the 
underlying code will be different for different end users. The goal is to give access to hardware acceleration where 
available, and to minimize the burden of using custom ML models.

The challenge with this approach is that the ML model format holds the complexity. A browser could bundle a runtime 
(like TF Lite) or full TensorFlow, and upgrade versions with each release, roughly once every 6 weeks for Chrome. 
Upgrades aren’t the issue. Backwards and forwards compatibility are. Forwards compatibility is an issue, since models may be 
trained with newer versions of the ML frameworks, and browsers may lag several weeks or more behind.


## What about sandboxing and isolation of the executed models?

Does allowing arbitrary ML models to be executed open up major security holes? How will isolation be handled?

Currently, a common approach for native mobile apps is to bundle an ML runtime, like TF lite or CoreML, with each installed 
app. Because there’s an app review step for the app stores, there’s an extra layer of review. Security isolation is provided 
by the operating system.

For the web, it’s a little trickier, since a single web browser installation manages multiple web apps. Isolation is handled 
by the browser. With the browser calling out to a shared installation of an ML runtime, that runtime would need to provide 
isolation between callers. 


## What about fingerprinting? Doesn’t this make it easier?

Yes, potentially. Because the API will be available or not, or execute much faster or slower, depending on the device, 
operating system, and version, more information will be available for fingerprinting and tracking of users. 

Eventually, over time, ML hardware will become more common, and the fingerprinting risk will diminish, so that perhaps it 
won’t be notably worse than for today’s GPUs and CPUs. 

In the meantime, it might be possible to bucket the API behavior into fewer categories, so there’s less information available 
for fingerprinting. 


## Is it too high level to ever get agreement and ship?

A trend in web standards is to focus on small, low level APIs that are primarily used internally by JavaScript libraries and 
frameworks, which in turn provide higher level conveniences to web developers. An argument made in defense of the low level 
focus is that the low level APIs unlock the higher level capabilities. Those higher level layers are often heavily focused 
on ergonomics and stylistic preferences, and it’s better to let client libraries figure those things out.

An ML Inference API is lower level than the Shape Detection APIs, and higher level than WebGL and WebGPU. Perhaps an 
Inference API is the lowest level API that supports the interoperability of ML models themselves.

Web developers do not typically produce their own ML models. They receive models, created by a data scientist, and add them 
to a web app. It’s analogous to how web developers receive images created by a graphic designer and add them to a page. The 
ML Inference API is a low-level, generic API, like an image tag. The web platform could have evolved without an image tag, 
and instead offered only a canvas and a byte array input stream reader. But then web developers would be dependent on image 
rendering libraries built on top of those primitives, each with its own supported image formats, buffering behavior, and 
rendering. With access to a standard image tag, with standard image formats supported in all the browsers, developers and 
hobbyists were able to create content easily. Maybe ML inference is a similarly fundamental building block for the future of 
the intelligent web. 

The existence of widely adopted ML inference APIs like TensorFlow Serving suggests that an API at this level of abstraction 
is broadly useful and can be standardized.
