
## Global Info

* Definition system

## Spec Info

* AST

* Project structure (for path completion)

* Master filter for Overlay case

## Completion Kind

* Type expression

* Scalar value of ${ramlType}.${property} (except type expression)

* Scalar value of ${ramlType}.${property} from new line. (Allows value and annotations, ('strict' for examples))

* Object value of ${ramlType}.${property}
  * Method.body
  * Method.responses
  * Method.headers
  * Resource.uriParameters

* Property name of ${ramlType}

* Parameter name of ${template}

* Parameter name of ${securityScheme}

* ${DataType} instance property name
* Value of ${DataType}.${property}
  * example
  * enum
  * defaultValue
  * Annotation type reference
