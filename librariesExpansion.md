## Libraries Exapnsion Procedure

### RAML Components and Their Dependencies

By **RAML component** we understand any resource, resource type, trait, type, annotation type and security scheme.

We say that RAML component `A` depends on RAML component `B` if one of the two conditions is satisfied:

* `B` is referenced in definition of `A`
* there exists RAML component `C` referenced in defenition of `A`, such that `C` depends on `B`

For example:

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

* The dependency relation between RAML components is transitive i.e. if `A` depends on `B` and `B` depends on `C` then `A` depends on `C`.

* For each RAML component its dependencies set can be algorythmically calculated.

* For each pair of RAML components the question about one being dependent on another can be algorythmically solved.

By **dependencies set** of RAML specification we understand a union of depndencies sets of components declared in the specification.

### Goals

#### Libraries Expansion for a Single RAML API or Library Specification
For a RAML API or library specification provide a RAML specification, which is semantically equivalent to the expanded (in sense of templates) specification.
The resulting specification must contains its dependencies set and must not use any libraries.

#### Libraries Expansion for Extensions/Overlays Chain
For an overaly/extension chain (we consider master API as the last element of the chain) provide
an API specification which is semantically equivalent to expansion (in both senses of templates and overlays/extensions) of the chain.
The resulting API specification must not use any libraries and must contain its dependensies set.

### Implementation Sketch

#### Single Specification Libraries Expansion Procedure (SSLEP).

1. Collect a minimal set of libraries which covers dependecies set of the specification.

1. Assign a unique identifier to each library from the set.

1. Create a copy of the specification. Clean `uses` section of the copy.

1. Clean `resourceTypes`, `traits`, `types`, `annotations` and `securitySchemes` sections of the copy. For API case remove all the resources (if any) from the copy.

1. Take **outer dependencies set** of the specification, i.e. set of those dependencies, which are declared outside the specification.

1. For each RAML component from the outer dependencies set perform the four steps:

  * append as prefix identifier of library declaring the component to the component name
  
  * apply **References Patching Procedure** to all the references in the component definition:
    
	* remove library prefix (if any) from the refernce name
	
	* append as prefix identifier of library declaring the referenced component to the reference name
	
  * inline all include nodes (and fragments among them), applying the reference patching procedure to them
  
  * append the patched component to the specification copy

1. For each component declaration of the specificatbin perform the three steps:
  
  * apply the reference patching procedure
  
  * inline all include nodes (and fragments among them), applying the reference patching procedure to them
  
  * append the patched resource declaration to the copy

1. For Api specification expand the copy in sense of templates.
  
_Note 1:_ On reference patching substeps of 6 and 7 steps we consider as references those template parameters which affect types in sense of https://github.com/raml-org/raml-spec/issues/510
  
_Note 2:_ In case of libraries expantion of a single specification libraries directly used by the specification can be assigned their names as identifiers.
In such approach patching substep of step 7 becomes redundant if none of the fragments involved use libraries.
But this substep shows its usefullness in extensions/overlays case described below.


#### Libraries Expansion for Extensions/Overlays chain

In this case we apply almost the same SSELP to each element of the chain, starting from the master API.
The resulting specifications can be subjected to regular procedures of expanding overlays/extensions.

The difference between original SSLEP and the one to be applied here lays in the first two steps:

1. Apart from dependencies set of the specification in context, the libraries set taken on the first step must also cover
   dependencies sets of all the remaining chain elements.
  
2. Each time we perform the second step, we must select same identifiers for same libraries.

In fact, once steps 1 and 2 are performed for the master API, their results may be reused for the remaining elements of the chain.

_Note:_ Suppose, all libraries directly used by the master API are assigned their names as identifiers.
It may happen that one of these names is used to import some other library in one of the remaining chain elements.
In such situation identifier of the library can not be equal to the name, which it is used under in the chain element, and the patching substep of the step 7 becomes sufficient.


### Constructing Library Identifiers

#### Single Specification Case

Lets not consider fragments at first and say that a library `L` is **reachable** from a RAML specification with path `p` if one of the two conditions are satisfied

1. the spec uses `L` under name `p`

1. the spec uses some library `L'` under name `n`, such that `L` is reachable from `L'` with path `p'` and `p=n.p'`

Lets define path length as a number of segments obtained by spliting it by `.` symbol.
Thus, if it happens that on some step of path construction library name contains `.` symbol, this name contributes more then one segment to the path.

In a simple case, when we do not involve fragments which use libraries,
each library from the set constructed in the first step of SSLEP is reachable with some path from the specification.

For each library we consider a set of paths it is reachable with and take the shortest one as the library identifier.
If there are more then one paths with minimal length, we take the lexically smallest one.

Now lets involve libraries used by fragments into definition of reachable libraries.
For a specification we sort all fragments it includes by order of first inclusion and, thus, ascribe indexes to these fragments.
Note that the same fragment may obtain different indexes in context of different specifications.

Now we can state the complete definition of reachable libraries.

A library `L` is **reachable** from a RAML specification with path `p` if one of the four conditions are satisfied

1. the spec uses `L` under name `p`

1. the spec uses some library `L'` under name `n`, such taht `L` is reachable from `L'` with path `p'` and `p=n.p'` 

1. some fragment included by the spec uses `L` under name `n` and `p=FR.{fragmentIndex}.n`

1. some fragment included by the spec uses some library `L'` under name `n`, such taht `L` is reachable from `L'` with path `p'` and `p=FR.{fragmentIndex}.n.p'`

#### Extensions/Overlays Case

For libraries reachable from master API we construct identifiers just the same way as in Single Specification Case.
When trying to consequently apply the same approach to the remaining elements of the extensions/overlays chain, we may obtain collisions as

```
L1 is reachable from Master with path p
L2 is reachable from Extension with path p
L1 != L2
```

In order to resolve such collisions, we can append appropriate integer indexes as postfixes to the newly obtained conflicting identifiers.

 
