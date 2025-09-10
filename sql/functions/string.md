---
sidebar_position: 2
---

# String Functions

Functions for text manipulation, pattern matching, and string operations.

## Basic String Operations

### length(string) / char_length(string) / character_length(string)
Returns the number of characters in a string.
```sql
SELECT length('hello') -- Returns: 5
SELECT char_length('hello') -- Returns: 5
SELECT character_length('hello') -- Returns: 5
```

### bit_length(string)
Returns the number of bits in a string.
```sql
SELECT bit_length('hello') -- Returns: 40
```

### octet_length(string)
Returns the number of bytes in a string.
```sql
SELECT octet_length('hello') -- Returns: 5
SELECT octet_length('你好') -- Returns: 6 (UTF-8 encoding)
```

### lower(string)
Converts string to lowercase.
```sql
SELECT lower('HELLO World') -- Returns: 'hello world'
```

### upper(string)
Converts string to uppercase.
```sql
SELECT upper('hello world') -- Returns: 'HELLO WORLD'
```

### initcap(string)
Capitalizes the first letter of each word.
```sql
SELECT initcap('hello world') -- Returns: 'Hello World'
```

## String Concatenation

### concat(string1, string2, ...)
Concatenates multiple strings.
```sql
SELECT concat('Hello', ' ', 'World') -- Returns: 'Hello World'
```

### concat_ws(separator, string1, string2, ...)
Concatenates strings with a separator.
```sql
SELECT concat_ws(', ', 'apple', 'banana', 'orange') -- Returns: 'apple, banana, orange'
```

### || operator
String concatenation operator.
```sql
SELECT 'Hello' || ' ' || 'World' -- Returns: 'Hello World'
```

## Substring Operations

### substr(string, start [, length]) / substring(string, start [, length])
Extracts a substring.
```sql
SELECT substr('Hello World', 7) -- Returns: 'World'
SELECT substr('Hello World', 7, 3) -- Returns: 'Wor'
SELECT substring('Hello World' FROM 7 FOR 3) -- Returns: 'Wor'
```

### left(string, n)
Returns the first n characters.
```sql
SELECT left('Hello World', 5) -- Returns: 'Hello'
```

### right(string, n)
Returns the last n characters.
```sql
SELECT right('Hello World', 5) -- Returns: 'World'
```

### split_part(string, delimiter, position)
Splits string and returns the specified part.
```sql
SELECT split_part('a,b,c', ',', 2) -- Returns: 'b'
```

## Trimming Functions

### trim(string) / btrim(string [, characters])
Removes leading and trailing spaces or specified characters.
```sql
SELECT trim('  hello  ') -- Returns: 'hello'
SELECT trim(BOTH 'x' FROM 'xxxhelloxxx') -- Returns: 'hello'
SELECT btrim('xxxhelloxxx', 'x') -- Returns: 'hello'
```

### ltrim(string [, characters])
Removes leading spaces or specified characters.
```sql
SELECT ltrim('  hello') -- Returns: 'hello'
SELECT ltrim('xxxhello', 'x') -- Returns: 'hello'
```

### rtrim(string [, characters])
Removes trailing spaces or specified characters.
```sql
SELECT rtrim('hello  ') -- Returns: 'hello'
SELECT rtrim('helloxxx', 'x') -- Returns: 'hello'
```

## Padding Functions

### lpad(string, length [, fill])
Pads string on the left to specified length.
```sql
SELECT lpad('hello', 10) -- Returns: '     hello'
SELECT lpad('hello', 10, '*') -- Returns: '*****hello'
```

### rpad(string, length [, fill])
Pads string on the right to specified length.
```sql
SELECT rpad('hello', 10) -- Returns: 'hello     '
SELECT rpad('hello', 10, '*') -- Returns: 'hello*****'
```

## Search and Replace

### position(substring IN string) / strpos(string, substring)
Returns the position of substring in string (1-based).
```sql
SELECT position('or' IN 'Hello World') -- Returns: 8
SELECT strpos('Hello World', 'or') -- Returns: 8
```

### starts_with(string, prefix)
Checks if string starts with prefix.
```sql
SELECT starts_with('Hello World', 'Hello') -- Returns: true
```

### ends_with(string, suffix)
Checks if string ends with suffix.
```sql
SELECT ends_with('Hello World', 'World') -- Returns: true
```

### contains(string, substring)
Checks if string contains substring.
```sql
SELECT contains('Hello World', 'Wor') -- Returns: true
```

### replace(string, from, to)
Replaces all occurrences of 'from' with 'to'.
```sql
SELECT replace('Hello World', 'World', 'Universe') -- Returns: 'Hello Universe'
```

### translate(string, from, to)
Replaces characters based on character mapping.
```sql
SELECT translate('12345', '134', 'abc') -- Returns: 'a2cb5'
```

### overlay(string PLACING replacement FROM start [FOR length])
Replaces a portion of string with replacement.
```sql
SELECT overlay('Hello World' PLACING 'Beautiful' FROM 7 FOR 5) 
-- Returns: 'Hello Beautiful'
```

## Pattern Matching

### like / ilike
Pattern matching with wildcards (% and _).
```sql
SELECT 'Hello World' LIKE 'Hello%' -- Returns: true
SELECT 'Hello World' ILIKE 'hello%' -- Returns: true (case-insensitive)
```

### similar to
SQL regular expression pattern matching.
```sql
SELECT 'abc' SIMILAR TO '(a|b)*' -- Returns: false
SELECT 'aaa' SIMILAR TO '(a|b)*' -- Returns: true
```

## String Transformation

### reverse(string)
Reverses the string.
```sql
SELECT reverse('hello') -- Returns: 'olleh'
```

### repeat(string, n)
Repeats string n times.
```sql
SELECT repeat('ab', 3) -- Returns: 'ababab'
```

### chr(n)
Returns character with ASCII code n.
```sql
SELECT chr(65) -- Returns: 'A'
```

### ascii(string)
Returns ASCII code of first character.
```sql
SELECT ascii('A') -- Returns: 65
```

## String Splitting and Array Functions

### string_to_array(string, delimiter [, null_string])
Splits string into array.
```sql
SELECT string_to_array('a,b,c', ',') -- Returns: ['a', 'b', 'c']
SELECT string_to_array('a,b,,c', ',', '') -- Returns: ['a', 'b', NULL, 'c']
```

### array_to_string(array, delimiter [, null_string])
Joins array elements into string.
```sql
SELECT array_to_string(ARRAY['a', 'b', 'c'], ',') -- Returns: 'a,b,c'
SELECT array_to_string(ARRAY['a', NULL, 'c'], ',', 'NULL') -- Returns: 'a,NULL,c'
```

## Encoding Functions

### encode(data, format)
Encodes binary data to text.
```sql
SELECT encode('hello'::bytea, 'hex') -- Returns: '68656c6c6f'
SELECT encode('hello'::bytea, 'base64') -- Returns: 'aGVsbG8='
```

### decode(string, format)
Decodes text to binary data.
```sql
SELECT decode('68656c6c6f', 'hex') -- Returns: 'hello'
SELECT decode('aGVsbG8=', 'base64') -- Returns: 'hello'
```

## Hash Functions

### md5(string)
Returns MD5 hash of string.
```sql
SELECT md5('hello') -- Returns: '5d41402abc4b2a76b9719d911017c592'
```

### sha224(string)
Returns SHA-224 hash of string.
```sql
SELECT sha224('hello') -- Returns: 'ea09ae9cc6768c50fcee903ed054556e5bfc8347907f12598aa24193'
```

### sha256(string)
Returns SHA-256 hash of string.
```sql
SELECT sha256('hello') -- Returns: '2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824'
```

### sha384(string)
Returns SHA-384 hash of string.
```sql
SELECT sha384('hello')
```

### sha512(string)
Returns SHA-512 hash of string.
```sql
SELECT sha512('hello')
```

## UUID Functions

### uuid()
Generates a random UUID v4.
```sql
SELECT uuid() -- Returns: '550e8400-e29b-41d4-a716-446655440000'
```

## Usage Examples

### Clean and standardize names
```sql
SELECT 
    initcap(trim(lower(name))) as standardized_name
FROM users;
```

### Extract domain from email
```sql
SELECT 
    split_part(email, '@', 2) as domain
FROM users;
```

### Mask sensitive data
```sql
SELECT 
    left(credit_card, 4) || repeat('*', 8) || right(credit_card, 4) as masked_card
FROM payments;
```

### Parse structured strings
```sql
SELECT 
    split_part(log_entry, ':', 1) as timestamp,
    split_part(log_entry, ':', 2) as level,
    split_part(log_entry, ':', 3) as message
FROM logs;
```