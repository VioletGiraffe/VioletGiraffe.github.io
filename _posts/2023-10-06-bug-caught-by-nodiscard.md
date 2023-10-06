## This curious typo could be a serious bug

Here's an interesting bug I found in my old code. It's a small typo that completely nukes an `if` switch, and the best part is that it is undetected by:
* MSVC with `/W4`
* clang with `-Wall -Wextra`
* GCC with `-Wall -Wextra`

---

Here it is:

```
struct FancyString {
    enum CaseSensitivity {CaseInsensitive, CaseSensitive};
    bool contains(const char* what, CaseSensitivity = CaseSensitive);
};

void bug(const FancyString& log)
{
    if (log.contains("exception"), FancyString::CaseInsensitive)
        throw std::exception();
}
```

Do you see it?    
The only reason I noticed is the new major release of the `FancyString` library added [[nodiscard]] where approapriate. That, of course, did finally catch this error, which was lurking in our codebase for many years.

Godbolt: [https://godbolt.org/z/zYsb36vf9](https://godbolt.org/z/zYsb36vf9)

Conclusions:
* `[[nodiscard]]` is great, don't be lazy and always put it where appropriate.
* Compilers need more static analysis and warnings for suspicioius code constructs.
