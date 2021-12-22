# Web ML Model Loader API Explainer

## Introduction

The Machine Learning Model Loader API is a proposed web API to take a custom, pre-trained machine learning (ML) model 
in a standard format, and apply it to example data in JavaScript in order to perform inference, like classification,
regression, or ranking. The idea is to make it easy and performant to use a custom, pre-built machine learning model
in web apps, across devices and browsers.

## Authors

*   Jonathan Bingham
*   Honglin Yu

## Participate

*   [Issue tracker](https://github.com/webmachinelearning/model-loader/issues)
*   [Web Machine Learning Community Group](https://www.w3.org/groups/cg/webmachinelearning)


## Goals

Like the related [Web Neural Network API](https://webmachinelearning.github.io/webnn) specification, some of the main the goals are: 

*   Improve performance, by accessing specialized hardware on device and eliminating network latency
*   Preserve privacy, by not shipping user data across the network
*   Provide a fallback model if network access is unavailable, possibly using a smaller and lower quality model on-device

The Model Loader API also has these additional goals:

*   Provide an API targeted at web developers, rather than ML framework authors. 
*   Optimize model execution beyond what's possible if each operation needs to be expressed as a Javascript method call.
*  Allow faster evolution by limiting the JavaScript API surface and relying on an external model format, rather than
   relying on a set of builder methods in JavaScript.

## Non-Goals

*   Support training a model in the browser
*   Modify a model at run time
*   Enable federated learning
*   Offer pre-trained models provided by the browser

The non-goals can be addressed with client-side JavaScript libraries and published models, or future Web APIs. 

## User research

The API is modeled after existing ML model loading APIs like:

*   [TensorFlow Serving](https://www.tensorflow.org/tfx/serving/serving_basic)
*   [Clipper](https://github.com/ucbrise/clipper)
*   [TensorRT Inference Server](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorrtserver)
*   [MXNet Model Server](https://github.com/awslabs/multi-model-server)

These and other APIs are already widely used by many products and organizations for large volumes of requests.

Using an API that’s similar to a server-based API makes it easier to switch between server-based and local-based
inference.


# Sample code

```js
// First, create an MLContext. This is consistent with the WebNN API. And we will 
// add two new fields, “numThread” and "modelFormat". 
const context = await navigator.ml.createContext(
                                     { devicePreference: "cpu",
                                       powerPreference: "low-power",
                                       numThread: 0,   // the default 0 means 
                                                       // "decide automatically". 
                                       modelFormat: "tflite" });
// Then create the model loader using the ML context.
loader = new MLModelLoader(context);
// In the first version, we only support loading models from ArrayBuffers. We 
// believe this covers most of the usage cases. Web developers can download the 
// model, e.g., by the fetch API. We can add new "load" functions in the future
// if they are really needed.
const modelUrl = 'https://path/to/model/file';
const modelBuffer = await fetch(modelUrl)
                            .then(response => response.arrayBuffer());
// Load the model.
model = await loader.load(modelBuffer);
// Use the `model.compute` function to get the output of the model from some 
// inputs.  Example ways of using this function includes, 
// 1. When there is only one input tensor of the model, one can simply input the 
// tensor, without specifying the name of it (the user can still designate this 
// input tensor by name if they like).
z = await model.compute({ data: new Float32Array([10]), 
                          dimensions: [1]) });
// 2. When there are multiple input tensors, the user has to designate the name 
// of the input tensors by their names.
z = await model.compute({ x: { data: new Float32Array([10]), 
                               dimensions: [1] },
                          y: { data: new Float32Array([20]), 
                               dimensions: [1] } });
// 3. The client can also specify the output tensor. This is consistent with the 
// WebNN API and can be useful, e.g., when the output tensor is a GPU buffer. At 
// this time, the function will return an empty promise. The dimension of the 
// output tensor specified must match the dimensions of the output tensor of the 
// model. 
z_buffer = ml.tensor({data: new Float64Array(1), 
                      dimensions: [1] });
await model.compute({ data: new Float32Array([10]), 
                      dimensions: [1] },
                    z_buffer);
// For the output tensor(s),
// Similar to the input arguments, if there is only one output tensor, the 
// `compute` function returns a tensor in case 1 and 2, and there is no need to 
// specify the name of the output tensor in case 3. But if there are multiple 
// output tensors, the output in case 1 and 2 will be a map from tensor name to 
// tensors, and in case 3, the output argument must be a map from tensor name to
// tensors too.
// For case 1 and 2, where the actual output data locate will depend on the 
// context: if it is CPU context, the output tensor’s buffer will be RAM buffer(s)
// and if the context is GPU context, the output tensor’s buffer will be GPU 
// buffer(s).
```

See also: [draft spec](https://webmachinelearning.github.io/model-loader/)


# Detailed design discussion

## What ML model format(s) should be supported? 

Any supported model format must be:

*   An open standard, without IP restrictions
*   Versioned
*   Backwards compatible (or else translatable)
*   Vendor neutral

Chrome is proposing to prototype with a TF Lite flat buffer format, in order to gauge developer interest in the
API and get feedback on the API shape. The TF Lite flat buffer format does not currently meet the requirements
for a web standard format. We expect the standard format of the eventual Model Loader API to be different. 

For the Model Loader API to succeed, there will need to be converters from all major ML formats into a web standard
representation.

## ML is evolving quickly. How will this stay up to date?

It will require ongoing investment and updates for browsers to keep pace with advances in ML and hardware. Model
formats evolve as well, and versioning will be required in order to ensure that new ML models are supported.

## How can web developers protect their proprietary ML models?

Short answer: they can’t, under this initial proposal.

But eventually this may need to be solved somehow. Data is often proprietary, and companies and organizations
consider the ML model to be IP. For server-based inference, the model never needs to leave the remote server,
which provides at least some level of protection.

For OS vendors, there could be a way to specify local URLs that cannot be read from JavaScript. Eg, a URL scheme
like ml://. There may well be other solutions. Creating a new URL scheme is not very appealing.

For web applications, maybe there’s a more DRM-like solution. Maybe the browser could have an encrypted model
download space that’s write-only, and it could fetch the remotely stored model over a secure connection. Once
downloaded, the model could be referenced, but no JavaScript library could read or access the contents. Only the
internal implementation code could access the model directly.

This requires further thought, by experts.

# Security considerations

It will be a lot of work. The browser will need to parse and validate the model in depth and provide sandboxing and
process isolation. For example, in chromeOS, each model instance will be isolated in a dedicated process and
sandbox, that is, even one web page loads two instances of the same model, each instance will be run in its own
process and sandbox.

Fuzz testing can help for operations. Each hardware driver will need to be secured. Security will
be a major part of the effort to implement general-purpose ML on the Web.

Because the users are responsible for downloading the model, we can not restrict the origin of the model inside this
API. So, depending on the users' feedback, we may provide optional CSP directive (e.g. "ml-model-src") to include
the checksum of allowed models that this API can check against.

# Privacy considerations

This API potentially makes fingerprinting easier. Because the API will be available or not, or execute much faster
or slower, depending on the device, operating system, and version, more information will be available for
fingerprinting and tracking of users.

Also, it might be useful for the API to expose some details about available hardware, because of the large impact on
performance. The details could be used for fingerprinting.

Eventually, over time, ML hardware will become more common, and the fingerprinting risk may diminish, so that perhaps
it won’t be notably worse than for today’s GPUs and CPUs. In the meantime, it might be possible to bucket the API
behavior into fewer categories, so there’s less information available for fingerprinting.

# Alternatives considered

## Graph API

The graph-based Web NN API and the Model Loader API are complementary approaches:

The graph API targets JavaScript frameworks, like ONNX.js and TensorFlow.js. The Model Loader API targets web developers.
Most web developers just want to perform inference on a pre-trained model. They don’t need to construct a model in
JavaScript, so the web platform API surface doesn’t need to include all of those operations, and web developers wouldn’t
use them directly.

In the graph API, the operation definition is encoded in JavaScript and must be approved one by one. With a model loader
API, the shape of the API is approved once, and then the file format can be updated by external maintainers. The web
standards process can approve updates to the spec for the format.

We'll need to do some benchmarking to understand if there are performance differences, and get feedback from developers to
see if it's valuable to offer both types of API.

Note that the model loader API could be implemented on top of a graph API by parsing the model into the graph. We're working
to share common data structures and API surfaces for both types of API in order to keep them in sync.

## Application-specific APIs

Application-specific APIs like face detection and barcode detection are also a complementary approach. They have some
limitations:

Developers want to run custom models too. Application-specific APIs can cover the top use cases only, and with limited flexibility.
Organizations want to differentiate based on their data and their own trained models. They can’t with canned models.

The APIs may give different results on different browsers, if the models themselves are provided with the browser and
operating system.

A generic Model Loader API addresses these concerns. Something like the Shape Detection APIs could be built on top of the
Model Loader API, with the benefit that developers could swap in their own models, and use those same models across browsers
and devices.

## Multiple model formats

In order to potentially support multiple model formats, we considered creating a distinct MLModelLoader class for
TFLiteModelLoader, to allow the method signature to support TF Lite specific attributes and be more friendly to IDEs.
Given that our goal is to support a common web standard format, and the method signature for common model formats like
TFLite and ONNX can be unified, we opted to stay with a more general class.

## Stakeholder feedback / opposition

The Model Loader API is currently included as a Tentative Specification in the
[Web Machine Learning Working Group Charter](https://www.w3.org/2021/04/web-machine-learning-charter.html).

*   Apple: No public signals.  
*   Google: Positive
*   Intel : Collaborating in Working Group
*   Microsoft: Collaborating in Working Group
*   Mozilla: No public signals
*   Salesforce: Collaborating in Working Group
 
## References & acknowledgements

Many thanks for valuable feedback and advice from:

*   Alex Russell
*   Anssi Kostiainen
*   Chai Chaoweeraprasit
*   Daniel Smilkov
*   Dominique Hazael-Massieux
*   Ganesan Ramalingam
*   Greg Whitworth
*   James Darpinian
*   Jeffrey Yaskin
*   Jiewei Qian
*   Jon Napper
*   Kai Ninomiya
*   Nikhil Thorat
*   Ningxin Hu
*   Rafael Cintron
