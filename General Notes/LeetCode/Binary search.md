# lower bound and upper bound
### `std::lower_bound` / `std::upper_bound` with custom comparator

**Signatures:**

cpp

```cpp
lower_bound(first, last, value, comp);  // comp(*it, value)
upper_bound(first, last, value, comp);  // comp(value, *it)
```

**What they return:**

- `lower_bound` → first element for which `comp(*it, value)` is `false` (first elem NOT "less than" value)
- `upper_bound` → first element for which `comp(value, *it)` is `true` (first elem strictly "greater than" value)