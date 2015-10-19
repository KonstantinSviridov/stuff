It is convenient to design an API client as consisting of four parts:

1. API independent execution core.
2. Model, reflecting API structure, deligating calls to the core.
3. API independent authorization and validation services prompted by the core.
4. API dependent authorization and validation databases prompted by the services.

With client designed like this a call decomposes to following steps:

1. The Model passes call information to the Core. This info must contain a method key of some kind which will be used to access services.
2. The Core passes the key to the Services in order to obtain ways to validate and authorize the particular method.
3. Once the ways are obtained, the Core performs authorization and validation.
4. The call is executed and response obtained.
5. The response is validated. (This step is intended to check RAML source correctness)
6. The Core constructs response object and passes it to the Model.

Let's take a look how a variation of this process works in the Typescript client:

1. The model registers the `RamlWrapper.Api` instance in the `AuthenticationManager` (via `ExecutionEnvironment`).
For the yet unknow `RamlWrapper.Api` the `AuthenticationManager` determines correspondence between `RamlWrapper.Method`s and `RamlWrapper.SecurityScheme`s.

2. `APIExecutor.execute()` recieives call information which consists of:
  * **complete call URL**
  * **method name**
  * (optional) **payload object**
  * (optional) **options objects** (containing query and header parameters)
  * **canonicalMethodUri** (complete method URI)

3. The `APIExecutor` utilizes the call info in order to build `harExecutor.ExtendedHarRequest` which is passed to `AuthenticationDecorator`. In fact `harExecutor.ExtendedHarRequest` is exactly `Har.Request` + `canonicalMethodUri`.

4. `AuthenticationDecorator` asks the `AuthenticationManager` to insert authorization parameters into `HAR.Request`. For this purpose it uses `canonicalMethodUri` to find `RamlWrapper.Method` inside context `RamlWrapper.Api` and passes this `RamlWrapper.Method` to the `AuthenticationManager` along with `HAR.Raquest`.

5. After being authorized, the `HAR.Request` is passed to `ValidatingExecutor`. It obtains `RamlWrapper.Method` just the same way as `AuthenticationDecorator`, takes RAML types of its bodies (if any) and validates request agains them.

6. The `HAR.Request` is passed to `SimpleExecutor` which delegates it to `XmlHttpRequest` module.

7. The response obtained is wrapped to `HAR.Response` and passed back to `ValidatingExecutor`.

8. The `ValidatingExecutor` validates response againts RAML types of response obtained from the `RamlWrapper.Method` and passes it back to `APIExecutor` (via `AuthenticationDecorator` which does nothing on the way back).

9. The `APIExecutor` uses `HAR.Raquest` to construct response object and returns this object to the Model.



Thus, implementing a client generator can be separated to

1. Designing protocol of comunication between model and core.
2. Implementing execution core
3. Implementing a model generator
4. Implementing services core
3. Implementing services database generator
