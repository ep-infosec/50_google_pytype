# Overlays

<!--* freshness: { owner: 'rechen' reviewed: '2021-12-09' } *-->

<!--ts-->
   * [Overlays](#overlays)
      * [Introduction](#introduction)
      * [Example](#example)
      * [Mechanics](#mechanics)
      * [Adding an overlay](#adding-an-overlay)

<!-- Added by: rechen, at: 2022-06-22T23:51-07:00 -->

<!--te-->

## Introduction

pytype sometimes needs extra type information or needs to perform complex
operations that cannot be expressed in a pyi file. (For more details on
motivation, see [Special Builtins][special-builtins]). In such cases, we write
an *overlay* to directly manipulate abstract values.

## Example

A common pattern in Python code is to check the runtime version using
`sys.version_info`. In order to handle version checks properly, pytype needs to
load the value of `sys.version_info`, not just its type. To achieve this, we
write an overlay that maps `version_info` to a method that will directly build
the version tuple:

```python
class SysOverlay(overlay.Overlay):
  """A custom overlay for the 'sys' module."""

  def __init__(self, ctx):
    member_map = {
        "version_info": build_version_info
    }
    [...]
```

And the build method has access to the context's `python_version` attribute:

```python
def build_version_info(ctx):
  [...]
  version = []
  # major, minor
  for i in ctx.python_version:
    version.append(ctx.convert.constant_to_var(i))
  [...]
```

## Mechanics

When pytype goes to resolve an import, it will first [check][overlay-check] for
an overlay and, if one is present, use it in preference to loading the pyi.
Overlays inherit from the [overlay.Overlay][overlay.Overlay] class, a subclass
of abstract.Module that [overrides][member-conversion] member lookup so that
when a member name is present in the overlay map, the representation of that
member is constructed by calling the constructor specified in the map. The
constructor can be any callable that accepts a context instance and returns an
abstract.BaseValue instance.

## Adding an overlay

1.  If a file named `overlays/{module}_overlay.py` does not yet exist for the
    module in question, create one and add the following boilerplate (replace
    `foo` with the module name):

    ```python
    from pytype.overlays import overlay

    class FooOverlay(overlay.Overlay):

      def __init__(self, ctx):
        member_map = {}
        ast = ctx.loader.import_name("foo")
        super().__init__(ctx, "foo", member_map, ast)
    ```

    Then add the new overlay to [overlay_dict][overlay_dict], and create a new
    target for the file in [`overlays/CMakeLists.txt`][overlays-cmake].

1.  In the `{Module}Overlay` initializer, add an entry to `member_map` for each
    new member. The key should be the member name and the value the constructor.

1.  Implement the new module members! The existing overlays contain plenty of
    examples of how to do this.

[overlays-cmake]: https://github.com/google/pytype/blob/main/pytype/overlays/CMakeLists.txt

[member-conversion]: https://github.com/google/pytype/blob/2f2a1483751171421490c352f05955655ea572fa/pytype/overlay.py#L45

[overlay-check]: https://github.com/google/pytype/blob/2f2a1483751171421490c352f05955655ea572fa/pytype/vm.py#L1569-L1580

[overlay_dict]: https://github.com/google/pytype/blob/main/pytype/overlay_dict.py

[overlay.Overlay]: https://github.com/google/pytype/blob/2f2a1483751171421490c352f05955655ea572fa/pytype/overlay.py#L6

[special-builtins]: special_builtins.md
