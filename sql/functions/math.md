---
sidebar_position: 1
---

# Math Functions

Mathematical and arithmetic functions for numeric operations.

## Basic Arithmetic

### abs(x)
Returns the absolute value of a number.
```sql
SELECT abs(-15.7) -- Returns: 15.7
```

### ceil(x) / ceiling(x)
Returns the smallest integer greater than or equal to x.
```sql
SELECT ceil(4.3) -- Returns: 5
SELECT ceiling(4.3) -- Returns: 5
```

### floor(x)
Returns the largest integer less than or equal to x.
```sql
SELECT floor(4.7) -- Returns: 4
```

### round(x [, d])
Rounds x to d decimal places (default 0).
```sql
SELECT round(3.14159, 2) -- Returns: 3.14
SELECT round(3.5) -- Returns: 4
```

### trunc(x [, d])
Truncates x to d decimal places (default 0).
```sql
SELECT trunc(3.14159, 2) -- Returns: 3.14
SELECT trunc(3.9) -- Returns: 3
```

## Power and Root Functions

### sqrt(x)
Returns the square root of x.
```sql
SELECT sqrt(16) -- Returns: 4
```

### cbrt(x)
Returns the cube root of x.
```sql
SELECT cbrt(27) -- Returns: 3
```

### power(x, y) / pow(x, y)
Returns x raised to the power of y.
```sql
SELECT power(2, 3) -- Returns: 8
SELECT pow(2, 3) -- Returns: 8
```

### exp(x)
Returns e raised to the power of x.
```sql
SELECT exp(1) -- Returns: 2.718281828...
```

## Logarithmic Functions

### ln(x)
Returns the natural logarithm of x.
```sql
SELECT ln(2.718281828) -- Returns: 1
```

### log(x) / log10(x)
Returns the base-10 logarithm of x.
```sql
SELECT log(100) -- Returns: 2
SELECT log10(100) -- Returns: 2
```

### log2(x)
Returns the base-2 logarithm of x.
```sql
SELECT log2(8) -- Returns: 3
```

## Trigonometric Functions

### sin(x)
Returns the sine of x (x in radians).
```sql
SELECT sin(pi()/2) -- Returns: 1
```

### cos(x)
Returns the cosine of x (x in radians).
```sql
SELECT cos(0) -- Returns: 1
```

### tan(x)
Returns the tangent of x (x in radians).
```sql
SELECT tan(pi()/4) -- Returns: 1
```

### asin(x)
Returns the arcsine of x in radians.
```sql
SELECT asin(1) -- Returns: 1.5707963... (π/2)
```

### acos(x)
Returns the arccosine of x in radians.
```sql
SELECT acos(0) -- Returns: 1.5707963... (π/2)
```

### atan(x)
Returns the arctangent of x in radians.
```sql
SELECT atan(1) -- Returns: 0.7853981... (π/4)
```

### atan2(y, x)
Returns the arctangent of y/x in radians.
```sql
SELECT atan2(1, 1) -- Returns: 0.7853981... (π/4)
```

### sinh(x)
Returns the hyperbolic sine of x.
```sql
SELECT sinh(0) -- Returns: 0
```

### cosh(x)
Returns the hyperbolic cosine of x.
```sql
SELECT cosh(0) -- Returns: 1
```

### tanh(x)
Returns the hyperbolic tangent of x.
```sql
SELECT tanh(0) -- Returns: 0
```

### asinh(x)
Returns the inverse hyperbolic sine of x.
```sql
SELECT asinh(0) -- Returns: 0
```

### acosh(x)
Returns the inverse hyperbolic cosine of x.
```sql
SELECT acosh(1) -- Returns: 0
```

### atanh(x)
Returns the inverse hyperbolic tangent of x.
```sql
SELECT atanh(0) -- Returns: 0
```

## Conversion Functions

### radians(x)
Converts degrees to radians.
```sql
SELECT radians(180) -- Returns: 3.14159... (π)
```

### degrees(x)
Converts radians to degrees.
```sql
SELECT degrees(pi()) -- Returns: 180
```

## Sign and Comparison

### sign(x) / signum(x)
Returns the sign of x (-1, 0, or 1).
```sql
SELECT sign(-5) -- Returns: -1
SELECT sign(0) -- Returns: 0
SELECT sign(5) -- Returns: 1
```

### greatest(x1, x2, ...)
Returns the greatest value from a list.
```sql
SELECT greatest(2, 5, 1, 9, 3) -- Returns: 9
```

### least(x1, x2, ...)
Returns the smallest value from a list.
```sql
SELECT least(2, 5, 1, 9, 3) -- Returns: 1
```

## Random Functions

### random()
Returns a random float between 0 and 1.
```sql
SELECT random() -- Returns: 0.8473... (random value)
```

## Special Values

### pi()
Returns the value of π.
```sql
SELECT pi() -- Returns: 3.14159265...
```

### e()
Returns the value of e (Euler's number).
```sql
SELECT e() -- Returns: 2.71828182...
```

### infinity() / '-infinity'::float
Returns positive or negative infinity.
```sql
SELECT infinity() -- Returns: Infinity
SELECT '-infinity'::float -- Returns: -Infinity
```

### isnan(x)
Checks if x is NaN (Not a Number).
```sql
SELECT isnan('NaN'::float) -- Returns: true
```

### isinf(x)
Checks if x is infinite.
```sql
SELECT isinf(infinity()) -- Returns: true
```

### isfinite(x)
Checks if x is finite (not infinite or NaN).
```sql
SELECT isfinite(3.14) -- Returns: true
```

## Numeric Operations

### factorial(n)
Returns the factorial of n (n!).
```sql
SELECT factorial(5) -- Returns: 120 (5*4*3*2*1)
```

### gcd(a, b)
Returns the greatest common divisor of a and b.
```sql
SELECT gcd(48, 18) -- Returns: 6
```

### lcm(a, b)
Returns the least common multiple of a and b.
```sql
SELECT lcm(4, 6) -- Returns: 12
```

## Usage Examples

### Calculate compound interest
```sql
SELECT 
    principal * power(1 + rate/100, years) as final_amount
FROM investments;
```

### Calculate distance between points
```sql
SELECT 
    sqrt(power(x2 - x1, 2) + power(y2 - y1, 2)) as distance
FROM points;
```

### Normalize values to 0-1 range
```sql
SELECT 
    (value - min_value) / (max_value - min_value) as normalized
FROM (
    SELECT 
        value,
        MIN(value) OVER () as min_value,
        MAX(value) OVER () as max_value
    FROM measurements
);
```