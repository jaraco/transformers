```python
from typing import Optional, Iterable, Union
class backports:
    transforming_annotations = lambda func: func
class Item: pass
always_iterable = lambda: None
```

# Problem

Many functions exhibit the following pattern.


```python
def do_something(a: Optional[float] = None, b: float = 0) -> str:
    if a is None:
        a = 0

    # do "something" with `a` and `b`, assuming `a` is an integer, producing result
    result = (a ** 2) + b
    
    return str(result)
```

This approach presents several problems or areas for improvement:

- The initial clause of the function masks the type transformation, and mutates the variable value.
- Concerns that could be separated (and re-used) aren't readily separated.

## Variable masking

By selectively re-assigning the value of `a`, the function signature needs to support the more complex signature, adding complexity and cognitive burden. If the transformation is more than a simple two-line transform of a single parameter, this burden is even greater. Moreover, this approach fails to protect the programmer or the static analyzer from the possibility of interacting effects.


```python
def do_something(a, b):
    if a is None:
        a = 0 if b else 1  # a and b are entangled in this branch of the code
```

Conversely, if the transformation of `a` could be isolated from the body of the function, the complexity and cognitive burden could be isolated to that value.


```python
def none_is_zero(val: Optional[float]):
    return val if val is not None else 0

def do_something(a, b):
    a = none_is_zero(a)  # no chance of entagling b
```

However, this approach still requires `a` to be re-assigned and introduces potential for logic before and after the transformation, not clearly communicating that this transformation could always be done at the function's invocation.

## Separation of Concerns

The `do_something` function is addressing several concerns:

- Accepting two types for `a` but collapsing them to a `float`.
- Performing the essential part of the function.
- Transforming the result into a particular output type.

This pattern is common and appears even in CPython code ([regex _compile flags](https://github.com/python/cpython/blob/af50c84643ce21cfbdfdabbdfae6bd5e1368c542/Lib/re.py#L282-L283)).

```python
def _compile(pattern, flags: [int, RegexFlag]):
    # internal: compile pattern
    if isinstance(flags, RegexFlag):
        flags = flags.value
    ...
```

In fact, one might be tempted to separate essential concerns from the other two transformations.


```python
def make_str(val: float) -> str:
    return str(val)


def _do_something_essential(a: float, b: float):
    """Do 'something' with `a' and `b`"""
    return (a ** 2) + b


def do_something(a: Optional[float] = None, b: float = 0) -> str:
    if a is None:
        a = 0
    
    return make_str(do_something(a, b))
```

Note the elegance and simplicity of `_do_something_essential`.

But already, this approach is getting a little unwieldy and still doesn't help much if one wishes to re-use the parameter transform (`a is None -> a = 0`) or the return value transform.

# Proposed

Inspired by the simplicity and power of decorators, what if Python could designate transformation functions to be applied to parameters and return values. Such an approach promises to address the concerns listed above. In fact, such an approach could conceivably be implemented with a decorator today.


```python
def transform_result(xform):
    # stubbed
    return lambda func: func

def transform_param(param, xform):
    # stubbed
    return lambda func: func

@transform_result(make_str)
@transform_param('a', none_is_zero)
def do_something(a: float, b: float):
    """Do 'something' with `a' and `b`"""
    return (a ** 2) + b
```

This approach, however, is inelegant in that the parameter name has to be repeated and the type annotations may not recognize the effect of the transform. What if the function's annotations could be used:


```python
def do_something(a: none_is_zero, b: float) -> make_str:
    """Do 'something' with `a' and `b`"""
    return (a ** 2) + b
```

The annotations here are arbitrary callables that themselves have type annotations and are recognized as transforming functions.

## Advantages

This approach has a number of benefits, addressing the problems above:

- Elegant, simple declaration of intended behavior.
- Clear separation of concerns.
- Avoids re-writing variables in the scope.
- Easy re-use of transformations.
- Explicit type transformation.

## Challenges


### Compatibility

Older Python versions wouldn't have this functionality, but a compatibility shim could likely be implemented that would provide the functionality for older Pythons.


```python
@backports.transforming_annotations
def do_something(a: none_is_zero, b: float) -> make_str:
    """Do 'something' with `a' and `b`"""
    return (a ** 2) + b
```

### Ambiguity between types and transforms

Perhaps the biggest challenge is that `int` is both a simple type annotation and a callable that transforms certain types to another type. One might be tempted to use or even expect to be able to use `int` to transform an incoming `str` to an integer. Or similarly for `str` to do the reverse.


```python
def do_something(a: none_is_zero, b: float) -> str:  # do you mean make it a str or expect a str?
    return (a ** 2) + b
```

I can imagine a few approaches to address this concern.

- Require that transforming functions be explicitly created, such as with `make_str` above.
- Provide a wrapping helper that specifies that a type is to be used as a transform (`-> transform(str)`).
- Provide a wrapping helper or explicit types for non-transforming type declarations (e.g. `Int` or `strict(int)`).

This last option probably has the most compatibility concerns, but would allow constructors to also double as transformers.

# Questions

- Has this approach been considered previously (where)?
- Are there other challenges I haven't recognized?
- Other feedback or questions?

# Other examples

## always iterable

Consider the following common case where a parameter can be a single item or an iterable of items.


```python
def process(item: Item):
    "stubbed"
    
def process_items(item_or_items: Optional[Union[Item, Iterable[Item]]]):
    if item_or_items is None:
        item_or_items = []
    if not hasattr(item_or_items, '__iter__'):
        item_or_items = [item_or_items]
    return (process(item) for item in item_or_items)
```

The first four lines of `process_items` (or some variation of that) can be found many places and is common enough that [more_itertools implements that behavior](https://more-itertools.readthedocs.io/en/stable/api.html#more_itertools.always_iterable). Applying that functionality, the code could be implemented thus:


```python
def process_items(items: always_iterable):
    return (process(item) for item in items)
```
