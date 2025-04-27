#python-jail

Дан jail на SimpleEval. В нем вот такой список импортированных функций

```python
import builtins
import hashlib
import math
import random

ALLOWED_BUILTINS = (
    'abs',
    'all',
    'any',
    'bin',
    'bool',
    'chr',
    'complex',
    'divmod',
    'enumerate',
    'filter',
    'float',
    'format',
    'hash',
    'hex',
    'int',
    'iter',
    'len',
    'list',
    'max',
    'min',
    'next',
    'ord',
    'pow',
    'range',
    'reversed',
    'round',
    'sorted',
    'str',
    'sum',
    'zip',
)

def get_allowed_functions():
    """Get the dictionary of allowed functions from various modules."""
    functions = {
        **{fname: func for fname, func in builtins.__dict__.items() if fname in ALLOWED_BUILTINS},
        **{fname: func for fname, func in hashlib.__dict__.items() if not fname.startswith('__')},
        **{fname: func for fname, func in math.__dict__.items() if not fname.startswith('__')},
        **{fname: func for fname, func in random.__dict__.items() if not fname.startswith('__')},
    }
    
    # Remove potentially dangerous functions
    functions.pop('os', None)
    functions.pop('sys', None)
    
    return functions 
```

Если его вывести, то можно обнаружить импортированный `_os`, который является по факту модулем os. Делаем реверс шелл и гг