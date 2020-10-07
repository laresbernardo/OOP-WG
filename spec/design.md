---
editor_options:
  markdown:
    mode: gfm
---

# Design specification

This document presents a broad overview of the system.

## Objects

We define an object in this system as any R object with:

-   A class object attribute, a reference to the **class object**, and retrieved with `classObject()`.
-   For S3 compatibility, a class attribute, a character vector of class names.
-   Additional attributes storing **properties** defined by the class, accessible with `@`/`prop()`.

## Classes

Classes are first class objects (Req2).
A class object is a function which can be called to construct an object of that class.
It has the following components:

-   **Name**, a human-meaningful descriptor for the class.
    This is used for print method and error messages; it does not identify the class.

-   **Parent**, the class object of the parent class.
    This implies single inheritance (Req6).

-   A **constructor**, an user-facing function used to create new objects of this class.
    It always ends with a call to `newObject()` to initialize the class.
    This the function (wrapped appropriately) that represents the class.

-   **A validator**, a function that takes the object and returns `NULL` if the object is valid, otherwise a character vector of error messages (like the methods package).

-   **Properties**, a list of property objects that define object data.

Each component corresponds to an argument in `newClass()`:

``` r
newClass(
  name, 
  parent = Object, 
  constructor = function(...) newObject(...), 
  validator = function(x) NULL,
  properties = list()
)
```

For example:

``` r
Range <- newClass("Range", 
  Vector, 
  constructor = function(start, end) {
    stopifnot(is.numeric(start), is.numeric(end), end >= start)
    newObject(start = start, end = end)
  },
  validator = function(x) {
    if (x@end < x@start) {
      "end must be greater than or equal to start"
    }
  }, 
  properties = c(start = "numeric", end = "numeric")
)

Range(start = 1, end = 10)
```

### Initialization

Initializing an instance of a class with `newObject()`:

1.  Inspects the enclosing scope to find the "active" class.
2.  Creates the prototype, by either by calling the parent constructor or by creating a base type and adding `class` and `classObject` attributes to it.
3.  Validates properties then adds to prototype.
4.  Validates the complete object.

Steps 2 and 3 are similar to calling `structure()`, except that property values will be initialized and validated through the property system.

### Shortcuts

By convention, any argument that takes a class object can instead take the name of a class object in string.
The name will be used to find the class object in the calling frame.

Similarly, instead of providing a list of property objects, you can instead provide a named character vector.
For example, `c(name = "character", age = "integer")` is shorthand for `list(newProperty("name", "character"), newProperty("age", "integer"))`.

### Validation

## Properties

A property is an encapsulated component of the object state that is publicly accessible via a simple syntax.
The motivation for properties is that R users expect transparency; they want to get to the actual data inside an object and directly manipulate it to get their work done, without worrying about bespoke APIs.
Properties support typical usage while protecting encapsulation and hiding implementation details.

Compared to S3 attributes, properties are considerably stricter: a class defines the names and types of its properties.
Compared to S4 slots, properties enable an object to change its internals without breaking existing usage because it can provides a custom accessor that redirects to the new representation.
There will be built-in support for emitting deprecation messages.

Every property definition has a:

-   A **name**, used to label output for humans.
-   An optional **class**
-   An optional **accessor** function that overrides getting and setting, much like an active binding (by default, the value is stored as attribute, like S3/S4).

Property objects are created by `newProperty()`:

``` r
newProperty(
  name, 
  class = NULL, 
  accessor = NULL
)
```

While it would be tempting to support public vs. private scoping on properties, it is not obvious how to support that, because no code is more privileged than any another.
Nor is it clear whether the package should define the boundary, when packages sometimes define methods on classes defined in other packages and want to take advantage of the internal representation.
The encapsulation afforded by properties is a good compromise.

Properties of an object can be accessed using `@`/`@<-` or `prop()`/`prop<-`.
Setting the properties of an object always triggers validation.

## Generic

A generic is a function that provides an interface with implementation provided by methods.
For introspection purposes, it knows its name and the names of the arguments in its signature (the arguments considered during dispatch).

Calling `newGeneric()` defines a new generic.
It has the signature:

``` r
newGeneric(name, FUN, signature)
```

The `signature` would default to the first argument, i.e. `formals(FUN)[1]`.
The body of `FUN` would resemble S3 and S4 generics.
It might just call `UseMethod()`.

By convention, any argument that takes a generic function, can instead take the name of a generic function supplied as a string.
The name will be used to find the class object in the calling frame.

## Methods

### Creation

Methods are defined by calling `method<-(generic, signature, method)`:

``` r
method(generic, signature) <- function(x, ...) {} 
```

-   `generic` is a generic function.

-   `signature` is a single class object (single dispatch) or list of class objects (nested dispatch).

`method<-` performs validation ensuring that the method is compatible with the generic (i.e. all arguments before `...` have the same names in the same order; if the generic doesn't have `...` all arguments must be same).

Documentation will discuss the risks of defining a method when you don't own either the generic or the class.

`method<-` is designed to work at run-time (not just package build-time) so that methods can be defined when suggested packages are loaded later:

``` r
whenLoaded("pkg", {
  method(mean, pkg::A) <- function() 10
  method(sum, pkg::A) <- function() 5
})
```

### Dispatch

Dispatch will be nested, meaning that if there are multiple arguments in the generic signature, it will dispatch on the first argument, then the second.
Nested dispatch is likely easier to predict and understand compared to treating all arguments with equal precendence.
Nested dispatch is also easier to implement efficiently, because classes would effectively inherit methods, and we could implement that inheritance using environments.

For example, a `plot()` generic dispatching on `x` could be implemented like this:

``` r
plot <- function(x) {
  methods(classObject(x))$plot(x)
}
```

For multiple dispatch, we could apply the builder pattern.
While a `publish()` that publishes an object `x` to a destination `y`, dispatching on both arguments, could be:

``` r
publish <- function(x, y) {
  methods(classObject(x))$publish(x, y)
}
```

where `class(x)$publish` returns the pregenerated

``` r
function(x, y) {
  methods(classObject(y))$publish.plot(x, y)
}
```

assuming `x` is a "plot" object.

Alternatively, the generics could just call `UseMethod()`, which would gain support for nested dispatch.

# Compatibility

## S3

Since the class attribute has the same semantics as S3, S3 dispatch should be fully compatible.
The new generics should also be able to handle legacy S3 objects.

## S4

### Generics

In principle, we could modify the methods package so that S4 generics can operate on the new class definitions.
Since the new generics will fallback to S3 dispatch, they should support S4 objects just as S3 generics support them now.

### Classes

# Documentation

The primary challenge is that the availability of methods and classes depends on which packages are installed, so we will need to generate index pages that list the methods for a generic or the methods with a particular class in their signature.
The documentation blurbs could be lazily scraped at runtime to build an index, similar to how the help search index is created now.
We would then generate help pages from the index.

The index of installed methods could provide additional benefits, including introspection.
A generic could prompt the user to load a package if it fails to find an applicable method, while there is an installed, unloaded method available.
This casd would arise when a method is defined outside of the packages that define the class and generic.