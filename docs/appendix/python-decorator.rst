==============================
Why does Decorator Need @wraps
==============================


``@wraps`` preserve attributes of the original function, otherwise attributes
of the decorated function will be replaced by **wrapper function**. For example

Without ``@wraps``

.. code-block:: python

    >>> def decorator(func):
    ...     def wrapper(*args, **kwargs):
    ...         print('wrap function')
    ...         return func(*args, **kwargs)
    ...     return wrapper
    ...
    >>> @decorator
    ... def example(*a, **kw):
    ...     pass
    ...
    >>> example.__name__  # attr of function lose
    'wrapper'

With ``@wraps``

.. code-block:: python

    >>> from functools import wraps
    >>> def decorator(func):
    ...     @wraps(func)
    ...     def wrapper(*args, **kwargs):
    ...         print('wrap function')
    ...         return func(*args, **kwargs)
    ...     return wrapper
    ...
    >>> @decorator
    ... def example(*a, **kw):
    ...     pass
    ...
    >>> example.__name__  # attr of function preserve
    'example'
