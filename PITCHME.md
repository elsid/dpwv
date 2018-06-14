# Dynamic polymorphism without virtual

---

## Problem

```
GET /users/23474322/contacts/31

PUT /users/23474322/lists

DELETE /users/23474322/tags/3
```

---

## State Machine

Make input
```
GET /users/23474322/contacts/31
->
path = ["users", "23474322", "contacts", "31"], method = "GET"
```

Process input
...

---

## Handlers

```c++
void get_contact(UsersId user_id, ContactId contact, Stream& stream);
void create_list(UsersId user_id, Stream& stream);
void delete_tag(UsersId user_id, TagId tag_id, Stream& stream);
```

---

## Simple problem

```c++
struct Fizz;
struct Buzz;

void foo(Fizz);
void bar(Buzz);

// If type is foo should construct Fizz from given value and call foo.
// If type is bar should construct Buzz from given value and call bar.
// Otherwise should throw an exception.
void dispatch(std::string type, std::string value);
```

---

## If-else solution

```c++
void dispatch(std::string type, std::string value) {
  if (type == "foo") {
    foo(Fizz {std::move(value)});
  } else if (type == "bar") {
    bar(Buzz {std::move(value)});
  } else {
    using std::string_literals;
    throw std::invalid_argument("Unknown type: "s + type);
  }
}
```

---

## What if there are more?
```c++
void foo(Fizz);
void bar(Buzz);
void far(Fuzz);
void boo(Bizz);
void bor(Zzzz);
void fao(Ffff);
void bao(Bbbb);
void bbb(Bfzz);
void fff(Ffzz);
void wtf(Ziff);

void dispatch(std::string type, std::string value);
```

---

## Ok, no problem

```c++
void dispatch(std::string type, std::string value) {
  if (type == "foo") { foo(Fizz {std::move(value)}); }
  } else if (type == "bar") { bar(Buzz {std::move(value)});
  } else if (type == "far") { far(Fuzz {std::move(value)});
  } else if (type == "boo") { boo(Bizz {std::move(value)});
  } else if (type == "bor") { boo(Zzzz {std::move(value)});
  } else if (type == "fao") { fao(Ffff {std::move(value)});
  } else if (type == "bao") { bao(Bbbb {std::move(value)});
  } else if (type == "bbb") { bbb(Bfzz {std::move(value)});
  } else if (type == "fff") { fff(Ffzz {std::move(value)});
  } else if (type == "wtf") { wtf(Ziff {std::move(value)});
  } else { throw std::invalid_argument("Go home"); }
}
```

---

## No, wait...

Was there any typo?

---

<section><pre><code>
void dispatch(std::string type, std::string value) {
  if (type == "foo") { foo(Fizz {std::move(value)});
  } else if (type == "bar") { bar(Buzz {std::move(value)});
  } else if (type == "far") { far(Fuzz {std::move(value)});
  } else if (type == "boo") { boo(Bizz {std::move(value)});
  } else if (type == "bor") { <mark>boo</mark>(Zzzz {std::move(value)});
  } else if (type == "fao") { fao(Ffff {std::move(value)});
  } else if (type == "bao") { bao(Bbbb {std::move(value)});
  } else if (type == "bbb") { bbb(Bfzz {std::move(value)});
  } else if (type == "fff") { fff(Ffzz {std::move(value)});
  } else if (type == "wtf") { wtf(Ziff {std::move(value)});
  } else { throw std::invalid_argument("Go home"); }
}
</pre></code></section>

---

## How about little optimization?

```c++
enum Type;
Type make_type(std::string);
void dispatch(std::string type, std::string value) {
  switch (make_type(type)) {
    case Type::foo: foo(Fizz {std::move(value)});
    case Type::bar: bar(Buzz {std::move(value)});
    case Type::far: far(Fuzz {std::move(value)});
    case Type::boo: boo(Bizz {std::move(value)});
    case Type::bor: bor(Zzzz {std::move(value)});
    case Type::fao: fao(Ffff {std::move(value)});
    case Type::bao: bao(Bbbb {std::move(value)});
    case Type::bbb: bbb(Bfzz {std::move(value)});
    case Type::fff: fff(Ffzz {std::move(value)});
    case Type::wtf: wtf(Ziff {std::move(value)});
  }
}
```

---

## How about virtual?

```c++
struct Base {
  virtual void call(std::string value) = 0;
};
using Map = std::map<Type, std::unique_ptr<Base>>;
void dispatch(Map alternatives, std::string type, std::string value) {
  alternatives.at(make_type(type))->call(std::move(value));
}
struct Foo : Base {
  void call(std::string value) override {
    const Fizz typed_value {value};
    // ...
  }
};
```

---

## How about different number of arguments?

```c++
void foo(Fizz);
void bar(Fizz, Buzz);
```

---

## Interface class

```c++
struct Interface {
  using It = std::vector<std::string>::const_iterator;

  virtual int dispatch0(It it, It end) const = 0;
  virtual int dispatch1(It it, It end, const std::string& arg0) const = 0;
  virtual int dispatch2(It it, It end, const std::string& arg0,
                        const std::string& arg1) const = 0;
  // ...
};
```

---

## Dispatcher class

```c++
struct Dispatcher : Interface {
  std::map<std::string, std::shared_ptr<Interface>> alternatives;
  void dispatch2(It it, It end, const std::string& arg0,
                 const std::string& arg1) const override {
    if (it == end) {
      throw std::invalid_argument("path is not found");
    }
    const auto next = alternatives.find(*it);
    if (next != alternatives.end()) {
      return next->second->dispatch2(++it, end, arg0, arg1);
    } else {
      const auto& arg2 = *it;
      return dispatch3(++it, end, arg0, arg1, arg2);
    }
  }
};
```

---

## Endpoint class

```c++
struct Foo : Interface {
  void dispatch1(It, It, const std::string& arg0) const override {
    return foo(Fizz {arg0});
  }
};
struct Bar : Interface {
  void dispatch0(It, It) const {
    throw std::logic_error("invalid call");
  }
  void dispatch1(It, It, const std::string&) const {
    throw std::logic_error("invalid call");
  }
  void dispatch2(It, It, const std::string& arg0,
                 const std::string& arg1) const override {
    return bar(Fizz {arg0}, Buzz {arg1});
  }
};
```

---

## Or we can use tuples as for arguments

```c++
struct Base {
  using It = std::vector<std::string>::const_iterator;
  using StringCref = std::reference_wrapper<const std::string>;

  virtual int dispatch0(It it, It end, const std::tuple<>& arguments) const = 0;
  virtual int dispatch1(It it, It end, const std::tuple<StringCref>& arguments) const = 0;
  virtual int dispatch2(It it, It end, const std::tuple<StringCref, StringCref>& arguments) = 0;
  // ...
};
```

---

## Or even more general approach

```c++
struct Base {
  using Arguments = std::variant<
    std::tuple<>,
    std::tuple<StringCref>,
    std::tuple<StringCref, StringCref>,
    // ...
  >;
  virtual void dispatch(It it, It end, const Arguments& arguments) = 0;
};
```

---

## Then endpoint class will be visitor

```c++
template <class Visitor>
struct Endpoint : Interface {
  void dispatch(It, It, const std::string&, const Arguments& arguments) const override {
    return std::visit(
      [] (const auto& arguments) { return hana::unpack(arguments, Visitor {}); },
      arguments
    );
  }
};
struct BarVisitor {
  template <class ... Ts>
  void operator ()(Ts&& ...) const {
    throw std::logic_error("invalid call");
  }
  void operator ()(Base::StringCref arg0, Base::StringCref arg1) const {
    return bar(Fizz {arg0.get()}, Buzz {arg1.get()});
  }
};
using Bar = Endpoint<BarVisitor>;
```

---

## Why do need virtual at first?

If we can't do this...
```c++
class Inteface {
  template <class ... Ts>
  virtual void dispatch(It it, It end, Ts&& ...) = 0;
};
```
...but can do with std::variant and std::visit.

---

## Let's do it different way

Back to the original problem...

---

## Back to the original problem

```c++
void get_contact(UsersId user_id, ContactId contact, Stream& stream);
void create_list(UsersId user_id, Stream& stream);
void delete_tag(UsersId user_id, TagId tag_id, Stream& stream);

const auto api =
  "users"_l / user_id / (
    "contacts"_l / contact_id / "GET"_m(get_contact)
    | "lists"_l / "PUT"_m(create_list)
    | "tags"_l / tag_id / "DELETE"_m(delete_tag)
  );

template <class Api>
void dispatch(const Api& api, const std::vector<std::string>& path, const std::string& method);
```

@[4-10](Define API by DSL using overloaded operators)
@[12-13](Call API method using given input)

---

## Location

```c++
template <class CharT, CharT ... c>
constexpr auto operator "" _l() {
  return location<hana::string<c ...>>;
}

template <class CharT, CharT ... c>
constexpr auto operator "" _location() { /*...*/ }
```

---

## Location

```c++
struct LocationTag {};

template <class T>
struct Location {
  using address_type = T;
  using hana_tag = LocationTag;
};

template <class T>
constexpr auto location = Location<std::decay_t<T>> {};
```

---

## Method

```c++
struct MethodTag {};

template <class T>
struct Method {
  using name_type = T;

  template <class T>
  constexpr auto operator ()(T&& v) const {
    return tree(hana::make_tuple(hana::make_pair(*this, handler(std::forward<T>(v)))));
  }
};
```

---

## Handler

```c++
struct HandlerTag {};

template <class T>
struct Handler {
  using hana_tag = HandlerTag;
  T impl;
};

template <class T>
constexpr auto handler(T&& value) {
  return Handler<std::decay_t<T>> {std::forward<T>(value)};
};
```

---

## Tree

```c++
struct TreeTag {};

template <class T>
struct Tree {
  using hana_tag = TreeTag;
  T value;
};

template <class T>
constexpr auto tree(T&& value) {
  return Tree<std::decay_t<T>> {std::forward<T>(value)};
}
```

---

## Parameter

```c++
struct ParameterTag {};

template <class T>
struct Parameter {
  using value_type = T;
  using hana_tag = ParameterTag;
};

template <class T>
constexpr auto parameter = Parameter<std::decay_t<T>> {};

constexpr auto user_id = parameter<UserId>;
```

---

## Sequence operator

```c++
template <class Lhs, class Rhs>
constexpr auto operator /(const Tree<Location<Lhs>>& lhs, const Location<Rhs>& rhs) {
  return lhs.value / rhs;
}

template <class Lhs, class Rhs>
constexpr auto operator /(const Location<Lhs>& lhs, const Location<Rhs>& rhs) {
  return tree(hana::make_tuple(hana::make_pair(lhs, rhs)));
}

template <class Lhs1, class Lhs2, class Rhs>
constexpr auto operator /(const hana::pair<Lhs1, Lhs2>& lhs, const Location<Rhs>& rhs) {
  return tree(hana::make_tuple(hana::make_pair(hana::first(lhs),
                               (tree(hana::second(lhs)) / rhs).value)));
}
```

@[1-4](Append location)
@[6-10](Concat locations)
@[12-16](Rotate tree)

---

## Alternatives operator

```c++
template <class ... Lhs, class ... Rhs>
constexpr auto operator |(const Tree<hana::tuple<Lhs ...>>& lhs,
                          const Tree<hana::tuple<Rhs ...>>& rhs) {
  return tree(hana::concat(lhs.value, rhs.value));
}
```
---

## API object type

```c++
using Users = decltype("users"_s);
using Contacts = decltype("contacts"_s);
using Tags = decltype("contacts"_s);
using API = Tree<hana::tuple<
  hana::pair<Location<decltype("users"_s)>, hana::tuple<
    hana::pair<Parameter<UserId>, hana::tuple<
      hana::pair<Location<decltype("contacts"_s)>, hana::tuple<
        hana::pair<Method<decltype("contacts"_s)>, Handler<decltype(get_all_contacts)>
        // ...
      >>,
      hana::pair<Location<decltype("tags"_s)>, hana::tuple<
        // ...
      >>
    >>
  >>
>>;
```

---

## How to call handler?

---

## We can find element in tuple!

---

```c++
// hana::find_if
```

---

## Is there something more efficient?

---

## Binary search!

---

```c++
template <class Result, class ... T, class Value, class Less, class Equal, class Function>
constexpr std::optional<Result> binary_search(const hana::tuple<T ...>& range,
    const Value& value, Less&& less, Equal&& equal, Function&& function) {
  return detail::binary_search_impl<std::size_t(0), sizeof ... (T), Result>(
    range,
    value,
    std::forward<Less>(less),
    std::forward<Equal>(equal),
    std::forward<Function>(function)
  );
}
```
Set low and high at the beginning.

---

```c++
template <std::size_t low, std::size_t high, class Result, class Range, class Value,
          class Less, class Equal, class Function>
constexpr std::enable_if_t<(high - low >= 2), std::optional<Result>> binary_search_impl(
    const Range& range, const Value& value, Less&& less, Equal&& equal, Function&& function) {
  if (less(value, hana::at_c<hana::size_c<(low + high) / 2>>(range))) {
    return binary_search_impl<low, (low + high) / 2, Result>(range, value, std::forward<Less>(less),
      std::forward<Equal>(equal), std::forward<Function>(function));
  } else {
    return binary_search_impl<(low + high) / 2, high, Result>(range, value, std::forward<Less>(less),
      std::forward<Equal>(equal), std::forward<Function>(function));
  }
}
```
@[5-7](If value less than middle use left subrange)
@[8-10](Otherwise use right subrange)

---

For one element just check equality
```c++
template <std::size_t low, std::size_t high, class Result, class Range, class Value,
          class Less, class Equal, class Function>
constexpr std::enable_if_t<(high - low == 1), std::optional<Result>> binary_search_impl(
      const Range& range, const Value& value, Less&&, Equal&& equal, Function&& function) {
  if (equal(value, hana::at_c<low>(range))) {
    return function(hana::at_c<low>(range));
  } else {
    return {};
  }
}
```

---
For empty return empty optional
```c++
template <std::size_t low, std::size_t high, class Result, class Range, class Value,
          class Less, class Equal, class Function>
constexpr std::enable_if_t<(high == low), std::optional<Result>> binary_search_impl(
    const Range&, const Value&, Less&&, Equal&&, Function&&) {
  return {};
}
```

---

Now alternative operator requires ordering
```c++
template <class ... Lhs, class ... Rhs>
constexpr auto operator |(const Tree<hana::tuple<Lhs ...>>& lhs,
                          const Tree<hana::tuple<Rhs ...>>& rhs) {
  return tree(
    hana::concat(hana::sort.by(
        hana::ordering(hana::first),
        hana::concat(hana::filter(lhs.value, is_location), hana::filter(rhs.value, is_location))
      ),
      hana::concat(hana::sort.by(
          hana::ordering(hana::first),
          hana::concat(hana::filter(lhs.value, is_method), hana::filter(rhs.value, is_method))
        ),
        hana::concat(
          hana::filter(lhs.value, is_handler),
          hana::concat(
            hana::filter(rhs.value, is_handler),
            hana::concat(hana::filter(lhs.value, is_parameter), hana::filter(rhs.value, is_parameter))
          ))))); }
```

---

## Let's implement dispatch

```c++
template <class T, class Path, class MethodName, class ... Args>
constexpr void dispatch(const Tree<T>& tree, const Path& path,
        const MethodName& method, Args&& ... args) {
    using ContextType = Context<typename Path::const_iterator, std::decay_t<MethodName>>;
    return dispatch_for_tuple(
        ContextType {std::cbegin(path), std::cend(path), method},
        tree.value,
        hana::tuple<>(),
        std::forward<Args>(args) ...
    );
}
```

---

## Before implement deeper is there anything we forget?

---

## Return value!

---

```c++
template <class Result, class T, class Path, class MethodName, class ... Args>
constexpr CallResult<Result> dispatch(const Tree<T>& tree, const Path& path,
        const MethodName& method, Args&& ... args) {
    using ContextType = Context<typename Path::const_iterator, std::decay_t<MethodName>>;
    return dispatch_for_tuple<Result>(
        ContextType {std::cbegin(path), std::cend(path), method},
        tree.value,
        hana::tuple<>(),
        std::forward<Args>(args) ...
    );
}
```
---

## Return type

```c++
enum class ErrorCode {
  ok,
  methodNotFound,
  locationNotFound,
};

template <class T>
using CallResult = std::variant<
  std::conditional_t<std::is_same_v<T, void>, std::monostate, T>,
  ErrorCode
>;
```

@[1-5](To do not deal with exceptions for errors)
@[7-11](Emulate std::expected)
@[9](Replace void with std::monostate to be able return "void" inside variant)
---

```c++
template <class Result, class Context, class Parameters, class ... Key, class ... Node, class ... Args>
constexpr CallResult<Result> dispatch_for_tuple(const Context& context,
    const hana::tuple<hana::pair<Key, Node> ...>& node, Parameters&& parameters, Args&& ... args) {
  if (!at_path_end(context)) {
    if (auto result = dispatch_for_location<Result>(
      context,
      hana::filter(node, is_location),
      std::forward<Args>(args) ...
    )) {
      return std::move(result).value();
    }
    return dispatch_for_parameter<Result>(
      context,
      hana::filter(node, is_parameter),
      std::forward<Args>(args) ...
    );
  }
  return dispatch_for_method<Result>(
    context,
    hana::filter(node, is_method),
    std::forward<Args>(args) ...
  );
}
```

@[4](If current path iterator is not the end of the path)
@[5-9](Try to use path value as location)
@[10](Return if it is)
@[12-16](Otherwise use path value as parameter)
@[18-22](If path is ended use method)

---

```c++
template <class Result, class Context, class Parameters, class ... Args>
constexpr std::optional<CallResult<Result>> dispatch_for_location(const Context&,
    const hana::tuple<>& /* node */, Parameters&&, Args&& ...) {
  return {};
}

template <class Result, class Context, class Parameters, class ... Key, class ... Node, class ... Args>
constexpr std::optional<CallResult<Result>> dispatch_for_location(const Context& context,
    const hana::tuple<hana::pair<Location<Key>, Node> ...>& node,
    Parameters&& parameters, Args&& ... args) {
  return binary_search<CallResult<Result>>(node, *context.path_it, Less {}, Equal {},
    return_monostate_on_void([&] (const auto& next) {
      return dispatch_for_tuple<Result>(
        with_next_path_iterator(context),
        hana::second(next),
        std::forward<Parameters>(parameters),
        std::forward<Args>(args) ...
      );
    })
  );
}
```

@[4-5](If there is no locations return nullopt)
@[11-20](If there are locations, use binary search to find one)
@[13-17](And do recursive call for next path value)
