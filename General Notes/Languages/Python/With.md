
```python
with expression as variable:
    `# code block`
```


- `expression` must evaluate to a context manager 
	- Context manager: an object with `__enter__()` and `__exit__()` methods).

- `variable` is optional — it’s the result of `expression.__enter__()`.

- with ... as ...
	- `__enter__()` function is called.
	- If anything goes wrong, `__exit__()` is called to cleanup


**Eg. opening a file**

```python
with open('example.txt', 'r') as file:
    content = file.read()
```

```python
file = open('example.txt', 'r')
try:
    content = file.read()
finally:
    file.close()
```


Basically, the point is to make sure that file closes at the end, even if something goes wrong during the read().