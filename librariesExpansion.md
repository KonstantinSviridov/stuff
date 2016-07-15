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

By **outer dependencies set** of the specification we understend a set of those dependencies, which are declared outside the specification.

### Goal

Suppose we have a valid expanded RAML API specification in our hands.
The goal is to modify the specification in such way, that:

1. the resulting specification is semantically equivalent to the input specification
2. the resulting specification contains its dependencies set in it
3. the resulting specification does not use libraries.

### Implementation Sketch


1. Collect a minimal set of libraries which covers dependecies set of the specification.

1. Assign a unique identifier to each library from the set.

1. For each RAML component from the outer dependencies set create a copy of the component and apply the four steps to the copy:

  * inline all include nodes (and fragments among them)
  
  * append as prefix identifier of library declaring the component to the copy name
  
  * apply **References Patching Procedure** to all the references in the copy definition:
    
	* remove library prefix (if any) from the refernce name
	
	* append as prefix identifier of library declaring the referenced component to the reference name
  
  * append the patched component to the specification

1. Clean `uses` section of the specification.
  
### Constructing Library Identifiers

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
