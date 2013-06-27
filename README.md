# Plan for a Modernizr plugin for RequireJS

This isn't a thing yet. Just an idea.

***Edit:** since my first draft, I've spotted [require-is](https://github.com/guybedford/require-is), which looks like it performs similar logic, but without the Modernizr integration. A helpful starting point, maybe...?*

## Aim

[RequireJS](http://requirejs.org/) is a JavaScript module loader and dependency manager.

[Modernizr](http://modernizr.com) is a browser feature detection library.

While RequireJS normally manages *software* and *asset* dependencies between modules, modules targeting browser environments often also have dependencies on certain *browser capabilities*.

For example, a module which provides local information may depend on the Geolocation API (as well as any of its software dependencies).

**require-modernizr** would aim to enable dependencies on browser capabilities to be handled in the same way as software dependencies, using AMD module syntax with RequireJS and the r.js optimiser.

An auxiliary aim is to encourage developers to consider browser capabilities to be part of their 'system' and embrace working with variability between browsers, rather than considering it an inconvenience and an afterthought.


## Usage

### Basic Example

An AMD module which has both software and browser dependencies might look like this:

```javascript
require(
    ['jquery', 'lib/maps', 'modernizr!geolocation'],
    function ($, maps, hasGeolocation) {
        // Code here only executed if all software dependencies are satisfied
        // AND all browser dependencies are satisfied (via feature detects)
    }
);
```

Notes:

* Browser dependencies are included in the usual dependency list
* They use the syntax: `modernizr!<property>`
* The detect result, `Modernizr[property]`, is passed to the module's function as an argument (the naming of this argument is arbitrary)
* This should always evaluate to `true` (the function should not be executed otherwise), but in some cases this may be an object with additional properties, e.g. in the case of `Modernizr.audio`
* I propose the convention that browser capabilities should follow software dependencies (and other asset dependencies, such as [text](http://requirejs.org/docs/api.html#text))

### Fallbacks

In the example above, the module is executed if all dependencies are satisfied, but not if the browser dependencies prove not to be supported. This "feature gating" technique is efficient and easy to maintain.

However, in some cases it is desirable to provide a fallback or polyfill in the case where feature detection fails.

A fallback should be provided as a separate module with this syntax:

```javascript
require(
    ['jquery', 'lib/nomaps', 'modernizr!no-geolocation'],
    function ($, nomaps, hasGeolocation) {
        // Provide fallback
    }
);
```

Notes:

* The syntax: `modernizr!no-<property>` is used, mirroring the class names used by Modernizr
* Again, the detect result, `Modernizr[property]`, is passed to the module's function as an argument
* This should always evaluate to `false`, doesn't need to be checked within the module and is just provided for consistency
* This may have different software dependencies from the 'positive' detection case
* It is guaranteed that only one or other of these modules will run; but it may be that neither runs if the relevant software dependencies are not satisfied


## Behaviour

### Basic Behaviour

* The use of a `modernizr!<property>` dependency implies a dependency on Modernizr, and more specifically the named feature detect
* If the feature detection script cannot be loaded, the module is not executed and an error is thrown as usual in RequireJS
* If the feature detect *can* be loaded but the detect's result is `false`, the module is not executed and no error is thrown
* If the detect's result is `false`, other modules which cite this module as a dependency are not executed either

### Optimiser

Detection results are not known when the optimiser runs. Therefore, when a module with a `modernizr!<property>` dependency is encountered by the r.js optimiser:

* The Modernizr property name is 'memorised'
* The module is included in the optimized build, wrapped in a conditional to ensure it only runs if its browser dependencies are satisfied by the relevant Modernizr feature detect(s):

    ```javascript
    if (Modernizr.geolocation) {
        (function ($, maps, hasGeolocation) {
            // Module implementation
        }());
    }
    ```

* A Modernizr build is created including detects for all browser dependencies of modules in the build and prepended to optimized output

### Multiple build-out

As an advanced feature, the optimiser could have an optional mode in which modules with browser dependencies are *not* included in the main optimized build.

Instead:

* Multiple additional 'bundle' scripts are created: one for each possible permutation of feature detection results
* The main optimized build still includes the relevant feature detects
* When executed in the browser, the main optimized build uses the feature detection results to construct the path of the bundle which represents its detection results
* It loads this script, giving a lean collection of modules which are all compatible with the current browser environment


## Additional considerations

### Asynchronous detects

Some of Modernizr's feature detects are asynchronous. *require-modernizr* must account for this by ensuring all feature detects for a module's browser dependencies have completed before determining whether to execute the module's code.
