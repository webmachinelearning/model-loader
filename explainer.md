# Web ML Model Loader API Explainer

**Model Loader** is a proposed web API to load a custom, pre-trained machine learning model in a standard format, compile
it for the available hardware, and apply it to example data in JavaScript in order to perform inference, like classification, 
regression, or ranking. The idea is to make it as easy as possible for web developers to use a custom, pre-built machine learning model in their web app, across devices and browsers. 

Performing inference locally can:

*   Preserve privacy, by not shipping user data across the network
*   Improve performance, by eliminating network latency
*   Provide a fallback if network access is unavailable, possibly using a smaller and lower quality model

The API is modeled after existing inference APIs like TensorFlow Serving, Clipper, TensorRT, and MXNet Model Server, which 
are already widely used by many products and organizations for large volumes of requests. Using an API that’s similar to a 
server-based API makes it easier to switch between server-based and local-based inference.

Unlike the Shape Detection API, the model loader APIs are generic. Application-specific libraries and APIs could be built on 
top.

The API does not support training a model, modifying a model, federated learning, or other functionality. Maybe future 
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

const modelLoader = new ModelLoader();
const model = await modelLoader.load(modelUrl)
const compiledModel = await model.compile()
compiledModel.predict(exampleList, options)
  .then(inferences => inferences.forEach(result => console.log(result)))
  .catch(e => {
    console.error("Inference failed: " + e);
  });
```


# Frequently asked questions

## How do we know this level of API will be useful?

This API proposal is modeled after existing services like TensorFlow Serving, Clipper, TensorRT, and MXNet Model Server
which provide an API at the same level of abstraction. These existing APIs are already used by thousands of organizations
large and small, serving billions of inference requests per day. 

## Why not just use a machine learning library like brain.js or tensorflow.js?

You totally can. In fact, those libraries could function as a polyfill for browsers that don’t yet support a standard 
model loader API.

It’s possible that in the relatively near future, the operating system on every new phone, laptop, server, and other device 
will ship with a built-in ML engine that takes advantage of the underlying hardware. That ML inference software 
will have the support of the OS authors, will provide versioning, and will be heavily optimized. In that hypothetical future 
world, it wouldn’t make much sense to ship a duplicate copy of the same functionality, likely in a worse-performing 
implementation. 

The benefits of a built-in model loader API are:

*   Friendlier developer experience.
*   Operating system-level code can implement the API in C++, and the web can provide an API to access it. By contrast, 
    JavaScript is slower than native code. Realistically, WASM is probably the alternative.
*   Full access to hardware acceleration is available without the developer doing extra work or relying on a library that’s 
    limited to WASM, WebGPU, or WebGL.
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
    JavaScript, so the web platform API surface doesn’t need to include all of those operations, and web developers wouldn’t 
    use them directly.
*   Only a small number of ML library authors would actually use a graph API directly. The library authors would in turn 
    create the model loader API that web developers want.
*   A graph API would have a very large API surface, and the operations in it are rapidly evolving. It’s easier to let a 
    model format evolve than to codify it in JavaScript APIs.
*   Building the complete graph representation into a JavaScript API means it can’t be used for proprietary models. The 
    entire model can be accessed by JavaScript. That said, a DRM-like solution for protected models would be very hard.
*   Standardizing at the graph level will further fragment the web, at the level where web developers write their application 
    code, because they’re forced to rely on higher level ML libraries and abstractions. They’ll end up not as web developers 
    writing with web standards, but as TensorFlow.js developers, or WinML developers, writing with framework-specific APIs. 
*   The Google team that has worked very closely with the NN API, and knows its history in great detail, does not recommend 
    moving forward at this level of abstraction, and is actively working on alternatives like MLIR to address the pitfalls and 
    shortcomings.


## Why does the TensorFlow team not recommend standardizing on a graph API like the Neural Network API?

The short answer is that ML is evolving too quickly, and the graph and operation level of abstraction has been too unstable.
Even with an operating system that a single company largely controls, on devices that have a relatively short lifespan, the
graph model has not worked out. Given that the web requires compatibility for much longer timeframes, and is developed by
a community, the TensorFlow team believes that what has not worked for Android is even less likely to work for the web.

This same concern applies to model formats for a model loader API that are ML operation-based. In the time since
TensorFlow launched, the number of operations has grown rapidly. The field of ML is rapidly changing, and new capabilities
are approaches are published daily. Any graph or operation-based approach is going to face an ongoing challenge of
standardizing and evolving. It's not possible to design a great API, ship, and be done, and know that the API will endure.
Its surface will need to be continually added to, and parts of it will fall into disuse as the field finds better approaches.
This isn't the type of API that lends itself to massive scale deployment with backwards compabibility guarantees.

The Android and TensorFlow teams have had experience shipping an API to hundreds of millions of devices, supporting it,
and evolving it. The fact that they have decided against tensor operations as the level of abstraction
going forward ought to be a caution against following the same path for web standards.


## What kinds of machine learning could a model loader API support?

The idea is to support any type, such as image classification, binary classification, logistic regression, sequence models, 
or ranking.

The API signature, inputs, and outputs would need to be carefully reviewed to make sure they work for all supported ML model 
types.


## What are the requirements for the model format(s)?

Any supported model format must be:

*   An open standard, without IP restrictions
*   Versioned
*   Backwards compatible (or else translatable)
*   Vendor neutral

In order for a model format to endure, it must also be:

*   At the right level of abstraction


## What is the right level of abstraction for the model format?

The model format could be:

1. A set of tensor operations with a graph
2. Hardware-specific instructions
3. An intermediate representation 

Recommended, in the long-term: 3. A near to mid-term solution could be 1.

The most widely used open-source formats are PyTorch’s and TensorFlow’s. There’s also Keras, which is supported by 
TensorFlow, Theano, and CNTK. Also of note is ONNX, which is intended to be convertible to and from other frameworks like 
PyTorch and TensorFlow. For all of these model formats, there are issues around backwards compatibility, and a translation 
layer may be required to ensure models continue to run in the future. All of these currently popular formats are based on
the same premise: enumerating a set of high-level operations and encoding them, often in protocol buffers). This level
of abstractionis exactly what the TensorFlow team recommends against, and is moving away from.

Ultimately, performance will depend upon hardware-specific instructions. But that's not the right level of abstraction for
the web, since the web platform needs to be hardware-independent.

An intermediate representation has the potential to address the limitations of operation sets and hardware-specific
instructions. One example of an intermediate representations is
MLIR (multi-layer intermediate representation)](https://mlir.llvm.org/), which is part of the LLVM foundation.

Potential benefits:

*   Extensible: New operations can be compiled into the same intermediate representation, without changes to the spec
*   Performant: Because it's closer to the metal, it can perform better than operation sets.
*   Custom ops can be defined in the intermediate representation, instead of JavaScript

Note that a graph API could be built using MLIR, and JavaScript APIs for all of the primitives could be defined. This would
satisfy the web vision of exposing browser internals at the lowest level.


## Why not adopt MLIR today?

If an intermediate representation like MLIR is the right approach for the web, why not standardize on it today?

*   It isn't ready. It's very much a work in progress.
*   Proof-of-concept support exists for TensorFlow, PyTorch, and ONNX using XLA HLO, but 's not production-ready.

These are pretty serious barriers. MLIR could be the right long-term solution, but it can't be adopted today.


## Is a model loader API blocked on MLIR or an alternative intermediate representation becoming a standard?

Not necessarily. It's possible that the shape of the API could be standardized on, without the details of an IR being
resolved yet. The community group could proceed to implement using an existing operation-level representation, like ONNX
or TFlite. Perhaps by the time the browser vendors are ready to ship, an IR will be ready.

Alternatively, browsers could ship an operation-based format sooner, and an IR later. The operation-based format could
be experimental, and never ship without a flag. Or the web platform could support both kinds of format, indefinitely,
and let developers choose which to use. Over time, the IR may become more popular.

Options:

A. Wait for an IR to become a standard before shipping anything; ship only an IR format.
B. Ship an experimental API based on an operation-level format. Wait for IR before shipping to production.
C. Ship an operation-based format, and later deprecate it and support an IR instead.
D. Ship an operation-based format, and later add a second format that is an IR. 

Recommended: B or D?

For either level of abstraction, whether an operation-based format, or an IR, the web could standardize on:

1. A single canonical model format, used on all devices by all browsers.
2. A single canonical model format, supported by all devices, and optional additional formats, that may not be supported on 
   any given device.
3. A set of permissible model formats, with at least one format guaranteed to be available, but that format might be 
   different per device. In this option, all browsers running on a given device could use the same format, accessing the 
   same underlying inference engine.
4. A set of permissible model formats, all optional. Each device supports zero or more formats.

Recommended: Long-term, option 1 is the ideal. Near to mid-term, 2 unblocks developers and OS and ML framework authors. 

In principle, it would be great to standardize on a single format, and a single level of abstraction.
If a single canonical model format is adopted at a given level of abstraction, browser vendors could optionally
support additional formats. There are multiple reasons they might want to do this:

*   As a way to try out new functionality not yet supported in the standard
*   For performance, to avoid a conversion step from the interop format to the internally used format, if the conversion is 
    expensive.
*   As a hedge in case the standard interoperable format doesn’t evolve quickly enough

In principle, an IR would address these concerns, and they would mostly only apply to higher-level operation-based APIs,
due to the rapid evolution of ML and need for new operations.

If multiple formats are supported, web developers will have a more complicated time, but maybe it’s not totally horrible. 
Developers already serve different images for different screen resolutions. It might not be such a burden to ship different 
ML models for different devices, if the tooling exists to create those different models. In practice, developers may need 
to ship different models per device anyway due to differences in memory and CPU. For example, they might ship a smaller 
model on a mobile device, and a larger, more accurate model for desktop. As long as the model format is browser-independent, 
it might be ok. In other words, it might be ok if a web site needs to serve ONNX for Windows desktop, CoreML for iOS, and 
TFlite for Android, as long as those models work with the same API in Edge, Firefox, Safari, and Chrome, and there’s
an easy enough way to create the models in the necessary formats.


## How can web developers protect their proprietary ML models?

Short answer: they can’t, under this initial proposal.

But eventually this may need to be solved somehow. Data is often proprietary, and companies and organizations consider the ML 
model to be IP. For server-based inference, the model never needs to leave the remote server, which provides at least
some level of protection.

For OS vendors, there could be a way to specify local URLs that cannot be read from JavaScript. Eg, a URL scheme like ml://.
There may well be other solutions. Creating a new URL scheme is not very appealing.

For web applications, maybe there’s a more DRM-like solution. Maybe the browser could have an encrypted model download space 
that’s write-only, and it could fetch the remotely stored model over a secure connection. Once downloaded, the model could be 
referenced, but no JavaScript library could read or access the contents. Only the internal implementation code could access 
the model directly. 

This requires further thought, by experts.


## How would a browser vendor actually implement the API?

Naively, the hope is that the implementation would be a thin wrapper over native code that runs on the operating
system. For example, the underlying ML implementation could be provided by WinML on Windows, the NN API or future MLIR API
on Android, CoreML on iOS, and ML Service in Chrome OS. The browser development team would not need much ML expertise in
order to implement the API.

After discussions with browser experts, it's clear that the browser needs to deeply understand the model and its
internals, in order to avoid executing untrusted code. Only after parsing and validating thoroughly would a handoff to
a lower-level compiler or sandboxed execution engine be safe.


## How will backwards compatibility be maintained?

In this proposal, it's up to the model format to ensure compatibility.

Maybe backwards compatibility isn't as important as we generall assume? ML is evolving quickly, and newer ML models tend to 
render older ones obsolete very quickly.

Also, retraining often is important for some applications in order to maintain model freshness. For example, any rankings 
that consider recency or trends will need to be based on fresh models. If developers update their models often, there 
wouldn’t be a need to support old model formats, beyond the most recent version or two. It could even be built into the spec 
that no model older than, eg, 1 year is guaranteed to work.

Even with frequent retraining, the model format needs to be versioned and there needs to be a way to maintain compatibility 
across subsequent releases. Implementations have to deal with parsing the model format and converting as needed. 

If the model format isn’t stable, and conversion between model formats is impossible or slow, backwards compatibility could 
require bundling multiple ML runtime versions. That could lead to bloat. Of course, with client-side solutions, the ML 
runtime is loaded separately for every site, so there’s even more potential bloat.

Another idea is to make sure that there are polyfills, and set the expectation that the API will be versioned, and older
versions will be deprecated and removed.


## ML is evolving quickly. How will this stay up to date?

Device hardware is diverse. Operating systems are varied. Machine learning implementations are changing. ML models are 
changing even faster, because new data is constantly becoming available. It’s a reality that things will change, and the 
underlying code will be different for different end users. The goal is to give access to hardware acceleration where 
available, and to minimize the burden of using custom ML models.

The challenge is that the ML model format holds the complexity. Forwards compatibility is an issue, since models may be 
trained with newer versions of the ML frameworks, and browsers may lag several weeks or more behind.


## What about sandboxing and isolation of the executed models?

Does allowing arbitrary ML models to be executed open up major security holes? How will isolation be handled?

Currently, a common approach for native mobile apps is to bundle an ML runtime, like TF lite or CoreML, with each installed 
app. Because there’s an app review step for the app stores, there’s an extra layer of review. Security isolation is provided 
by the operating system.

For the web, it’s a little trickier, since a single web browser installation manages multiple web apps. Isolation is handled 
by the browser. With the browser calling out to a shared installation of an ML runtime, that runtime would need to provide 
isolation between callers. Also, the browser would need to parse and validate the model in depth.


## What about fingerprinting? Doesn’t this make it easier?

Yes, potentially. Because the API will be available or not, or execute much faster or slower, depending on the device, 
operating system, and version, more information will be available for fingerprinting and tracking of users. 

Eventually, over time, ML hardware will become more common, and the fingerprinting risk will diminish, so that perhaps it 
won’t be notably worse than for today’s GPUs and CPUs. 

In the meantime, it might be possible to bucket the API behavior into fewer categories, so there’s less information available 
for fingerprinting. 


## Is it too high level to ever get agreement and ship?

A trend in web standards is to focus on small, low level APIs that are primarily used internally by JavaScript libraries and 
frameworks, which in turn provide higher level conveniences to web developers. An argument made in support of the low level 
focus is that the low level APIs unlock the higher level capabilities. Those higher level layers are often heavily focused 
on ergonomics and stylistic preferences, and it’s better to let client libraries figure those things out.

An model loader API is lower level than the Shape Detection APIs, and higher level than WebGL and WebGPU.

Web developers do not typically produce their own ML models. They receive models, created by a data scientist, and add them 
to a web app. It’s analogous to how web developers receive images created by a graphic designer and add them to a page. The 
model loader API is a low-level, generic API, like an image tag. The web platform could have evolved without an image tag, 
and instead offered only a canvas and a byte array input stream reader. But then web developers would be dependent on image 
rendering libraries built on top of those primitives, each with its own supported image formats, buffering behavior, and 
rendering. With access to a standard image tag, and standard image formats supported in all the browsers (even though not
prescribed by the spec!), developers and hobbyists were able to create content easily. Maybe ML model loading is a similarly
fundamental building block for the future of the intelligent web. 

The existence of widely adopted model serving APIs like TensorFlow Serving suggests that an API at this level of abstraction 
is broadly useful and can be standardized.

The internals of a model loader API could also stabilize to the point where exposing the low-level instructions or operations
directly to web developers makes sense.
