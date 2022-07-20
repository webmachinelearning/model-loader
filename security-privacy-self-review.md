Responses to the [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for the [Model Loader API](https://github.com/webmachinelearning/model-loader).

### 1. What information does this feature expose, and for what purposes?
This API allows an origin to query the machine learning model formats (e.g. Tensorflow, PyTorch, ONNX, etc.) supported by a particular browser and OS. This is necessary for the website to decide if they can make use of the API. For example, if a website only provides Tensorflow models, but the browser don't support them, the website shouldn't use this API.

### 2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?
Yes.

### 3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?
Doesn't expose PII information.

### 4. How do the features in your specification deal with sensitive information?
The API doesn't deal with sensitive information, except if it's provided by the origin. Each model is isolated per-origin.

### 5. Do the features in your specification introduce state that persists across browsing sessions?
No.

### 6. Do the features in your specification expose information about the underlying platform to origins?
Yes. The website can infer some capabilities of the underlying platform. For example, if a platform supports Tensorflow model format, and which versions of that model format.

### 7. Does this specification allow an origin to send data to the underlying platform?
Yes.

The API allows the origin to send a packaged machine learning model that represents multiple steps of numerical computation (e.g. multiply, add, etc.). The model will be processed by a platform-dependent program (usually a machine learning framework or library).

The API also allows the origin to send some data (as inputs to the model), usually an array of numbers. The size of the array is determined by the loaded model, and can be validated before the model computes the outputs.

### 8. Do features in this specification enable access to device sensors?
No.

### 9. Do features in this specification enable new script execution/loading mechanisms?
No.

Possible computations a model can perform is well defined per model format. The origin can't execute arbitrary code.

### 10. Do features in this specification allow an origin to access other devices?
No.

### 11. Do features in this specification allow an origin some measure of control over a user agent’s native UI?
No.

### 12. What temporary identifiers do the features in this specification create or expose to the web?
No temporary identifiers are used.

### 13. How does this specification distinguish between behavior in first-party and third-party contexts?
We are considering to disable this API and reject all relevant method calls in third-party contexts by default. Website authors can explicitly enable this API via permission policy for third-party contexts.

See: https://github.com/webmachinelearning/model-loader/issues/22 .

### 14. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?
The API behaves in the same way.

### 15. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes. See explainer.

- [Security considerations](https://github.com/webmachinelearning/model-loader/blob/main/explainer.md#security-considerations)
- [Privacy considerations](https://github.com/webmachinelearning/model-loader/blob/main/explainer.md#privacy-considerations)

### 16. Do features in your specification enable origins to downgrade default security protections?
No.

### 17. How does your feature handle non-"fully active" documents?
The loaded model will be released when navigating away. Pending computations will be cancelled.

### 18. What should this questionnaire have asked?
No more questions.