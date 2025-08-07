---
marp: true
theme: uncover
class: invert
paginate: true
---

# German strings

eine deutsche string - a case for yet another string type

---

# Who am I

---

# One more string type?

Sneak peak into benchmarks

--- 

# String type bestiary / classification

- Regular old vector<char>
- COWs
- SSO strings
- Interned strings?
- C-strings
- string_views???

https://github.com/rosetta-rs/string-rosetta-rs?tab=readme-ov-file

---

# Which type is this one?

---

# Implementation details

Some tradeoffs

---

# What are string classes

---

# A bit of code examples

- Compare string
- Copy
- etc

---

# A spinoff - don't try to beat std::memcmp

---

# A spinoff - representation and UB - array vs union

---

# Complain a bit about std interface?

---

# Comparing to the big 3

---

# Notes on benchmarking

---

# Gotchas and where not to use

---

# It's available here

Github link, whatever

---

# Why is it german though?

Andy Pavlo, Umbra, Munich, CedarDB

---

# Sources and further info

https://cedardb.com/blog/german_strings/
https://cedardb.com/blog/strings_deep_dive/
https://datafusion.apache.org/blog/2024/09/13/string-view-german-style-strings-part-1/
https://datafusion.apache.org/blog/2024/09/13/string-view-german-style-strings-part-2/
https://www.tunglevo.com/note/an-optimization-thats-impossible-in-rust/
https://the-mikedavis.github.io/posts/german-string-optimizations-in-spellbook/

---

# Thank you for your attention