---
marp: true
theme: gaia
class: invert
paginate: true
---

<style>
.centered-image {
  text-align: center;
}

/* Global code block styling */
pre {
  font-size: 0.8em; /* Adjust as needed: 0.7em, 0.75em, 0.9em */
}

code {
  font-size: 0.85em; /* For inline code */
}
</style>

# German strings

A case for yet another string type

<style scoped>
.author {
  position: absolute;
  bottom: 20px;
  left: 25px;
  font-size: 0.8em;
  opacity: 0.8;
  text-align: left;
}
</style>

<div class="author">
Dmytro Shynkar<br>
dmytroshynkar@gmail.com <br>
StockholmCpp 0x38
</div>

---

## Plan

- Intro and performance promises
- Implementation details
- Test on an almost real application
- Name context

<!-- 
note: If you only want to know what this string type has to do with Germany you'll have to wait till the end ;)
-->

---

## We love strings...

... and we have some already!

* `<const> char*` - a C string, a NULL-terminated char array
* `std::basic_string<>` - owning mutable string, convertible to a C string, usually has SSO and takes up 32 bytes
* `std::basic_string_view<>` - a view into some string data, may not be null terminated
* Your favorite custom implementation

---

## We love strings...

### So why would we need any more?

* 2x smaller memory footprint for the string object
* 30-50% faster search and sorting operations

TODO: A chart showing sorting depending on array size and std::count if for the same measure

<!-- 
note: Based on totally trust-worthy benchmarks!
-->

--- 

## Anatomy of `std::string`
```
class string
{
    size_t size;
    size_t capacity;
    char* data;
};
```

<div class="centered-image">

![w:600](assets/std_string_simple.svg)

</div>

--- 

## Anatomy of `std::string`

A bit closer to the truth

<style scoped>
.three-columns {
  display: flex;
  justify-content: space-between;
  gap: 30px;
  margin-top: 50px;
}

.column {
  flex: 1;
  text-align: left;
}
</style>

<div class="three-columns">
<div class="column">

```cpp
// gcc
struct string
{
    char* ptr;
    size_t size;
    union {
        size_t capacity;
        char buf[16];
    };
};
```

</div>
<div class="column">

```cpp
// msvc
struct string
{
    union {
        char* ptr;
        char buf[16];
    };
    size_t size;
    size_t capacity;
}
```

</div>
<div class="column">

```cpp
// clang
union string
{
    struct {
        size_t capacity;
        size_t size;
        char* ptr;
    } large;

    struct {
        unsigned char is_large:1;
        unsigned char size:7;
        char buf[sizeof(large) - 1];
    } small;
}
```

</div>
</div> 
<!-- 
note: Courtesy of raymond chen!
-->

---

## Anatomy of `std::string`

A common thing - SSO and a 32 byte size


<div class="centered-image">

![w:800](assets//std_string_union.svg)

</div>

Pictured here: `libstdc++`(GCC)

---

## Anatomy of `std::string`

- Always owning the memory
- Mutable
- Local for 15 to 23 byte-long strings
- Null-terminated

* `std::string_view` for non-owning strings
* Just a pointer and a size, 16 bytes

<!-- 
note: While all of the 3 big implementations make different tradeoffs while staying in the bound of std::string interface they share a lot in common
-->

---

## What can we do differently?

If we are not bound by the standard and consider some factoids...

* Search and sort operations usually dominate
* It's common to look at a small chunk of the string
* Oftentimes a batch of strings all share a common lifetime
* Most strings are small
* `std::string` *is usually good at the last one*

<!-- 
note: Memory-mapped files, request scope, etc
-->

---

## What can we do differently?

## **Everything!**

* *Sometimes* own the data
* Immutable
* 12 bytes for small strings
* No null termination
* Always have a 4-byte prefix locally

<!-- 
note: We will go over each part now
-->

---

## What can we do differently?

<div class="centered-image">

![w:800](assets/german_string.svg)

</div>


---

## Sometimes owning the data - string classes

Why have separate `string` and `string_view` types at all?

By tagging the pointer with 2 bits of data we can assign each string to a *class*

- `temporary` - usual owning string
- `persistent` - always valid string
- `transient` - caller defined lifetime


---

## Sometimes owning the data - string classes

### Pros

- Avoids unnecessary copies
- Clear runtime values for lifetimes

### Cons

- A new footgun
- Relies on the canonical pointer form and unused bits

---

## Prefix

- Possibly avoid the cache miss on comparison
- Neatly becomes the prefix for the SSO string

---

## Show me the code!

```cpp
struct german_string
{
    std::uint32_t size;
    union {
        char small[12];
        struct {
            std::uint32_t prefix;
            char* data;
        } large;
    };
};
```

* Clean translation of the diagram
* **Does not work!**

---

## Show me the code!

```cpp
struct german_string
{
    std::uint64_t state[2];
};
```

<div class="centered-image">

![w:600](assets/german_array.svg)

</div>

<!-- 
note: Rely on little endian
-->


---

## Show me the actual code!

Equality
```cpp
bool equals(const german_string& other) const
{
    if (_state[0] != other._state[0])
    {
        return false;
    }

    // If we are small, it's guaranteed that that the other one is as well 
    if (static_cast<std::uint32_t>(_state[0]) <= 12)
    {
        return _state[1] == other._state[1];
    }
    const char* my_ptr = reinterpret_cast<const char*>(_state[1] & ~PTR_TAG_MASK);
    const char* other_ptr = reinterpret_cast<const char*>(other._state[1] & ~PTR_TAG_MASK);
    return std::memcmp(my_ptr, other_ptr, _get_size()) == 0;
}
```

---

## Show me the actual code!

Compare
```cpp
int compare(const german_string& other) const
{
    const uint32_t min_size = std::min(_get_size(), other._get_size());
    const uint32_t min_or_prefix_size = std::min(min_size, sizeof(uint32_t));
    const int prefix_cmp = prefix_memcmp(_get_prefix(), other._get_prefix(), min_or_prefix_size);
    if (min_or_prefix_size == min_size || prefix_cmp != 0)
    {
        return prefix_cmp != 0 ? prefix_cmp : (_get_size() - other._get_size());
    }
    const int result = std::memcmp(
        _get_maybe_small_ptr() + 4, 
        other._get_maybe_small_ptr() + 4, 
        min_size - min_or_prefix_size
    );
    return result != 0 ? result : (_get_size() - other._get_size());
}
```

---

## Show me the actual code!

Bit magic!
```cpp
int prefix_memcmp(std::uint32_t a, std::uint32_t b, int n)
{
    std::uint32_t diff = a ^ b;
    std::uint32_t mask = (std::uint32_t)((0xFFFFFFFFull << (n * 8)) >> 32);
    diff &= mask;

    int first_diff = std::countr_zero(diff) / 8;
    std::uint8_t byte_a = ((std::uint64_t)a >> (first_diff * 8)) & 0xFF;
    std::uint8_t byte_b = ((std::uint64_t)b >> (first_diff * 8)) & 0xFF;
    return (int)byte_a - (int)byte_b;
}
```

<!-- 
note: This is approximately 2x faster than std::memcmp for 4 byte values
-->


---

## Does it actually work? - 1BRC

<div class="centered-image">

![w:500](assets/1brc.png)

<!-- 
note: Very fresh a definitely not biased. Aggregating a large volume of data from a CSV
-->

</div>

---

## Does it actually work? - 1BRC

- Find a simple open source C++ implementation
- Replace all `std::string` with `gs::german_string`

TODO: Do this and report result here

<!-- 
note: Maybe a demo
-->

---

## Where can I get some?

Most high performance OLAP systems have them!

- DuckDB
- Apache Arrow & DataFusion
- Polars
- Velox

Implementation written for this talk: https://github.com/dmitr101/german_strings

---

## Why German though?

<div class="centered-image">

![w:800](assets/cmu_adb_screenshot.png)

</div>

<!-- 
note: Because of this guy
-->

---

## Sources and further info
If you want to learn more about German strings, check out these resources:
- https://cedardb.com/blog/german_strings/
- https://cedardb.com/blog/strings_deep_dive/
- https://datafusion.apache.org/blog/2024/09/13/string-view-german-style-strings-part-1/
- https://datafusion.apache.org/blog/2024/09/13/string-view-german-style-strings-part-2/
- https://www.tunglevo.com/note/an-optimization-thats-impossible-in-rust/
- https://the-mikedavis.github.io/posts/german-string-optimizations-in-spellbook/

---

# Thank you for your attention

## Questions?