Fluid ViewHelper Compile Gap Filler
===================================

> Tiny library designed to fill the gap between TYPO3 CMS versions 7.6 and 8.x
 
Why do I need it?
-----------------

On Standalone Fluid 1.1.0 and upwards, ViewHelper compiling is assisted by Traits which allow the ViewHelper
to simply include a Trait and automatically become compilable. However, due to the signature differences it is
not possible to implement a `compile()` method on TYPO3 CMS ViewHelpers when those ViewHelpers belong to
packages which must operate on both TYPO3 CMS 7.6 and 8.x. There are currently two such Traits:

* `CompileWithRenderStatic` which compiles the ViewHelper by inserting a direct call to the `renderStatic`
  function on the ViewHelper (which you must then implement!)
* `CompileWithContentArgumentAndRenderStatic` which also compiles to a `renderStatic` call but with the
  added behavior that the first **optional** argument is used as a "content argument" which means that if
  that argument is empty, `renderChildren()` is called to retrieve a value. It achieves this by creating
  an alternative closure to replace the `renderChildrenClosure`. This is a fairly wide-spread use case in
  TYPO3 CMS and extensions and the benefit of the Trait is that you *do not have to check the arguments
  array for a set value - you can instead indiscriminately call the `$renderChildrenClosure()` that is
  provided for the `renderStatic` method and it will return the right value*.

Essentially: you need this bridge library for cross-version TYPO3 CMS Fluid ViewHelper compiling - and
you "need" (read: want) it to drastically decrease the size of and code duplication in your ViewHelpers.

What does it do?
----------------

Since Fluid Standalone changes the signature of the `compile()` method on ViewHelpers from requiring an
`AbstractNode` to requiring the proper `ViewHelperNode`, as well as the new namespace location for the
`TemplateCompiler` class a bridging library is required to provide Traits with the exact same class name
but different signatures depending on TYPO3 CMS version - and do so without breaking the strict signature
requirements of PHP 7.0+ (which is required by TYPO3 CMS 8.x).

Therefore:

* On TYPO3 CMS 8.4 and above, the package provides an alias for the corresponding Traits from Fluid
  Standalone version 1.1.0+. This allows ViewHelpers to `use` the bridge Trait with the exact same effect
  as using the "real" Trait.
* On TYPO3 CMS 7.6 the package provides a custom implementation of the Traits which are partially rewritten
  from the Fluid Standalone 1.1.0 implementations to be suitable for this version and this version only.

This means that by adding this package as a dependency and using the bridge versions of Traits your TYPO3
extension can provide perfectly compilable ViewHelpers (when they fit the two standard use cases) and have
them work equally well on TYPO3 CMS 7.6 and 8.x.

Installation
------------

To include this dependency you **must** do so with a version restriction that will match either the `1.0`
**or** the `2.0` branch of this package - or use a wildcard. This allows Composer to select the proper
version of the bridge library based on the selected version of TYPO3 CMS.

Valid require statements could be for example:

```
composer require namelesscoder/typo3-cms-fluid-gap:*
```

```
composer require "namelesscoder/typo3-cms-fluid-gap:>=1.0 <2.1"
```

```
composer require "namelesscoder/typo3-cms-fluid-gap:1.0 - 2.0"
```

Depending on your preferred strictness you may choose any of the above but the latter two are recommended
since they constrain the versions specifically to 1.0.x OR 2.0.x.

Special note about `CompilableInterface`
----------------------------------------

Small bit of history: on TYPO3 CMS 7.6 an interface exists which instructs the Fluid TemplateCompiler that a
ViewHelper supports compiling (e.g. implements the `compile()` method). Since Fluid Standalone and thus
TYPO3 CMS 8.x this interface no longer exists - it has been replaced with a fallback compiling mechanism
combined with the requirement to intentionally disable compiling if a ViewHelper is wholly uncompilable.

This means that you **will be implementing an interface which is deprecated on one of your supported TYPO3
CMS versions, namely 8.x**. Unfortunately this cannot be avoided.

Furthermore it requires you to be prepared to remove this implementation of the interface once your extension
no longer supports TYPO3 CMS 7.6. And it requires you to drop compiling support on TYPO3 CMS 7.6 if/when your
extension must begin to support a TYPO3 CMS version on which the temporary aliases for the deprecated
interface get removed.

So this is a caveat that you must be prepared to address later on.

An example of a proper implementation working on TYPO3 CMS 7.6 and 8.x but registering as a deprecated
usage when running on TYPO3 CMS 8.x:

```php
namespae My\Extension\ViewHelpers;

class MySpecialViewHelper implements \TYPO3\CMS\Fluid\Core\ViewHelper\CompilableInterface
{    
    use \NamelessCoder\FluidGap\Traits\CompileWithRenderStatic;
    // when using a "content argument" taken from `renderChildren()` if argument is empty:
    // use \NamelessCoder\FluidGap\Traits\CompileWithContentArgumentAndRenderStatic;
    
    public static function renderStatic(
        array $arguments, 
        \Closure $renderChildrenClosure, 
        TYPO3\CMS\Fluid\Core\Rendering\RenderingContextInterface $renderingContext
    ) {
        return 'This is what my ViewHelper returns';
    }
}

```

Side note: the `RenderingContextInterface` is also deprectaed on TYPO3 CMS 8.x and will also need to be
migrated once that gets removed. TYPO3 CMS 8.4 appears to receive a migration analysis tool which will
catch any such occurrences in your own and third party extensions alike.
