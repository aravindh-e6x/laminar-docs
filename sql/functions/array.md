---
sidebar_position: 6
---

# Array Functions

Functions for working with array data types.

## Array Construction

### array[elements] / ARRAY[elements]
Creates an array from elements.
```sql
SELECT array[1, 2, 3]; -- Returns: [1, 2, 3]
SELECT ARRAY['a', 'b', 'c']; -- Returns: ['a', 'b', 'c']
```

### make_array(element1, element2, ...)
Creates an array from arguments.
```sql
SELECT make_array(1, 2, 3); -- Returns: [1, 2, 3]
SELECT make_array('a', 'b', 'c'); -- Returns: ['a', 'b', 'c']
```

### array_repeat(element, count)
Creates array by repeating element.
```sql
SELECT array_repeat(5, 3); -- Returns: [5, 5, 5]
SELECT array_repeat('x', 4); -- Returns: ['x', 'x', 'x', 'x']
```

### range(start, stop [, step])
Generates array of integers.
```sql
SELECT range(1, 5); -- Returns: [1, 2, 3, 4]
SELECT range(0, 10, 2); -- Returns: [0, 2, 4, 6, 8]
SELECT range(5, 1, -1); -- Returns: [5, 4, 3, 2]
```

### generate_series(start, stop [, step])
Generates series as array.
```sql
SELECT generate_series(1, 5); -- Returns: [1, 2, 3, 4, 5]
SELECT generate_series(0, 10, 2); -- Returns: [0, 2, 4, 6, 8, 10]
```

## Array Information

### array_length(array) / array_dims(array) / cardinality(array)
Returns array length.
```sql
SELECT array_length([1, 2, 3]); -- Returns: 3
SELECT cardinality([1, 2, 3]); -- Returns: 3
```

### array_ndims(array)
Returns number of dimensions.
```sql
SELECT array_ndims([1, 2, 3]); -- Returns: 1
SELECT array_ndims([[1, 2], [3, 4]]); -- Returns: 2
```

### array_empty(array) / empty(array)
Checks if array is empty.
```sql
SELECT array_empty([]); -- Returns: true
SELECT array_empty([1, 2]); -- Returns: false
```

## Array Element Access

### array[index]
Accesses element by index (1-based).
```sql
SELECT ([1, 2, 3])[1]; -- Returns: 1
SELECT ([1, 2, 3])[2]; -- Returns: 2
```

### array_element(array, index)
Accesses element by index.
```sql
SELECT array_element([1, 2, 3], 2); -- Returns: 2
```

### array_slice(array, start, end)
Extracts array slice.
```sql
SELECT array_slice([1, 2, 3, 4, 5], 2, 4); -- Returns: [2, 3, 4]
SELECT array_slice([1, 2, 3, 4, 5], -3, -1); -- Returns: [3, 4, 5]
```

### array_pop_front(array)
Returns array without first element.
```sql
SELECT array_pop_front([1, 2, 3]); -- Returns: [2, 3]
```

### array_pop_back(array)
Returns array without last element.
```sql
SELECT array_pop_back([1, 2, 3]); -- Returns: [1, 2]
```

## Array Modification

### array_append(array, element) / array_push_back(array, element)
Appends element to array.
```sql
SELECT array_append([1, 2], 3); -- Returns: [1, 2, 3]
SELECT array_push_back([1, 2], 3); -- Returns: [1, 2, 3]
```

### array_prepend(element, array) / array_push_front(array, element)
Prepends element to array.
```sql
SELECT array_prepend(0, [1, 2]); -- Returns: [0, 1, 2]
SELECT array_push_front([1, 2], 0); -- Returns: [0, 1, 2]
```

### array_concat(array1, array2, ...) / array_cat(array1, array2)
Concatenates arrays.
```sql
SELECT array_concat([1, 2], [3, 4]); -- Returns: [1, 2, 3, 4]
SELECT array_cat([1, 2], [3, 4], [5, 6]); -- Returns: [1, 2, 3, 4, 5, 6]
```

### array_resize(array, size [, fill])
Resizes array to specified size.
```sql
SELECT array_resize([1, 2, 3], 5, 0); -- Returns: [1, 2, 3, 0, 0]
SELECT array_resize([1, 2, 3], 2); -- Returns: [1, 2]
```

### array_remove(array, element)
Removes all occurrences of element.
```sql
SELECT array_remove([1, 2, 3, 2], 2); -- Returns: [1, 3]
```

### array_remove_n(array, element, n)
Removes first n occurrences of element.
```sql
SELECT array_remove_n([1, 2, 3, 2, 2], 2, 2); -- Returns: [1, 3, 2]
```

### array_remove_all(array, element)
Removes all occurrences of element.
```sql
SELECT array_remove_all([1, 2, 3, 2], 2); -- Returns: [1, 3]
```

### array_replace(array, from, to)
Replaces all occurrences.
```sql
SELECT array_replace([1, 2, 3, 2], 2, 5); -- Returns: [1, 5, 3, 5]
```

### array_replace_n(array, from, to, n)
Replaces first n occurrences.
```sql
SELECT array_replace_n([1, 2, 3, 2], 2, 5, 1); -- Returns: [1, 5, 3, 2]
```

### array_replace_all(array, from, to)
Replaces all occurrences.
```sql
SELECT array_replace_all([1, 2, 3, 2], 2, 5); -- Returns: [1, 5, 3, 5]
```

## Array Searching

### array_contains(array, element) / list_contains(array, element)
Checks if array contains element.
```sql
SELECT array_contains([1, 2, 3], 2); -- Returns: true
SELECT array_contains([1, 2, 3], 4); -- Returns: false
```

### array_has(array, element) / list_has(array, element)
Alias for array_contains.
```sql
SELECT array_has([1, 2, 3], 2); -- Returns: true
```

### array_has_any(array1, array2)
Checks if arrays have any common element.
```sql
SELECT array_has_any([1, 2, 3], [3, 4, 5]); -- Returns: true
SELECT array_has_any([1, 2, 3], [4, 5, 6]); -- Returns: false
```

### array_has_all(array1, array2)
Checks if array1 contains all elements of array2.
```sql
SELECT array_has_all([1, 2, 3, 4], [2, 3]); -- Returns: true
SELECT array_has_all([1, 2, 3], [2, 4]); -- Returns: false
```

### array_position(array, element) / array_indexof(array, element) / list_position(array, element)
Returns position of element (1-based).
```sql
SELECT array_position([1, 2, 3], 2); -- Returns: 2
SELECT array_position([1, 2, 3], 4); -- Returns: NULL
```

### array_positions(array, element) / list_positions(array, element)
Returns all positions of element.
```sql
SELECT array_positions([1, 2, 3, 2], 2); -- Returns: [2, 4]
```

## Array Transformation

### array_reverse(array) / list_reverse(array)
Reverses array.
```sql
SELECT array_reverse([1, 2, 3]); -- Returns: [3, 2, 1]
```

### array_sort(array [, desc [, nulls_first]]) / list_sort(array)
Sorts array.
```sql
SELECT array_sort([3, 1, 2]); -- Returns: [1, 2, 3]
SELECT array_sort([3, 1, 2], true); -- Returns: [3, 2, 1] (descending)
```

### array_distinct(array) / list_distinct(array)
Returns distinct elements.
```sql
SELECT array_distinct([1, 2, 2, 3, 3]); -- Returns: [1, 2, 3]
```

### flatten(array)
Flattens nested arrays by one level.
```sql
SELECT flatten([[1, 2], [3, 4]]); -- Returns: [1, 2, 3, 4]
SELECT flatten([[[1]], [[2]]]); -- Returns: [[1], [2]]
```

## Array Set Operations

### array_union(array1, array2)
Returns union of arrays (distinct elements).
```sql
SELECT array_union([1, 2, 3], [2, 3, 4]); -- Returns: [1, 2, 3, 4]
```

### array_intersect(array1, array2)
Returns intersection of arrays.
```sql
SELECT array_intersect([1, 2, 3], [2, 3, 4]); -- Returns: [2, 3]
```

### array_except(array1, array2)
Returns elements in array1 but not in array2.
```sql
SELECT array_except([1, 2, 3], [2, 3, 4]); -- Returns: [1]
```

## Array Aggregation

### array_min(array) / list_min(array)
Returns minimum element.
```sql
SELECT array_min([3, 1, 2]); -- Returns: 1
```

### array_max(array) / list_max(array)
Returns maximum element.
```sql
SELECT array_max([3, 1, 2]); -- Returns: 3
```

### array_sum(array) / list_sum(array)
Returns sum of elements.
```sql
SELECT array_sum([1, 2, 3]); -- Returns: 6
```

### array_mean(array) / array_avg(array) / list_mean(array) / list_avg(array)
Returns average of elements.
```sql
SELECT array_mean([1, 2, 3]); -- Returns: 2.0
```

### array_product(array) / list_product(array)
Returns product of elements.
```sql
SELECT array_product([2, 3, 4]); -- Returns: 24
```

## String Array Functions

### array_to_string(array, delimiter [, null_string]) / array_join(array, delimiter)
Joins array elements into string.
```sql
SELECT array_to_string(['a', 'b', 'c'], ','); -- Returns: 'a,b,c'
SELECT array_to_string(['a', NULL, 'c'], ',', 'NULL'); -- Returns: 'a,NULL,c'
```

### string_to_array(string, delimiter [, null_string]) / string_to_list(string, delimiter)
Splits string into array.
```sql
SELECT string_to_array('a,b,c', ','); -- Returns: ['a', 'b', 'c']
SELECT string_to_array('a,,c', ',', ''); -- Returns: ['a', NULL, 'c']
```

## Usage Examples

### Working with tags
```sql
-- Find products with specific tags
SELECT * FROM products
WHERE array_contains(tags, 'electronics');

-- Find products with any of the specified tags
SELECT * FROM products
WHERE array_has_any(tags, ['electronics', 'computers']);
```

### Array aggregation
```sql
-- Collect values into array
SELECT 
    category,
    array_agg(product_name) as products
FROM products
GROUP BY category;
```

### Array manipulation
```sql
-- Combine and deduplicate arrays
SELECT 
    array_distinct(
        array_concat(
            old_tags,
            new_tags
        )
    ) as all_tags
FROM product_updates;
```

### Unnesting arrays
```sql
-- Expand array into rows
SELECT 
    id,
    unnest(tags) as tag
FROM products;
```