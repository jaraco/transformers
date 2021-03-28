# Example in CPython

[regex _compile flags](https://github.com/python/cpython/blob/af50c84643ce21cfbdfdabbdfae6bd5e1368c542/Lib/re.py#L282-L283)

```
def _compile(pattern, flags):
    # internal: compile pattern
    if isinstance(flags, RegexFlag):
        flags = flags.value
    ...
```
