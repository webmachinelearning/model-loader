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

```js
// First, create an MLContext. This is consistent with WebNN API. And we will add a 
// new field, "modelFormat". 
const context = await navigator.ml.createContext(
                                     { devicePreference: "gpu",
                                       powerPreference: "low-power",
                                       modelFormat: "tflite" });
// Then create the model loader using the ML context. Notice that the indices for the
// input/output nodes are named and can be referenced in the "compute" function.
loader = new MLModelLoader(context, 
                           { inputs:  {x: 1, y: 2},
                             outputs: {z: 0 }));
// In the first version, we only support loading models from ArrayBuffers. We 
// believe this covers most of the usage cases. Web developers can download the 
// model, e.g., by the fetch API. We can add new "load" functions in the future
// if they are really needed.
const modelUrl = 'https://path/to/model/file';
const modelBuffer = await fetch(modelUrl)
                            .then(response => response.blob())
                            .then(blob => blob.arrayBuffer());
model = await loader.load(modelBuffer);
// Compute z = f(x,y) where the output buffer is pre-allocated. This is consistent 
// with the WebNN API and will be good when, for example, the output buffer is a 
// GPU buffer.
z_buffer = new Float64Array(1);
// The "model.compute" function is async and returns an empty promise.
// Here we make the input/output format consistent with the WebNN API.
await model.compute({ x: new Float64Array([10]),
                      y: new Float64Array([20])},
                    { z: z_buffer} );
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

A generic inference API addresses these concerns. Something like the Shape Detection APIs could be built on top of the 
Model Loader API, with the benefit that developers could swap in their own models, and use those same models across
browsers and devices.


## Why not use a graph API like the Neural Network API (NN API) on Android?

The graph-based WebNN API and the Model Loader API are complementary approaches:

*   The graph API targets JavaScript frameworks, like ONNX.js and TensorFlow.js. The Model Loader API targets web developers.
    Most web developers just want to perform inference on a pre-trained model. They don’t need to construct a model in
    JavaScript, so the web platform API surface doesn’t need to include all of those operations, and web developers wouldn’t 
    use them directly.
*   In the graph API, the operation definition is encoded in JavaScript and must be approved one by one. With a 
    model loader API, the shape of the API is approved once, and then the file format can be updated by external maintainers.
    The web standards process can approve updates to the spec for the format.

We'll need to do some benchmarking to understand if there are performance differences, and get feedback from developers to
see if it's valuable to offer both types of API.

Note that the model loader API could be implemented on top of a graph API by parsing the model into the graph. We're working
to share common data structures and API surfaces for both types of API in order to keep them in sync.


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

Chrome is proposing to prototype with a TF Lite flat buffer format, in order to gauge developer interest in the the API and
get feedback on the API shape. The TF Lite flat buffer format does not currently meet the requirements for a web standard
format. We expect the format to change. 

For Web ML APIs to succeed, there will need to be converters from all major ML formats into the web standard representation.


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


## How will backwards compatibility be maintained?

It's up to the model format to ensure backwards compatibility.


## ML is evolving quickly. How will this stay up to date?

It will require ongoing investment and updates for browsers to keep pace with advances in ML and hardware.


## What about security?

It will be a lot of work. Each hardware driver will need to be secured. The browser will need to parse and validate the
model in depth and provide sandboxing and process isolation. Fuzz testing can help for operations. Security will be a major
part of the effort to implement general-purpose ML on the Web.


## What about fingerprinting? Doesn’t this make it easier?

Yes, potentially. Because the API will be available or not, or execute much faster or slower, depending on the device, 
operating system, and version, more information will be available for fingerprinting and tracking of users. 

Also, it might be valuable for the API to expose some details about available hardware, because of the large impact on
performance. 

Eventually, over time, ML hardware will become more common, and  fingerprinting risk may diminish, so that perhaps it 
won’t be notably worse than for today’s GPUs and CPUs. In the meantime, it might be possible to bucket the API behavior
into fewer categories, so there’s less information available for fingerprinting. 


## Is it too high level to ever get agreement and ship?

The existence of widely adopted model serving APIs like TensorFlow Serving suggests that an API at this level of abstraction 
is broadly useful and can be standardized.
