## Libraries Expansion Procedure

### RAML Components and Their Dependencies

By **RAML component** we understand any resource, resource type, trait, type, annotation type and security scheme.

We say that RAML component `A` depends on RAML component `B` if there exists a sequence of transitions by reference
from definition of `A` to definition of `B`.
The dependency relation between RAML components is transitive i.e. if `A` depends on `B` and `B` depends on `C` then `A` depends on `C`.

#### Examples:

```
#'MyType' type depends on `lib.LibType` type
types:
  MyType:
    type: lib.LibType
```

```
#'/resource' resource depends on `lib.LibType` type
/resource:
  post:
    body:
      application/json:
        type: lib.LibType
```

```
#'ChildType' type depends on 'ParentType' type, 'lib.LibType' type,
#'lib2.PropertyType' type and 'lib2.SomeAnnotation' annotation type
types: 
  ParentType:
    type: lib.LibType
  
  ChildType:
    type: ParentType
	properties:
	  p1:
	    type: lib2.PropertyType
		(lib2.SomeAnnotation): 5
```



By **dependencies set** of RAML component `A` we understand a set of all RAML components which `A` depends on.

Obviously, the following statements are true:

* For each RAML component its dependencies set can be algorithmically calculated.

* For each pair of RAML components the question about one being dependent on another can be algorithmically solved.

### RAML Specifications and Their Dependencies

By **RAML specification** we understand a set of files, consisting of

* main RAML file defining API, library, extension, overlay or fragment
* all the other files, reachable by `!include` transitions from the main file.

By **dependencies set** of RAML specification we understand a union of dependencies sets of components declared in the specification.

By **outer dependencies set** of the specification we understand a set of those dependencies, which are declared outside the specification.

Let's say that specification **depends** on some library `L`, if `L` contains at least one dependency of the specification.

#### Example
```
#api.raml
#%RAML 1.0
title: API Dependencies Example
uses:
  typesLib: typesLib.raml
  resourceTypesLib: resourceTypes.raml

/resource:
  post:
    body:
      application/json: !include bodyType.raml
  put:
    body:
      application/json: typesLib.MyType
```

```
#bodyType.raml
#%RAML 1.0 TypeDeclaration

uses:
  customTypes: customTypes.raml

properties:
  customProperty: customTypes.MyCustomType
```

```
#customTypes.raml
#%RAML 1.0 Library

types:
  MyCustomType: object
```

```
#typesLib.raml
#%RAML 1.0 Library
uses:
  baseTypes: baseTypes.raml
  annotationTypes: annotationTypes.raml

types:
  MyType:
    type: baseTypes.BaseObjectType

  MyStringType:
    type: baseTypes.BaseStringType
    (annotationTypes.MyAnnotation): someStringValue
```

```
#baseTypes.raml
#%RAML 1.0 Library

types:
  BaseObjectType: object

  BaseStringType: string
```

```
#annotationTypes.raml
#%RAML 1.0 Library

annotationTypes:
  MyAnnotation: string
```

```
#resourceTypes.raml
#%RAML 1.0 Library

resourceTypes:
  myResourceType:
    get:
    post:
```

Here we consider `api.raml` to be the main file of specification, which consists of `api.raml` and `resourceTypes.raml`. Thus, the specification dependencies set is as follows:

* `/resource` resource
* `MyCustomType` type from `customTypes.raml` (outer dependency)
* `MyType` type from `typesLib.raml` (outer dependency)
* `BaseObjectType` type from `baseTypes.raml` (outer dependency)

Thus, dependency libraries of the specification are

* `customTypes.raml`
* `typesLib.raml`
* `baseTypes.raml`

At the same time, `resourceTypes.raml` and `annotationTypes.raml` do not belong to set of dependency libraries of the specification. 

### Goal

Suppose we have a valid expanded RAML API specification in our hands.
The goal is to enrich the specification with its outer dependencies, and, thus, make it free from libraries.

### Implementation Sketch

We assume that before the library expansion procedure starts, the following conditions are satisfied:

1. The specification is represented by single file, i.e. all the included files are inlined into the main file. 

1. Libraries used by fragments are mentioned in `uses` sections of the main file (see the _Enriching 'uses' Section_ section)

1. Conditions 1 and 2 are satisfied by each dependency library of the specification


Now the library expansion procedure is as follows:

1. Collect all the dependency libraries of the specification.

1. Assign a unique identifier to each library from the set.

1. For each RAML component from the outer dependencies set create a copy of the component and apply the four steps to the copy:
  
  * append as prefix identifier of library declaring the component to the copy name
  
  * apply **References Patching Procedure** to all the references in the copy definition:
    
	* remove library prefix (if any) from the reference name
	
	* append as prefix identifier of library declaring the referenced component to the reference name
  
  * append the patched component to the specification

1. Clean `uses` section of the specification.

#### Example

We will continue using the former example, which already has its dependency libraries computed.

Lets inline `bodyType.raml` into `api.raml` and contribute the `customTypes.raml` library to the `uses` section:

```
# Expanded api.raml
#%RAML 1.0
title: API Dependencies Example
uses:
  typesLib: typesLib.raml
  resourceTypesLib: resourceTypes.raml
  customTypes: customTypes.raml

/resource:
  post:
    body:
      application/json:
        properties:
          customProperty: customTypes.MyCustomType
  put:
    body:
      application/json: typesLib.MyType
```

The specification does not require to be expanded in sense of templates and all its dependency libraries do not include any files, thus, we can switch to providing the dependency libraries with their identifiers. Lets do it in the following way:

```
typesLib: typesLib.raml
customTypes: customTypes.raml
typesLib.baseTypes: baseTypes.raml
```

The specification has three outer dependencies: `MyCustomType` type from `customTypes.raml`, `MyType` type from `typesLib.raml` and `BaseObjectType` type from `baseTypes.raml`. The results of patching names and definitions of these components are:
```
customTypes.MyCustomType: object

typesLib.baseTypes.BaseObjectType: object

typesLib.MyType:
  type: typesLib.baseTypes.BaseObjectType 
```

Finally, let's contribute the patched outer dependencies into the main file and remove the `uses` section:

```
# Final api.raml
#%RAML 1.0
title: API Dependencies Example

types:
  customTypes.MyCustomType: object

  typesLib.baseTypes.BaseObjectType: object

  typesLib.MyType:
    type: typesLib.baseTypes.BaseObjectType

/resource:
  post:
    body:
      application/json:
        properties:
          customProperty: customTypes.MyCustomType
  put:
    body:
      application/json: typesLib.MyType
```


### Enriching 'uses' Section

We understand a **usage conflict** as a situation of different libraries being used under same name in fragments of RAML specification and/or its main file.

#### Examples of Usage Conflicts:

Here API and included resource type fragment use different libraries under same name:
```
#api.raml
#%RAML 1.0
title: Usage Conflict Example 1

uses:
  lib: typesLib.raml

resourceTypes:
  rt: !include rt.raml

/resource:
  type: rt
  post:
    body:
      application/json: lib.LibType
```

```
#rt.raml
#%raml 1.0 ResourceType

uses:
  lib: customTypesLib.raml

put:
  body:
    application/json: lib.LibType
```

In the next example there is no conflicts between API and fragments, but the fragments together are in conflict:
```
#api.raml
#%RAML 1.0
title: Usage Conflict Example 1

uses:
  typesLibrary: typesLib.raml

resourceTypes:
  rt1: !include rt1.raml
  rt2: !include rt2.raml

/resource1:
  type: rt1
  post:
    body:
      application/json: typesLibrary.LibType

/resource2:
  type: rt2

```

```
#rt1.raml
#%raml 1.0 ResourceType

uses:
  lib: customTypesLib1.raml

put:
  body:
    application/json: lib.LibType
```

```
#rt2.raml
#%raml 1.0 ResourceType

uses:
  lib: customTypesLib2.raml

put:
  body:
    application/json: lib.LibType
```

#### Enriching 'uses' Section In Conflicts Free Situation

In conflicts free situation all libraries used by fragments may be used under same names by the main file.

#### Enriching 'uses' Section in Situation of Usage Conflicts

A natural way of resolving usage conflicts is appending integer indexes to names of the conflicting libraries.

Below we describe a strict rule of ordering fragments included by the main file. Once the complete list of fragments is ordered by this rule, we can ascribe integer indexes to fragments themselves, and then, propagate them to library names if a conflict arises.

For the examples of conflicts above the rule would result in
```
[ rt.raml ]
```
and
```
[ rt1.raml, rt2.raml ]
```
Thus, the resulting usage sections would be
```
uses:
  lib: typesLib.raml
  lib0: customTypesLib.raml
  
```
and
```
uses:
  typesLibrary: typesLib.raml
  lib0: customTypesLib1.raml
  lib1: customTypesLib2.raml
```

#### Fragments Ordering

For the given RAML specification we construct a tree using files which form the specification as nodes of the tree:

* root node is the main file of the specification
* children of a node are exactly those files which are directly included into the file of the node

For each tree node we order its children by position of their first inclusion into file of the node.
Then we list all the files by traversing the tree in direct order.

First, we remove from the list all but the first occurence of each file and thus, obtain a list, which contains exactly once each file of the specification.  

Second, we remove from the list all files which are not RAML fragments.

The resulting list provides us with desired fragments order.


### Constructing Library Identifiers

We assume that all specifications considered in this section (including dependency libraries) are represented as single file without `!include` tags.

Lets say that a library `L` is **reachable** from a RAML specification with path `p` if one of the two conditions are satisfied

1. the specification uses `L` under name `p`

1. the specification uses some library `L'` under name `n`, such that `L` is reachable from `L'` with path `p'` and `p=n.p'`

Lets define **path length** as a number of segments obtained by splitting it by the `.` symbol.
Thus, if it happens that on some step of path construction library name contains `.` symbol, this name contributes more then one segment to the path.

Now each dependency library is reachable with some path from the specification.

For each such library we consider a set of paths it is reachable with and take the shortest one as the identifier.
If there are more then one paths with minimal length, we take the lexically smallest one.

#### Examples
The example in the _Implementation Sketch_ section uses the rule above for constructing identifiers. Lets consider some more complex cases, which are not covered by the example.

Until the end of the _Examples_ section we do not expose references to components declared in libraries, but we assume that all the libraries mentioned actually do belong to dependencies set.

```
#main file: api.raml
#% RAML 1.0
uses:
  resourceTypes: resourceTypesLib.raml
  types: typesLib.raml
```

```
#resourceTypesLib.raml
#% RAML 1.0
uses:
  types: typesLib.raml
  annotations: annotationsLib.raml
```

```
#typesLib.raml
#% RAML 1.0
uses:
  annotations: annotationsLib.raml
```

```
#annotationsLib.raml
#% RAML 1.0
uses:
  types: typesLib.raml
```

Here, we have three dependency libraries: `resourceTypesLib.raml`, `typesLib.raml` and `annotationsLib.raml`.
Consider the following sets of paths, the libraries are reachable with from the main file:

```
resourceTypesLib.raml: [
    resourceTypes
  ],
typesLib.raml: [
    types,
    resourceTypes.types
  ]
annotationsLib.raml: [
    resourceTypes.annotations,
    resourceTypes.types,annotations,
    types.annotations
]
```
As `typesLib.raml` and `annotationsLib.raml` depend on each other,  we could potentially consider the following paths for `typeLib.raml`: `resourceTypes.annotations.types`, `resourceTypes.annotations.types.annotations.types`,  etc.
But for all of these paths there, obviously, exist a shorter one, that's why we do not take them into consideration.

Now, with the path sets in hands, lets select identifiers.

The choice is obvious for `resourceTypesLib.raml`, as its path set has only one element.

For `typesLib.raml` we select `types` as it is the shortest one.

For `annotationsLib.raml` the shortest paths are `resourceTypes.annotations` and `types.annotations`. We take `resourceTypes.annotations` as it is lexically smaller.
