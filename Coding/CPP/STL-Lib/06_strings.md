# 06 — Strings & `string_view`

> `std::string` operations and the C++17 non-owning `std::string_view`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::string` basics

### Construction & assignment
```cpp
string a = "hello";
string b(5, 'x');                   // "xxxxx"
string c(a.begin() + 1, a.end());   // "ello"
string d = a + " world";            // "hello world"
a += "!";                           // "hello!"
```

### Comparison
```cpp
string x = "abc", y = "abd";
x == y;          // false
x < y;           // true  (lexicographic)
x.compare(y);    // < 0
```

### Indexing
```cpp
string s = "hello";
s[0] = 'H';                // "Hello"
s.front();  s.back();      // 'H', 'o'
s.at(10);                  // throws std::out_of_range
s.size();   s.length();    // both work
s.empty();
```

### Substring
```cpp
string s = "abcdef";
s.substr(2);        // "cdef"
s.substr(2, 3);     // "cde"   (pos, count)
```

### Find / rfind
```cpp
string s = "banana";
s.find('a');             // 1
s.find('a', 2);          // 3
s.rfind('a');            // 5
s.find("nan");           // 2
s.find("xyz");           // string::npos
if (s.find("xyz") == string::npos) cout << "not found";
```

### Insert / erase / replace
```cpp
string s = "hello";
s.insert(5, " world");           // "hello world"
s.erase(5, 6);                   // "hello"
s.replace(0, 5, "HI");           // "HI"
```

### Conversion
```cpp
int i      = stoi("42");
long long L= stoll("9999999999");
double d   = stod("3.14");
string s   = to_string(42);
string s2  = to_string(3.14);
```

### Iterate / sort / reverse characters
```cpp
string s = "hello";
for (char c : s) cout << c;
sort(s.begin(), s.end());        // "ehllo"
reverse(s.begin(), s.end());     // "ollhe"
```

### Useful character predicates from `<cctype>`
```cpp
isdigit('7');  isalpha('a');  islower('A');  isupper('Z');
tolower('A');                    // 'a'
transform(s.begin(), s.end(), s.begin(), ::tolower);
```

### Split / join (not built-in)
```cpp
vector<string> split(const string& s, char delim) {
    vector<string> out;
    stringstream ss(s);
    string token;
    while (getline(ss, token, delim)) out.push_back(token);
    return out;
}

string join(const vector<string>& parts, const string& sep) {
    string out;
    for (size_t i = 0; i < parts.size(); ++i) {
        if (i) out += sep;
        out += parts[i];
    }
    return out;
}
```

### Building strings — `stringstream`
```cpp
stringstream ss;
ss << "x=" << 42 << " y=" << 3.14;
string s = ss.str();   // "x=42 y=3.14"
```

---

## `std::string_view` (C++17)

A non-owning view of a contiguous character sequence. **Doesn't allocate**, **doesn't copy**.

### Construction
```cpp
string s = "hello world";
string_view sv1 = s;                 // view of full string
string_view sv2(s.c_str(), 5);       // "hello"
string_view sv3 = "literal";         // view of string literal
```

### Read-only API mirrors `string`
```cpp
string_view sv = "hello";
sv.size();
sv[0];                  // 'h'
sv.substr(1, 3);        // "ell"   — also a string_view, no allocation
sv.find('l');           // 2
sv == "hello";          // true
```

### Why it matters: zero-copy parameters
```cpp
// Before C++17 — common copy-and-conversion overhead
size_t count_l_old(const string& s) {
    return count(s.begin(), s.end(), 'l');
}

// C++17 — accepts string, char*, literal, substring, all without copying
size_t count_l(string_view sv) {
    return count(sv.begin(), sv.end(), 'l');
}

count_l("hello");                  // OK, no allocation
count_l(string("hello"));          // OK
count_l(string_view("hi").substr(0, 1));   // OK
```

### Lifetime gotchas
**The viewed data must outlive the view.**

```cpp
string_view bad() {
    string tmp = "oops";
    return tmp;            // tmp dies; returned view dangles. UB if used.
}

string_view also_bad = string("temp");   // temporary dies at end of expr — dangling

string s = "ok";
string_view ok = s;                       // fine, as long as s outlives ok
```

**Don't:** store `string_view` in a long-lived container if the source is a temporary.
**Don't:** assume it's null-terminated. `sv.data()` may not be `\0`-terminated.

---

## When to choose what

| Need | Use |
|---|---|
| Owns the chars | `string` |
| Read-only view, no copy | `string_view` (C++17) |
| Build a string from many parts | `stringstream` or repeated `+=`/`reserve` |
| C interop | `string::c_str()` |

## See also
- [09 Modifying Algorithms](09_algorithms_modifying.md) — algorithms on string ranges
- [19 C++17 Features](19_cpp17_features.md)
