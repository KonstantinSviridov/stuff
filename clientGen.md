#Client design

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

###Typescript client
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


#Client generator design
Thus, implementing a client generator can be separated to

1. Designing protocol of comunication between model and core.
2. Implementing execution core
3. Implementing a model generator
4. Implementing services core
3. Implementing services database generator


###Model generator
It may appear not handy to generate client model code directly from the parsed RAML. We suggest deviding this process on two steps:

1. Iterate throw the RAML AST and build some abstract model.
2. Serialize the gathered model into code on particular language.

API Workbench project provides a simplified Typescript code model: [TSDeclModel](see here)

Let's consider an example of gathering client model. Assume that we have following require statements in our exampl module:
```javascript
import TS = require('{some path to src/ramlscript/TSDeclModel}')
import raml2ts1 = require('{some path to src/ramlscript/raml2ts1}')
import RamlWrapper = require('{some path to src/raml1/artifacts/raml003Parser}')
import norebookModelBuilder = require('{some path to src/ramlscript/notebookModelBuilder}')
```

#####Parse API and create a `TS.TSModule`:
  ```javascript
  var api = norebookModelBuilder.loadApi1('{path to your RAML spec root .raml file}')
  var module = new TS.TSModule();
  ```

#####Create a model tree isomorphic to API structure:

We need `TSModelDecl` classes for representing classis and their members.

* `TS.TSInterface` is a `TSModelDecl` class which is used to define interfaces or classes (also enums and function interfaces). In our example for each `RamlWrapper.Resource` we will create a `TS.TSInterface` successor: `raml2ts1.ResourceMappedInterface`. The difference between these two is that with `raml2ts1.ResourceMappedInterface` you are able to retrieve original `RamlWrapper.Resource` which may appear useful on serialization step:
 
  ```javascript
  var resource:RamlWrapper.Resource = resourceMappedInterfaceInstance.original().originalResource;
  ```

* `TS.TSAPIElementDeclaration` is a `TSModelDecl` class which is used to define interface or class members -- both fields and methods. In our example for each child `RamlWrapper.Resource` (which belong to RamlWrapper.Api or another RamlWrapper.Resource) we will create a member (inside class corresponding to its parent) represented by `TS.TSAPIElementDeclaration` successeor: `raml2ts1.TSResourceMappedApiElement`. The reason to use `raml2ts1.TSResourceMappedApiElement` rather then `TS.TSAPIElementDeclaration` is just the same: with `raml2ts1.TSResourceMappedApiElement` you are able to retrieve original `RamlWrapper.Resource`:
  ```javascript
  var resource:RamlWrapper.Resource = tsResourceMappedApiElementInstance.originalResource;
  ```


```javascript
processResource(resource:RamlWrapper.Resource,parent:TS.TSInterface){

    var relUri = resource.relativeUri().value();	
	
    //generate name for member corresponding the resource in some way
    var memberName = constructFieldName(relUri);
	
    //here we add member to the parent class. On level 1 of recursion it is `apiClass`
    var member = new raml2ts1.TSResourceMappedApiElement(parent,memberName,resource);

    //generate name for class corresponding the resource in some way
    var className = constructClassName(relUri);
    //here we create class corresponding to the resource. It is passed to next level of recursion.
    var resourceClass =	new raml2ts1.ResourceMappedInterface(module,className,member);
    
    member.rangeType = resourceClass.toReference();
	
    resource.resources.forEach(x=>processResource(x,resourceClass));
}

var apiClass = new TS.TSInterface(module,'Api');
api.resources.forEach(x=>processResource(x,apiClass));
```

#####Serialize model  
Now the module is capable of returning classes corresponding to API and resources, which, in turn, can return a list of their members. Here is a dummy example of model serialization:
```javascript
function serializeClass(clazz:TS.TSInterface){
  
  var name = clazz.name();
  var content = 'export class ' + name + '{\n\n';
  clazz.children().forEach(x => content += serializeMember(x));
  content += '}';
  
  //some way to store obtained class code
  write(content,class);
}

function serializeMember(member:TS.TSApiElementDeclaration){
  var name = member.name();
  
  //retrieve members class
  var memberClass:raml2ts1.ResourceMappedInterface
            = (<raml2ts.ResourceMappedReference>mamber.rangeType).resourceInterface();
  
  var className = memberClass.name();
  return '    ' + name + ':' + 'className' + ';\n\n';
}

module.children().forEach(x=>serializeClass(x));
```


