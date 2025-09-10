---
sidebar_position: 4
---

# Aggregate Functions

Functions that operate on sets of values and return a single result. These are typically used with GROUP BY clauses or as window functions.

## Basic Aggregates

### count(*) / count(expression) / count(DISTINCT expression)
Counts rows or non-null values.
```sql
SELECT count(*) FROM orders; -- Count all rows
SELECT count(customer_id) FROM orders; -- Count non-null customer_ids
SELECT count(DISTINCT customer_id) FROM orders; -- Count unique customers
```

### sum(expression)
Calculates the sum of values.
```sql
SELECT sum(amount) FROM orders;
SELECT sum(DISTINCT price) FROM products;
```

### avg(expression) / mean(expression)
Calculates the average of values.
```sql
SELECT avg(amount) FROM orders;
SELECT mean(amount) FROM orders; -- Alias for avg
```

### min(expression)
Returns the minimum value.
```sql
SELECT min(amount) FROM orders;
SELECT min(created_at) FROM orders; -- Works with dates too
```

### max(expression)
Returns the maximum value.
```sql
SELECT max(amount) FROM orders;
SELECT max(created_at) FROM orders;
```

## Statistical Aggregates

### stddev(expression) / stddev_samp(expression)
Calculates sample standard deviation.
```sql
SELECT stddev(amount) FROM orders;
SELECT stddev_samp(amount) FROM orders; -- Same as stddev
```

### stddev_pop(expression)
Calculates population standard deviation.
```sql
SELECT stddev_pop(amount) FROM orders;
```

### variance(expression) / var_samp(expression)
Calculates sample variance.
```sql
SELECT variance(amount) FROM orders;
SELECT var_samp(amount) FROM orders; -- Same as variance
```

### var_pop(expression)
Calculates population variance.
```sql
SELECT var_pop(amount) FROM orders;
```

### covar_samp(y, x)
Calculates sample covariance.
```sql
SELECT covar_samp(sales, marketing_spend) FROM monthly_data;
```

### covar_pop(y, x)
Calculates population covariance.
```sql
SELECT covar_pop(sales, marketing_spend) FROM monthly_data;
```

### corr(y, x)
Calculates correlation coefficient.
```sql
SELECT corr(sales, marketing_spend) FROM monthly_data;
-- Returns value between -1 and 1
```

## Advanced Statistical Functions

### median(expression)
Calculates the median value.
```sql
SELECT median(amount) FROM orders;
```

### mode(expression)
Returns the most frequent value.
```sql
SELECT mode(category) FROM products;
```

### percentile_cont(fraction) WITHIN GROUP (ORDER BY expression)
Calculates continuous percentile.
```sql
-- Calculate 50th percentile (median)
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY amount) FROM orders;

-- Calculate 95th percentile
SELECT percentile_cont(0.95) WITHIN GROUP (ORDER BY response_time) FROM requests;
```

### percentile_disc(fraction) WITHIN GROUP (ORDER BY expression)
Calculates discrete percentile.
```sql
SELECT percentile_disc(0.5) WITHIN GROUP (ORDER BY amount) FROM orders;
```

### approx_percentile_cont(expression, percentile)
Approximate continuous percentile (faster for large datasets).
```sql
SELECT approx_percentile_cont(amount, 0.5) FROM orders;
SELECT approx_percentile_cont(amount, ARRAY[0.25, 0.5, 0.75]) FROM orders;
```

### approx_median(expression)
Approximate median (faster for large datasets).
```sql
SELECT approx_median(amount) FROM orders;
```

## Distribution Functions

### histogram(expression, num_buckets)
Creates a histogram with specified number of buckets.
```sql
SELECT histogram(amount, 10) FROM orders;
```

### width_bucket(expression, min, max, num_buckets)
Assigns values to histogram buckets.
```sql
SELECT 
    width_bucket(amount, 0, 1000, 10) as bucket,
    COUNT(*) as count
FROM orders
GROUP BY bucket
ORDER BY bucket;
```

## Array Aggregates

### array_agg(expression [ORDER BY ...])
Aggregates values into an array.
```sql
SELECT array_agg(name) FROM products;
SELECT array_agg(name ORDER BY price DESC) FROM products;
SELECT array_agg(DISTINCT category) FROM products;
```

### string_agg(expression, delimiter [ORDER BY ...])
Concatenates strings with delimiter.
```sql
SELECT string_agg(name, ', ') FROM products;
SELECT string_agg(name, ', ' ORDER BY name) FROM products;
```

## Boolean Aggregates

### bool_and(expression) / every(expression)
Returns true if all values are true.
```sql
SELECT bool_and(is_active) FROM users;
SELECT every(amount > 0) FROM orders;
```

### bool_or(expression) / any(expression) / some(expression)
Returns true if any value is true.
```sql
SELECT bool_or(is_premium) FROM users;
SELECT any(amount > 1000) FROM orders;
```

## Bitwise Aggregates

### bit_and(expression)
Bitwise AND of all values.
```sql
SELECT bit_and(flags) FROM settings;
```

### bit_or(expression)
Bitwise OR of all values.
```sql
SELECT bit_or(flags) FROM settings;
```

### bit_xor(expression)
Bitwise XOR of all values.
```sql
SELECT bit_xor(flags) FROM settings;
```

## Approximate Aggregates

### approx_distinct(expression)
Approximate count of distinct values (HyperLogLog).
```sql
SELECT approx_distinct(user_id) FROM events;
-- Much faster than COUNT(DISTINCT user_id) for large datasets
```

### approx_percentile_cont_with_weight(expression, weight, percentile)
Weighted approximate percentile.
```sql
SELECT approx_percentile_cont_with_weight(value, weight, 0.5) FROM weighted_data;
```

## Ordered Set Aggregates

### first_value(expression [ORDER BY ...])
Returns first value in ordered set.
```sql
SELECT first_value(amount ORDER BY created_at) FROM orders;
```

### last_value(expression [ORDER BY ...])
Returns last value in ordered set.
```sql
SELECT last_value(amount ORDER BY created_at) FROM orders;
```

### nth_value(expression, n [ORDER BY ...])
Returns nth value in ordered set.
```sql
SELECT nth_value(amount, 2 ORDER BY amount DESC) FROM orders;
-- Returns second highest amount
```

## Grouping Functions

### grouping(column)
Returns 1 if column is aggregated in grouping sets, 0 otherwise.
```sql
SELECT 
    category,
    subcategory,
    grouping(category),
    grouping(subcategory),
    sum(amount)
FROM sales
GROUP BY ROLLUP(category, subcategory);
```

## Window Function Support

Most aggregate functions can be used as window functions:

```sql
-- Running total
SELECT 
    date,
    amount,
    sum(amount) OVER (ORDER BY date) as running_total
FROM orders;

-- Moving average
SELECT 
    date,
    amount,
    avg(amount) OVER (
        ORDER BY date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg
FROM orders;

-- Ranking within groups
SELECT 
    category,
    product,
    price,
    rank() OVER (PARTITION BY category ORDER BY price DESC) as price_rank
FROM products;
```

## Usage Examples

### Calculate summary statistics
```sql
SELECT 
    count(*) as total_orders,
    sum(amount) as total_revenue,
    avg(amount) as avg_order_value,
    min(amount) as min_order,
    max(amount) as max_order,
    stddev(amount) as stddev_order,
    percentile_cont(0.5) WITHIN GROUP (ORDER BY amount) as median_order
FROM orders;
```

### Group by with multiple aggregates
```sql
SELECT 
    category,
    count(*) as product_count,
    avg(price) as avg_price,
    min(price) as min_price,
    max(price) as max_price,
    array_agg(name ORDER BY price DESC LIMIT 3) as top_3_products
FROM products
GROUP BY category;
```

### Conditional aggregation
```sql
SELECT 
    count(*) FILTER (WHERE status = 'completed') as completed_orders,
    count(*) FILTER (WHERE status = 'pending') as pending_orders,
    sum(amount) FILTER (WHERE status = 'completed') as completed_revenue
FROM orders;
```

### Time-based aggregation
```sql
SELECT 
    date_trunc('hour', created_at) as hour,
    count(*) as order_count,
    avg(amount) as avg_amount,
    approx_percentile_cont(amount, ARRAY[0.5, 0.95, 0.99]) as percentiles
FROM orders
GROUP BY date_trunc('hour', created_at)
ORDER BY hour;
```