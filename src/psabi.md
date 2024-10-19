# psABI

## Types

### C Primitive Sizes

`CHAR_BIT` is 8.

| Type         | Size    |
| ------------ | ------- |
| `bool`[^1]   | 1       |
| `short`      | 2       |
| `int`        | 4       |
| `long`       | 4       |
| `long long`  | 8       |
| `float`      | 4       |
| `double`     | 8       |
| `long double`| 8       |
| `void*`      | 4       |
| `intptr_t`   | 4       |
| `size_t`     | 4       |
| `intmax_t`   | 8       |
| `wchar_t`    | 2       |

[^1]: Referred to as `_Bool` in C since C99 until C23.

### Char Types

`char` is unsigned by default.

### Primitive Alignment

The Size and alignment of `align_max_t` are both 4. Each primitive less than or equal to 4 bytes in size is aligned to its size, rounded up to the next power of two bytes. 
Each primitive that is greater than 4 bytes in size are aligned to 4 bytes. This includes `_BitInt(N)` types.

### Floating Point Formats

!{#copyright}