<!-- .slide: data-background="#00000f" -->
# C++20 and C++23 for fun and profit

```c++ [1,5]
import std;

int main()
{
    std::println("Hello World");
}
```

https://www.modernescpp.com/index.php/c23-a-modularized-standard-library-stdprint-and-stdprintln/

<!-- .element: class="fragment" -->

---
# Table of sections
* C++20 Big Four
  * [Modules](#/3)
  * [Coroutines](#/4)
  * [Concepts](#/5)
  * [Ranges](#/6)
* [C++20 Core Language](#/7)
* [C++20 Library improvements](#/18)
* [C++23 Core Language](#/31)
* [Reference](#/41)
---

# C++20 The Big Four
## concepts
## ranges
## coroutines
## modules

---
## Modules - C++20 Big Four
* modules are similar to precompiled headers
* no macro leakage
* any order of inclusion

C++ 23 provides STL as a module.
* entire STL: `import std;`
* STL + stdlib (printf): `import std.compat;`
```cpp
import std.compat;

int main()
{
    printf("Import std.compat to get global names like printf()\n");

    std::vector<int> v{5, 5, 5};
    for (const auto& e : v)
    {
        printf("%i", e);
    }
}
```

--

## Modules - C++20 Big Four
* vendors ABI are incompatible
* better support in MSVC
* sharing/distribution model still not clear
* recommendation: wait, experiment

For a detailed overview of the current state of modules see presentation "C++20 Modules - Review of the Current State of C++ Modules 2024 - Luis Caro Campos - C++ on Sea 2024" <!-- .element: class="fragment" -->
https://www.youtube.com/watch?v=flu-f6SDnOE&ab_channel=cpponsea

---

## Coroutines - C++20 Big Four

* coroutines allow function to save state between calls and yield partial results
* in some cases (event based design) they streamline spaghetti code into simpler and prettier code. But this require writing special boilerplate.
* language support is there but library support only providing `std::generator` in C++23
* recommendation: use `std::generator` or wait

Raymond Chen "A map through the three major coroutine series" https://devblogs.microsoft.com/oldnewthing/20210504-01/?p=105178

--

## Coroutines - C++20 Big Four
```cpp
#include <generator>
#include <print>
#include <ranges>

std::generator<int> iota()
{
  for (int i = 0; /* Never stop */ ; ++i)
    co_yield i;
}

for (auto i : iota() | std::views::take(6))
{
  std::print("{} ", i);
}
// prints "0 1 2 3 4 5"
```
https://godbolt.org/z/K6ErTvW6e

"Introduction to C++ Coroutines and gRPC and How to Write Coroutine Support Code - Jonathan Storey - C++ on Sea 2024" <!-- .element: class="fragment" -->
https://www.youtube.com/watch?v=deUbQodyaC4&ab_channel=cpponsea

---

## Concepts - C++20 Big Four

* Improve and shorten template errors
* Impose interface on template
```cpp
#include <concepts>

auto gcd(std::signed_integral auto a, std::signed_integral auto b)
{
  if(b == 0) return a;
  else return gcd(b, a % b);
}
```

```
Error for calling with inapropriate type:
  error C2672: 'gcd': no matching overloaded function found.
  Note: the concept `std::signed_integral<double>` evaluated to false.

```
<!-- .element: class="fragment" -->
https://godbolt.org/z/Whn7r5K1o

--
## Concepts - C++20 Big Four
```cpp [4-10|12-13|17-22]
#include <iostream>
#include <vector>

// new concept definition, keyword requires
template<typename T>
concept Printable = requires(std::ostream& os, const T& msg)
{
    {os << msg};
};

// restrict myprint interface to only printables only
template <Printable T>
void myprint(const T& msg){
    std::cout << msg;
}

// example of a more complex concept
template<typename T>
concept has_string_data_member = requires(T v) {
    { v.m_name } -> std::convertible_to<std::string>;
    v.clone();
};
```

---

## Ranges - C++20/C++23 Big Four
* **Range** is an object referring to a sequence/range of elements
  * any container supporting begin/end is a range
  * protection agains misplaced iterators
* **range algorithms** provides nicer synthax
  * protection agains dangling iterators
```cpp
vector data { 11, 22, 33 };
sort(begin(data), end(data));
ranges::sort(data);
```
* **Views** transform/filter range: lazily evaluated, non-owning, non-mutating, cheap
* **Projection** transform elements before handing over to algorithm
* **Pipelining** - views can be chained with `|`
* Introduced with C++20, improved with C++23 for example `ranges::to` and `join_with`

--

## Ranges - C++20/C++23 Big Four
```cpp
#include <vector>
#include <ranges>
#include <iostream>

int main()
{
  std::vector<int> ints{0, 1, 2, 3, 4, 5};
  auto even = [](int i){ return 0 == i % 2; };
  auto square = [](int i) { return i * i; };

  for (int i : ints
    | std::views::filter(even)
    | std::views::transform(square))
  {
    std::cout << i << ' ';
  }
}
// 0 4 16
```
https://godbolt.org/z/MM68Khf8f

--

## Ranges - C++20/C++23 Big Four
* Implementing Python-like range
```cpp
#include <ranges>
#include <vector>
#include <print>
using namespace std;

vector<int> range(int begin, int end, int stepsize = 1)
{
  auto boundary = [end](int i){ return i < end; };
  return ranges::views::iota(begin)
    | views::stride(stepsize)
    | views::take_while(boundary)
    | ranges::to<vector>();
}

int main()
{
    for (auto i: range(1, 10, 2))
      std::print("{} ", i);
} // 1 3 5 7 9
```
https://godbolt.org/z/fsf5fWr5M

https://www.modernescpp.com/index.php/tag/ranges/

--

## Ranges - C++20/C++23 Big Four
* View requirement that copy, move and assignment of a view be in O(1).
  * Developer of the view marks it as view by trait `std::ranges::enable_view`.
  * Views typically refer (`span`, `ref_view`) or even modify (`transform_view`) elements in another range or generates new elements (`iota`).
* Range addaptor object is very similar to a view but syntax making them easier to chain through `|`
  * most views (`std::ranges::transform_view`) having corresponding adapter (`std::views::transform`).
  * views taking first argument as argument, while adapters taking first argument through `|`
  * examples from `std::views::` - `filter,transform,take,take_while,drop,drop_while,join,split,reverse`

---

# C++20 Core Language

* `operator <=>`
* { .value = "Hello" }
* init in `for`
* `consteval`
* `constexpr` changes
* `using enum`
* `[[likely]]`
* `[[assume]]` C++23
* `[[nodiscard("")]]`
* `match<"[a-z]">(str)`
* lambda changes

---

## The three-way Comparison operator <=>
```cpp
#include <compare>
struct MyInt {
  int i;
  bool b;
  double ad[2][2];
  auto operator<=>(const MyInt&) const = default;
};
```
* default `operator <=>` generates all six comparison operators such as `==, !=, <, <=, >, and >=`.
* similar to `strcmp()` or `string.compare()` but returns `strong_ordering`, `partial_ordering`, `weak_ordering`

```cpp
void sort_floats_nans(vector<float>&vec)
{
  sort(vec.begin(), vec.end(), [](float a, float b)
    {
      return is_lt(strong_order(a,b));
    });
}
```
<!-- .element: class="fragment" -->

---

## Designated initializers

```cpp
struct Data
{
  int key{};
  std::string value;
  bool option{};
};

Data d { .value = "Hello" };
```

---

## Initializers in ranged for

```cpp
// c++17
auto data = GetContainer();
for (auto& value: data.values)
{ /* ... */ }

// c++20
for (auto data = GetContainer(); auto& value: data.values)
{ /* ... */ }
```

---

## `consteval` and `constinit`
* `consteval` is strong version of `constexpr` producing immediate function. Compiler ensures every call to the function must produce a compile-time constant expression.
```cpp
consteval int sqr(int n) { return n*n; }
int r = sqr(100);  // OK
int x = 100;
int r2 = sqr(x); // Error
```

https://godbolt.org/z/8TnnzPMb9

* `if consteval { A } else { B }` C++23
  * no parameters
  * section A guaranted evaluated at compile time and allow usage of `consteval` functions

* `constinit` ensures that the variable with static storage duration is initialized at compile-time. https://www.modernescpp.com/index.php/c-20-static-initialization-order-fiasco/

---

## constexpr changes
* constexpr virtual functions
* constexpr function now may use
  * new/delete
  * dynamic_cast and typeid
  * try/catch

as a result `std::string` and `std::vector` now constexpr

---

## using enum
C++17
```cpp
switch (cardTypeSuit)
{
  case CardTypeSuit::Clubs: return "Clubs";
  case CardTypeSuit::Diamonds: return "Diamonds";
  case CardTypeSuit::Hearts: return "Hearts";
  case CardTypeSuit::Spades: return "Spades";
}
```
C++20
```cpp
switch (cardTypeSuit)
{
  using enum CardTypeSuit;
  case Clubs: return "Clubs";
  case Diamonds: return "Diamonds";
  case Hearts: return "Hearts";
  case Spades: return "Spades";
}
```

---

## `[[likely]]`, `[[unlikely]]`, `std::unreachable`, `[[assume]]`

New C++20 attributes `[[likely]]` and `[[unlikely]]` and C++23 `std::unreachable` are for hardcore assembly tuning audience. May hurt.

Likely/unlikely hints compiler which code path is preferable to make faster.

```cpp
double pow(double x, long long n) noexcept
{
    if (n > 0) [[likely]]
        return x * pow(x, n - 1);
    else [[unlikely]]
        return 1;
}
```

--

## `[[likely]]`, `[[unlikely]]`, `std::unreachable`, `[[assume]]`

C++23 `std::unreachable` invokes undefined behaviour, marking code as unreachable, which in special case can improve generated assembly. `[[assume(expression)]]` invokes undefined behaviour if expression is false.
<!-- .element: class="fragment" -->
```cpp
void do_0_1_2_3(int k) {
   switch (k) {
      case 0: case 2:
         handle_0_or_2(); break;
      case 1:
         handle_1();      break;
      case 3:
         handle_3();      break;
      default:
         std::unreachable();
   }
}
```
<!-- .element: class="fragment" -->

---

## custom message in the `[[nodiscard]]`

```cpp
[[nodiscard("Ignoring the return value will result in memory leaks.")]]
void* GetData()
{
  /* ... */
}
```

---

## Non-type template parameters
* in C++17 you may use integral type as non-type template parameter
* in C++20 more types allowed, such as floats and classes (with restrictions).

As a notable example of usage, Compile Time Regular Expression (CTRE) library:

```cpp
auto m { ctre::match<"[a-z]+([0-9]+)">(str) };
```
https://www.youtube.com/watch?v=8dKWdJzPwHw&ab_channel=CppCon

---

## Lambda changes
* Since C++20 if `this` needed in lambda, it has to be captured explicitly.
```cpp
auto cpp17 = [=]{return this->m_member;}
auto cpp20 = [=, this]{return this->m_member;}
```
* Templated lambda expressions
```cpp
[]<typename T>(const vector<T>& x){...}
```
https://godbolt.org/z/fW5Gexbq9

* C++23 - added attributes on function call (ex.`nodiscard`), before attributes allowed on function object (ex. `deprecated` ):
```cpp
auto a = [] [[nodiscard]] () [[deprecated]] { return 42; };

```

---

# C++20 Library improvements

* std::`format`
* *C++23* std::`print`
* std::`span`
* *C++23* std::`spanstream`,`mdspan`,`arr[x,y,z]`
* std::`source_location`
* std::`atomic<shared_ptr>`
* std::`jthread`
* std::`osyncstream`
* std::`atomic{...}.wait()`
* `<chrono>` timezones and changes
* `<bit>` operations
* `starts_with`, `contains`, `erase_if`
* `shift_left()`, `midpoint()`, `execution::unseq`
* std::`numbers`

--

## C++20 std::format
`<format>` is based on **fmt** library https://github.com/fmtlib/fmt and provides modern Python-like syntax for writing to output
* very fast (slightly outperforms `printf`)
* type safe
* easier to use than boost::format
```cpp
// Hello world!
std::cout << std::format("Hello {}!\n", "world");
// Read 1234 bytes from file1.txt
std::cout << std::format("Read {1} bytes from {0}", "file1.txt", 1234);
// test centering, padding, setting width
assert(std::format("{:*^6}", 'x') == "**x***");
```

--

## C++23 std::format improvements
* `std::print` is `std::cout` << `std::format`
* `std::println` is `std::cout` << `std::format` << `'\n'`
* containers and ranges can be printed, as well as `<chrono>` objects

```cpp
std::vector<std::vector<int>> vv = {{1, 2, 3},{11, 22, 33}};
std::println("{}", vv);
// [[1, 2, 3], [4, 5, 6]]
```

https://www.godbolt.org/z/WMb9MfqWK

Additionally `std::print` better works with unicode characters (MSVC requires /utf-8 compilation flag).

---

## std::span
```cpp
span<int> d { ptr, len };
```
Similar to `string_view`:
* Refers to some contiguous data
* Very cheap to copy, recommended to pass by value
* Does not own the data
* Never allocates/deallocates
* Supports begin, cbegin,..., front, back, operator[],data,empty,size

Special to `span`:
* Could be read/write
  * read-only span defined as `span<const int>`, not `const span<int>`
* Size could be dynamic (run-time) or static (compile-time)
* Supports size_bytes
* Sub-spans: subspan(offset,count), first(count), last(count)

  https://godbolt.org/z/rKabzb17j /
  https://godbolt.org/z/4rdqY3j6x


--

## C++23 std::spanstream
Adapter to put std::span into stream.
```cpp
char data[] = "11 22";
int a{}, b{};
std::ispanstream { std::span<char>{data} } >> a >> b;

char buf[32] {};
std::ispanstream { std::span<char>{buf} } << b << a;
```

## C++23 `arr[x,y,z]` C++23 `std::mdspan`
New multidimensional subscript operator: `data[x, y, z]`
```cpp
T& operator[](size_t x, size_t y, size_t z) noexcept { /*...*/ }
```
Multidimensional array view, extention of `std::span`:
```cpp
int* data { /* raw data */ };
// View data as contiguous memory representing cube 3x3x3
auto mySpan { std::mdspan(data, 3, 3, 3) };
mySpan[1, 1, 1] = 42;
```
Subset (slice) of mdspan - `std::submdspan`

---

## `std::source_location`

```cpp
#include <iostream>
#include <string_view>
#include <source_location>

void log(std::string_view message, const std::source_location& location = std::source_location::current())
{
std::cout << location.file_name() << ":" << location.line() << " " << message << '\n';
}

int main()
{
    log("Hello world!");
}
// example.cpp:12 Hello world!
```
https://godbolt.org/z/xWs5x6ebs

---

## `atomic<shared_ptr>`
`shared_ptr` is hard to use in multithreaded context as it requires understanding of how to properly use *global non-member functions* such as `atomic_load` + `atomic_comprare_exchange_weak`.

now it is more streamlined with new `atomic_shared_ptr` where all this are methods in class and cannot be forgotten by accident

---

## Joinable and Cancellable Threads
* `std::jthread`
  * supports cooperative cancellation
  * destructor automatically asks thread to cancel and calls `join()`
* `std::stop_token`
  * to check if stop requested
  * compatible with `condition_variable_any`
* `std::stop_source`
  * to request stop
* `std::stop_callback`
  * callback may be registered to be called if stop issued

--

## Joinable and Cancellable Threads

```cpp
std::jthread job { [](std::stop_token token) {
  while (!token.stop_requested()) {
    //...
  }
} };

job.request_stop();

bool b = job.get_stop_token().stop_requested();
```

---

## `std::osyncstream` ##
* Synchronized output streams std::basic_osyncstream allow threads to write without interleaving on the same output stream.
* Output pieces written to internal `std::basic_syncbuf` until osyncstream destroyed, then all accumulated data written to stream atomically.
```cpp
std::osyncstream(std::cout) << "Hello, " << "World" << std::endl << "!..";
```

---

## Synchronization
* `<semaphore>`
  * Lightweight synchronization primitive, can be used to implement any other concept such as mutex, latch, barrier.
  * counting semaphore: non-negative resource count
  * binary semaphore: 0 or 1 resource count
* `<latch>`
  * threads block at latch point until set number of threads reached, then all allowed to continue
  * std::latch instance is single-use
* `<barrier>`
  * like a latch but may be used multiple times

---

## Synchronization on `std::atomic`
* Wait/block for an atomic object to change its value, notified by a notification function
* Can be more performant than polling
* Methods
  * `wait()`
  * `notify_one()`
  * `notify_all()`

Also see `std::atomic_ref`

---

## `<chrono>` Dates
Date+Time and Date:
```cpp
auto utc{ sys_days{2020y/September/15d} + 9h + 35min + 10s };//2020-09-15 09:35:10 UTC

year_month_day fulldate1 { 2020y, September, 15d };

auto fulldate2 { 2020y / September / 15d };

year_month_day fulldate3 { Monday[3] / September / 2020 };
```

--

## `<chrono>` Durations
New duration type aliases: `days`, `weeks`, `months`, `years`
```cpp
weeks w{1}; // 1 week
days d{w}; // convert 1 week into days
```
* years = 365.2425 days (the average length of a Gregorian year)
* months = 30.436875 days (exactly 1/12 of years)

--

## `<chrono>` Clocks and Time ##
New clocks (besides `system_clock`, `steady_clock`, `high_resolution_clock`):
`utc_clock`, `tai_clock`, `gps_clock`, `file_clock`

New type alias `sys_time`: a time_point of a `system_clock` with certain duration
```cpp
using sys_days = sys_time<std::chrono::days>;

// date --> time_point
system_clock::time_point t { sys_days { 2020y / September / 15d } };

// time_point -> date
auto yearmonthday { year_month_day { floor<days>(t) } };
```
in this example the resolution is a day

--

## `<chrono>` Timezones ##
```cpp
auto utc{ sys_days{2020y/September/15d} + 9h + 35min + 10s };//2020-09-15 09:35:10 UTC

// UTC to Denver
zoned_time denver { "America/Denver", utc };

// Get current local time
auto localt { zoned_time { current_zone(), system_clock::now() } };

cout << localt << endl;  // 2020-09-15 09:35:10.153

```
https://godbolt.org/z/jeoxTnW41

---

## `<bit>` operations ##
Set of global non-member functions to operate on bits
* Rotate: rotl(), rotr()
* Counting
  * countl_zero(): number of consecutive 0 bits starting from most significant bit
  * countl_one(): number of consecutive 1 bits starting from most significant bit
  * countr_zero(): number of consecutive 0 bits starting from least significant bit
  * countr_one(): number of consecutive 1 bits starting from least significant bit
  * popcount(): number of 1 bits

## std::byteswap C++23
```cpp
static_assert(std::byteswap(0x12345678) == 0x78563412);
```
https://godbolt.org/z/hKnrhrEhW

---

## C++20 STL improvements for strings and containers
* `starts_with()` and `ends_with()` for strings and string_views:
```cpp
std::string str { "Hello world!" };
if(str.starts_with("Hello")) ....
```
* `contains()` for associative containers C++20 and string/string_view C++23:
```cpp
std::map myMap { std::pair {1, "one"s}, {2, "two"s}, {3, "three"s} };
if(myMap.contains(2)) ....;

std::println("{}", haystack.contains("Hello"sv));
```
* `erase()` and `erase_if()` for all containers remove the need for the remove-erase-idiom
```cpp
std::map<int, char> data
{ {1, 'a'}, {2, 'b'}, {3, 'c'}, {4, 'd'} };

const auto count = std::erase_if(data, [](const auto& item) {
  return (item.first & 1) == 1; });

std::println("removed {}: {}", count, data);
// removed 2: {2: 'b', 4: 'd'}
```
https://www.godbolt.org/z/WzqfPcafj

---

## various C++20 STL improvements
* `remove()`, `remove_if()`, and `unique()` for `list` and `forward_list` now return size_type removed elements
* `shift_left()` and `shift_right()` added to `<algorithm>`, shifts elements in a range
* `midpoint()` to calculate the midpoint of two numbers (better than naive formula (a+b)/2 which may trigger integer overflow)
* `lerp()` to do linear interpolation
* New unsequenced_policy `execution::unseq`: algorithm is allowed to be vectorized, e.g., executed on a single thread using instructions that operate on multiple data items.

---

## std::numbers
Mathematical constants added to STL into `std::numbers`:
* e, log2e, log10e
* pi, inv_pi, inv_sqrtpi
* ln2, ln10
* sqrt2, sqrt3, inv_sqrt3
* egamma
* phi

---

# C++23 Core Language #
## Literal suffix `uz` for `size_t`
```cpp
// issue on 64-bit
for (auto i = 0, s = v.size(); i < s; ++i){}
// fixed
for (auto i = 0uz, s = v.size(); i < s; ++i){}
```
## `#warning`
```cpp
#warning "You not expected to compile this file"
```
## `auto(x)` decay-copy
```cpp
extern void process(int&& value);
for (const auto& i : data) {
  process(auto(i)); // prvalue, C++23, creates a copy
  auto c = i; // lvalue
  process(c); // not compiles
  process(std::move(c)); // works, force converted to rvalue
}
```

--

# C++23 Core Language #
## `#elifdef`

* `#elifdef id` <==> `#elif defined(id)`
* `#elifndef id` <==> `#elif !defined(id)`


## garbage collection support removed

* Introduced in C++11, but never used.

---

## deducing **this**
Basic syntax
```cpp
// usual definition
void U::setValue(int val)
{ m_value = val; }

// same definition with explicit this
void U::setValue(this UU& self,
int val)
{ self.m_value = val; }

// call
u.setValue(42);
```
Usecases:
* to unify overloads, especially in template
* to replace CRTP
* to write recursive lambda

--

## deducing **this** - ref overloading
Usecase: streamline overloads on ref-qualified members

```cpp
std::string& UU::GetName() &
{ return m_name; }

const std::string& UU::GetName() const &
{ return m_name; }

std::string&& UU::GetName() &&
{ return std::move(m_name); }

// with deducing this:

template <typename Self>
auto&& UU::GetName(this Self&& self)
{ return std::forward<Self>(self).m_name; }

```

--

## deducing **this** - lambda
Usecase: recursive lambda expression
```cpp
auto fibonacci = [](this auto self, int n) {
  if (n < 2) { return n; }
  return self(n - 1) + self(n - 2);
};
```
* **this** points to object containing lambda, not lambda itself

--

## deducing **this** - CRTP
Classic code:
```cpp
template <typename Derived> struct Base {
  void interface() { static_cast<Derived*>(this)->implementation(); }
  void implementation(){ throw not_impl{}; }
};

struct Derived1: Base<Derived1>{
  void implementation() { Do1(); }
};

struct Derived2: Base<Derived2>{
  void implementation() { Do2(); }
};

template <typename T>
void execute(T& base) { base.interface(); }
```
Curiously Recurring Template Pattern is typically used to implement static polymorphism. A technique in C++ in which a class Derived is derived from a class template Base. The critical point is that Base has Derived as a template argument.

--

## deducing **this** - CRTP
New code:
```cpp
struct Base {
  template <typename Self> void interface(this Self&& self) { self.implementation(); }
  void implementation(){ throw not_impl{}; }
};

struct Derived1: Base{
  void implementation() { Do1(); }
};

struct Derived2: Base{
  void implementation() { Do2(); }
};

template <typename T>
void execute(T& base) { base.interface(); }
```
No longer Curiousely Recursive.

No weird static cast. Easier to reason.

---

# C++23 Standard Library
* std::optional Monadic Operations
* std::expected
* std::flat_(multi)map / flat_(multi)set
* stacktrace Library
* Not constructing string from nullptr
* std::move_only_function
* basic_string::resize_and_overwrite()
* std::to_underlying()
* better (heretoheneous) map.erase

---

## std::optional Monadic Operations

C++17 optional
```cpp
std::optional<int> getInt(std::string arg)
{
  try { return {std::stoi(arg)}; }
  catch (...) { return { }; }
}

std::cout << getInt("banana").value_or(-1);

if(auto ox = getInt("+42"); ox)
  std::cout << *ox;

std::optional<int> getFirst(const std::vector<int>& vec) {
  if ( !vec.empty() ) return std::optional<int>(vec[0]);
  else return std::optional<int>();
}
```

In C++23, std::optional is extended with monadic operations opt.and_then, opt.transform, and opt.or_else.

```cpp
auto res = getFirst(arr)
  .and_then(getInt)
  .transform( [](int n) { return n + 100;})
  .transform( [](int n) { return std::to_string(n); })
  .or_else([] { return std::optional{std::string("Error") }; });
```
* `and_then` returns the result of the given function call if it exists or an empty std::optional.
* `transform` returns a std::optional containing its transformed value or an empty std::optional.
* `or_else` returns the std::optional if it contains a value or the result of the given function otherwise.

---

## std::expected

Standard way to return value or error. Similar to `std::variant`.
```cpp
using eint = std::expected<int, std::string>;

eint half(int n)
{
  if(1 == (n & 1))
    return std::unexpected("half can only work with odd numbers");
  return n / 2;
}
```
```cpp
// option-like read
int dontcare = half(13).value_or(0);

// option-like check, also has_value() may be used
if(auto result = half(42); half)
  cornevity = result.value() + 3;
else
  throw MyExcept("Cornevity undefined - " + result.error());

// inplace construction
std::vector<eint> arr;
arr.emplace(17);
```
* std::expected never allocates, always holds  value or error
* `value()` throws std::bad_expected_access if object is error.
* std::expected supports monadic extention, simiar to `std::option`: exp.`and_then`, exp.`transform`, exp.`or_else`, and exp.`transform_error`

https://www.modernescpp.com/index.php/c23-a-new-way-of-error-handling-with-stdexpected/

---

## std::flat_(multi)map / std::flat_(multi)set
* map --> flat_map   multimap --> flat_multimap
* set --> flat_set   multiset --> flat_multiset
* keys and values stored flat in two continuos containers
* fast, cache-friendly
* long tested with `boost`
```cpp
std::flat_map<int, std::string> myMap;
myMap[2024] = "CppCon"s;

std::flat_map<int, std::string,
              std::less<int>,
              std::deque<int>,
              std::deque<std::string>> myMap;
```

---

## Stacktrace Library
```cpp
std::println("{}", std::stacktrace::current());
```
Example of custom exception automatically capturing stack trace instantiation:
```cpp
class MyException : public std::exception {
public:
   MyException(std::string message, std::stacktrace st = std::stacktrace::current())
      : m_message { std::move(message) }
      , m_stacktrace { std::move(st) } {}
   const char* what() const noexcept override { return m_message.c_str(); }
   const std::stacktrace& trace() const noexcept { return m_stacktrace; }
private:
   std::string m_message;
   std::stacktrace m_stacktrace;
};
```

```cpp
if(failure)
  throw MyException("TestFailure");
```

```cpp
int main() {
   try { foo(); }
   catch (const MyException& e) {
      std::println("Exception caught: {}", e.what());
      std::println("Stacktrace:\n{}", e.trace());
   }
}
```

---

## string cannot be constructed on nullptr
C++17: constructing strings from nullptr was undefined behaviour. Now - error.
```cpp
std::string ss{nullptr}; // C++23 compile error
std::string_view{nullptr}; // C++23 compile error
```
## std::move_only_function
Same as std::function, but movable.
```cpp
int Process(std::move_only_function<int()> f) { return f() * 2; }

std::print("{}", Process([p = std::make_unique<int>(42)] { return *p; }));
// 84
```

## std::to_underlying()
You cannot directly print enum class. So for logging and other purposes used `std::underlying_type_t` or C++23 short form `std::to_underlying`:
```cpp
enum class Color : uint32_t { Blue = 0x0000ff };
constexpr b1 = std::to_underlying(Color::Blue);
constexpr b2 = static_cast<std::underlying_type_t<Color>>(Color::Blue);
static_assert(b1 == b2);
std::println("{}", b1);
```
https://godbolt.org/z/vbbMT7M9r


---

## basic_string::resize_and_overwrite()

Sometimes std::string used as a cache buffer to communicate with some legacy API which provide data in a following way:
```cpp
// Before actual data transfer something measuring what is the maxumum necessary buffer size
const auto bufsize = ::Transmogrify(essense.data(), essense.size(), nullptr);
// Caller has to prepare buffer enough to fit data. Exising string buffer re-used between calls.
m_buf.resize(bufSize, 0);
// Actual call writes data and returns number of actually written bytes.
const auto actual = ::Transmogrify(essense.data(), essense.size(), m_buf.data());
// Caller has to downright container to actual size.
m_buf.resize(actualSize, 0);
```
Here and in some other cases, stuffing zeros into container during resize is extra work, as data will be anyway either replaced or not used.

```cpp
// less calls, less CPU, less readable
const auto bufsize = ::Transmogrify(essense.data(), essense.size(), nullptr);

m_buf.resize_and_overwrite(bufsize, [](char* buf, std::size_t buf_size) noexcept
    {
       const auto actual = ::Transmogrify(essense.data(), essense.size(), buf);
        return buf_size;
    });
```

---

## better (heretoheneous) lookup and erase
* Problem in C++11 with associative containers:
```cpp
std::map<std::string, int> mapint;
auto it = mapint.find("x"); // inefficient
```
lookup operations creating temporary std::string{"x"} object, because comparison operator expect `const std::string&` argument. Performance silently lost.

* Solution
  * C++14 for std::map lookup
  * C++20 for std::unordered_map lookup
  * C++23 for map/unordered_map erase and extract
```cpp
std::map<std::string, int, std::less<>> mapint;
```
This reroutes comparison through specialy tailored template containing flag `using is_transparent = void` which enable transparent comparisons.

--

## better (heretoheneous) lookup

* Note: this is NOT default, developer has to change code adding extra `less` into template arguments.


* According to some sources, comittee decided to not risk backward compatibility in case if there are maps in the wild where key type has costly conversion to `const char*` but has no defined cheap `operator<` defined.


https://www.cppstories.com/2021/heterogeneous-access-cpp20/


---


# Reference

* C++20: An (Almost) Complete Overview - Marc Gregoire - CppCon 2020
    * https://youtu.be/FRkJCvHWdwQ

* How C++20 Changes the Way We Write Code - Timur Doumler - CppCon 2020
    * https://www.youtube.com/watch?v=ImLFlLjSveM&ab_channel=CppCon

* The Small Pearls of C++20 - Rainer Grimm - C++ on Sea 2023
    * https://www.youtube.com/watch?v=BhMUdb5fRcE&ab_channel=cpponsea
    * https://www.modernescpp.org/wp-content/uploads/2023/06/C20TheSmallPearlsCppOnSea.pdf

* C++23: An Overview of Almost All New and Updated Features - Marc Gregoire - CppCon 2023
    * https://youtu.be/Cttb8vMuq-Y


--

# Reference

* CppReference
    * https://en.cppreference.com/w/cpp

* ModernesCPP (Rainer Grimm blog)
    * https://www.modernescpp.com/index.php/category/blog/c-20/
    * https://www.modernescpp.com/index.php/category/blog/c-23/

* CppCon
    * https://www.youtube.com/@CppCon
    * https://github.com/CppCon/

* CppOnSea
    * https://www.youtube.com/@cpponsea
    * https://github.com/philsquared/cpponsea-slides

* An Introduction to Multithreading in C++20 - Anthony Williams - CppCon 2022
    * https://www.youtube.com/watch?v=A7sVFJLJM-A&ab_channel=CppCon

* Effective Ranges: A Tutorial for Using C++2x Ranges - Jeff Garland - CppCon 2023
    * https://www.youtube.com/watch?v=QoaVRQvA6hI&ab_channel=CppCon

