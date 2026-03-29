3 kinds.
# Standard (find exact value)
```cpp
int binarySearch(vector<int>& nums, int target) {
    int l = 0;
    int r = nums.size() - 1;

    while (l <= r) {
        int mid = l + (r - l) / 2;

        if (nums[mid] == target)
            return mid;

        if (nums[mid] < target)
            l = mid + 1;
        else
            r = mid - 1;
    }

    return -1;
}
```

# Lower bound/first valid
```cpp
int l = min_value;
int r = max_value;

while (l <= r) {
    int mid = l + (r - l) / 2;

    if (valid(mid))
        r = mid - 1;
    else
        l = mid + 1;
}

return l;
```

- Take note of returning l!!! And not mid!!!
- May need to modify. see **find minimum in rotated sorted array**

# Upper Bound / Last Valid
```cpp
int l = min_value;
int r = max_value;

while (l <= r) {
    int mid = l + (r - l) / 2;

    if (valid(mid))
        l = mid + 1;
    else
        r = mid - 1;
}

return r;
```