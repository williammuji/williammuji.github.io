-sync-signal-safe.markdown  2020-09-02-linux-process-address-space.markdown  2020-09-17-memory-allocator.markdown
ubuntu@jwill:~/codebase/williammuji.github.io/_posts$ vi



--
layout: post
title:  "Abseil C++ Tips"
date: 2021-02-18 10:00:00 +0800
categories: jekyll update
---


001. string_view
055. Name Counting and unique_ptr
007. Temporaries, Moves, and Copies
122. Test Fixtures, Clarity, and Dataflow 
064. Raw String Literals
094. Callsite Readability and bool Parameters
086. Enumerating with Class
101. Return Values, References, and Lifetimes
107. Reference Lifetime Extension
049. Argument-Dependent Lookup
135. Test the Contract, not the Implementation
123. absl::optional and std::unique_ptr
131. Special Member Functions and \= default
042. Prefer Factory Functions to Initializer Methods
074. Delegating and Inheriting Constructors
010. Splitting Strings, not Hairs
003. String Concatenation and operator+ vs. StrCat()
036. New Join API
059. Joining Tuples
142. Multi-parameter Constructors and explicit
088. Initialization: =, (), and {}
061. Default Member Initializers
093. using absl::Span
141. Beware Implicit Conversions to bool
143. C++11 Deleted Functions (= delete)
011. Return Policy
120. Return Values are Untouchable
117. Copy Elision and Pass-by-value
148. Overload Sets
149. Object Lifetimes vs. = delete
136. Unordered Containers
144. Heterogeneous Lookup in Associative Containers
152. AbslHashValue and You
112. emplace vs. push_back
065. Putting Things in their Place
099. Nonmember Interface Etiquette
109. Meaningful `const` in Function Declarations
126. `make_unique` is the new `new`
119. Using-declarations and namespace aliases
130. Namespace Naming
134. make_unique and private Constructors
153. Don't Use using-directives
045. Avoid Flags, Especially in Library Code
147. Use Exhaustive switch Statements Responsibly
161. Good Locals and Bad Locals
168. inline Variables
166. When a Copy is not a Copy
146. Default vs Value Initialization
108. Avoid std::bind
163. Passing absl::optional parameters
171. Avoid Sentinel Values
172. Designated Initializers
173. Wrapping Arguments in Option Structs
175. Changes to Literal Constants in C++14 and C++17
176. Prefer Return Values to Output Parameters
177. Assignability vs. Data Member Types
005. Disappearing Act
140. Constants: Safe Idioms
116. Keeping References on Arguments
165. if and switch statements with initializers
140. Constants: Safe Idioms
181. Accessing the value of a StatusOr<T>
076. Use absl::Status
187. std::unique_ptr Must Be Moved
186. Prefer to Put Functions in the Unnamed Namespace


## 1. string_view
这篇文章最早是April 20, 2012发表的。

如果有个函数传入参数为(constant) string。常见方式有：
```cpp
// C Convention
void TakesCharStar(const char* s);

// Old Standard C++ convention
void TakesString(const std::string& s);

// string_view C++ conventions
void TakesStringView(absl::string_view s);    // Abseil
void TakesStringView(std::string_view s);     // C++17
```
那么如果传入的参数类型与函数定义的参数类型不一致，就会有所变化。
比如std::string -> const char\*
```cpp
void AlreadyHasString(const std::string& s) {
  TakesCharStar(s.c_str());               // explicit conversion
}
```
比如const char\* -> std::string
```cpp
void AlreadyHasCharStar(const char* s) {
  TakesString(s); 		// compiler will make a copy
}
```

所以引入std::string_view，string_view consists of only a pointer and a length, 
identifying a section of character data that is not owned by the string_view and cannot be modified by the view. 

string_view可以implicit由const char\*或者const std::string&转换。
std::string转换为std::string_view只需要O(1)time。
const char\*转换为std::string_view只需要加个strlen。
```cpp
void AlreadyHasString(const std::string& s) {
  TakesStringView(s); // no explicit conversion; convenient!
}

void AlreadyHasCharStar(const char* s) {
  TakesStringView(s); // no copy; efficient!
}
```
注意：
string_view is not necessarily NUL-terminated. 
```cpp
printf("%s\n", sv.data()); // DON’T DO THIS
```
这样写不安全，以下则是安全的。
```cpp
std::cout << "Took '" << s << "'";
```

## 55. Name Counting and unique_ptr

在每个代码行，如果对一个std::unique_ptr有多个names，那肯定错误。
```cpp
std::unique_ptr<Foo> NewFoo() {
  return std::unique_ptr<Foo>(new Foo(1));
}

void AcceptFoo(std::unique_ptr<Foo> f) { f->PrintDebugString(); }

void Simple() {
  AcceptFoo(NewFoo());
}

void DoesNotBuild() {
  std::unique_ptr<Foo> g = NewFoo();
  AcceptFoo(g); // DOES NOT COMPILE!
}

void SmarterThanTheCompilerButNot() {
  Foo* j = new Foo(2);
  // Compiles, BUT VIOLATES THE RULE and will double-delete at runtime.
  std::unique_ptr<Foo> k(j);
  std::unique_ptr<Foo> l(j);
}
```
Simple函数中，只有一个std::unique_ptr的name，就是AcceptFoo的参数f。
DoesNotBuild里却有两个name指向同个std::unique_ptr，g和f。
在任何时刻，一个std::unique_ptr应该只有一个name持有。
如果有多个name持有，可以通过std::move去除一个name。
```cpp
 void EraseTheName() {
   std::unique_ptr<Foo> h = NewFoo();
   AcceptFoo(std::move(h)); // Fixes DoesNotBuild with std::move
}
```

## 77. Temporaries, Moves, and Copies

Two Names: It’s a Copy
```cpp
std::vector<int> foo;
FillAVectorOfIntsByOutputParameterSoNobodyThinksAboutCopies(&foo);
std::vector<int> bar = foo;     // Yep, this is a copy.

std::map<int, string> my_map;
string forty_two = "42";
my_map[5] = forty_two;          // Also a copy: my_map[5] counts as a name.
```

One Name: It’s a Move
```cpp
std::vector<int> GetSomeInts() {
  std::vector<int> ret = {1, 2, 3, 4};
  return ret;
}

// Just a move: either "ret" or "foo" has the data, but never both at once.
std::vector<int> foo = GetSomeInts();
```
```cpp
std::vector<int> foo = GetSomeInts();
// Not a copy, move allows the compiler to treat foo as a
// temporary, so this is invoking the move constructor for
// std::vector<int>.
// Note that it isn’t the call to std::move that does the moving,
// it’s the constructor. The call to std::move just allows foo to
// be treated as a temporary (rather than as an object with a name).
std::vector<int> bar = std::move(foo);
```

Zero Names: It’s a Temporary
```cpp
void OperatesOnVector(const std::vector<int>& v);

// No copies: the values in the vector returned by GetSomeInts()
// will be moved (O(1)) into the temporary constructed between these
// calls and passed by reference into OperatesOnVector().
OperatesOnVector(GetSomeInts());
```

注意 Zombies
```cpp
T bar = std::move(foo);
CHECK(foo.empty()); // Is this valid? Maybe, but don’t count on it.
```

一个知识点，std::move doesn't move
```cpp
std::vector<int> foo = GetSomeInts();
std::move(foo); // Does nothing.
// Invokes std::vector<int>’s move-constructor.
std::vector<int> bar = std::move(foo);
```
只是个rvalue-reference，invoke move constructor

```cpp
void Foo(std::vector<string>* paths) {
  ExpandGlob(GenerateGlob(), paths);
}

std::vector<string> Bar() {
  std::vector<string> paths;
  ExpandGlob(GenerateGlob(), &paths);
  return paths;
}
```
一般来说，value semantics更清晰，简单，编译器还肯能进行优化(如RVO)，同时也减少堆分配(区别于heap)。

## 122. Test Fixtures, Clarity, and Dataflow

On the other hand, if you keep each test simple and as straightforward as possible, 
it’s easier to see that it is correct by inspection, understand the logic, and review for higher quality test logic.

上几个例子：
Dataflow in Fixtures
```cpp
class FrobberTest : public ::testing::Test {
 protected:
  void ConfigureExampleA() {
    example_ = "Example A";
    frobber_.Init(example_);
    expected_ = "Result A";
  }

  void ConfigureExampleB() {
    example_ = "Example B";
    frobber_.Init(example_);
    expected_ = "Result B";
  }

  Frobber frobber_;
  string example_;
  string expected_;
};

TEST_F(FrobberTest, CalculatesA) {
  ConfigureExampleA();
  string result = frobber_.Calculate();
  EXPECT_EQ(result, expected_);
}

TEST_F(FrobberTest, CalculatesB) {
  ConfigureExampleB();
  string result = frobber_.Calculate();
  EXPECT_EQ(result, expected_);
}
```
优化为
```cpp
TEST(FrobberTest, CalculatesA) {
  Frobber frobber;
  frobber.Init("Example A");
  EXPECT_EQ(frobber.Calculate(), "Result A");
}

TEST(FrobberTest, CalculatesB) {
  Frobber frobber;
  frobber.Init("Example B");
  EXPECT_EQ(frobber.Calculate(), "Result B");
}
```

Prefer Free Functions
```cpp
class BobberTest : public ::testing::Test {
 protected:
  void SetUp() override {
    bobber1_ = PARSE_TEXT_PROTO(R"(
        id: 17
        artist: "Beyonce"
        when: "2012-10-10 12:39:54 -04:00"
        price_usd: 200)");
    bobber2_ = PARSE_TEXT_PROTO(R"(
        id: 21
        artist: "The Shouting Matches"
        when: "2016-08-24 20:30:21 -04:00"
        price_usd: 60)");
  }

  BobberProto bobber1_;
  BobberProto bobber2_;
};

TEST_F(BobberTest, UsesProtos) {
  Bobber bobber({bobber1_, bobber2_});
  SomeCall();
  EXPECT_THAT(bobber.MostRecent(), EqualsProto(bobber2_));
}
```
优化为
```cpp
BobberProto RecentCheapConcert() {
  return PARSE_TEXT_PROTO(R"(
      id: 21
      artist: "The Shouting Matches"
      when: "2016-08-24 20:30:21 -04:00"
      price_usd: 60)");
}
BobberProto PastExpensiveConcert() {
  return PARSE_TEXT_PROTO(R"(
      id: 17
      artist: "Beyonce"
      when: "2012-10-10 12:39:54 -04:00"
      price_usd: 200)");
}

TEST(BobberTest, UsesProtos) {
  Bobber bobber({PastExpensiveConcert(), RecentCheapConcert()});
  SomeCall();
  EXPECT_THAT(bobber.MostRecent(), EqualsProto(RecentCheapConcert()));
}
``` 
* 尽量避免使用fixtures
* 尽量编码使用fixture的成员变量，因为这样data flow很难追踪，任何一条代码路径都可能修改这个变量值
* 如果需要对变量进行复杂的初始化，那会让测试用例更难阅读，可以考虑使用一个辅助函数来约定这个初始化以及返回对象
* 如果fixture包含成员变量，尽量避免函数直接操作这些变量，传为参数会让数据流更为清晰
* 测试先写，这样API设计会更好，更清晰

## 64. Raw String Literals

R"tag(whatever you want to say)tag"
字符串常量，tag允许16个字符

以前
```cpp
const char concert_17_raw[] =
    "id: 17\n"
    "artist: \"Beyonce\"\n"
    "date: \"Wed Oct 10 12:39:54 EDT 2012\"\n"
    "price_usd: 200\n";
```
现在
```cpp
const char concert_17_raw[] = R"(
    id: 17
    artist: "Beyonce"
    date: "Wed Oct 10 12:39:54 EDT 2012"
    price_usd: 200)";
```

特例，非空tag，适用于字符串中出现")"情况
```cpp
std::string my_string = R"foo(This contains quoted parens "()")foo";
```

## 86. Enumerating with Class

Unscoped Enumerations
C++11之前就是这种枚举，有两个shortcomings
1. 枚举变量跟枚举类型一个scope
2. 枚举变量可以隐式转为数值

```cpp
enum CursorDirection { kLeft, kRight, kUp, kDown };
CursorDirection d = kLeft; // OK: enumerator in scope
int i = kRight;            // OK: enumerator converts to int
```
然而
```cpp
// error: redeclarations of kLeft and kRight
enum PoliticalOrientation { kLeft, kCenter, kRight };
```
C++11稍作修改，让枚举值变为local，且跟以前兼容
```cpp
CursorDirection d = CursorDirection::kLeft;  // OK in C++11
int i = CursorDirection::kRight;             // OK: still converts to int
```

问题，枚举值可以隐式转为数值经常会造成BUG，另外枚举变量在大型工程，多种库中造成命名冲突。
C++11引入scoped枚举
1. 枚举变量只是枚举类型的内部变量
2. 不可以隐式转为数值
```cpp
enum class CursorDirection { kLeft, kRight, kUp, kDown };
CursorDirection d = kLeft;                    // error: kLeft not in this scope
CursorDirection d2 = CursorDirection::kLeft;  // OK
int i = CursorDirection::kRight;              // error: no conversion
```
其他位置枚举也可以定义同样枚举变量名
```cpp
// OK: kLeft and kRight are local to each scoped enum
enum class PoliticalOrientation { kLeft, kCenter, kRight };
```
有一点，如果需要转为数值，需要显式地转换。

C++还可以设置枚举类型underlying类型
```cpp
// Use "int" as the underlying type for CursorDirection
enum class CursorDirection : int { kLeft, kRight, kUp, kDown };
```
或者
```cpp
// Use "char" as the underlying type for CursorDirection
enum class CursorDirection : char { kLeft, kRight, kUp, kDown };
```
如果枚举变量值超过char类型的大小，编译器会报错。

总之，结论是尽量用scoped枚举，以减少命名空间污染，隐式转换到数值类型带来的BUG。


## 94. Callsite Readability and bool Parameters

```cpp
int main(int argc, char* argv[]) {
  ParseCommandLineFlags(&argc, &argv, false);
}
```
无法知道最后一个参数是啥意思
好的函数名应该比较有可读性，当然参数也应该具备可读性，比如std::string_view s(x, y)，如果以前
没见过string_view，std::string_view(my_str.data(), my_str.size())以及std::string_view s("foo")
就很清晰，如果用bool参数，true和false，就没能一眼知道参数意思，就比如ParseCommandLineFlags函数，
如果再增加一个bool参数，那就更无法一眼知道参数意义。

改进一：
```cpp
int main(int argc, char* argv[]) {
  ParseCommandLineFlags(&argc, &argv, false /* preserve flags */);
}
```
用注释来说明字段意思，但是如果字段代码修改了，注释没有同步呢
改进二：
```cpp
int main(int argc, char* argv[]) {
  ParseCommandLineFlags(&argc, &argv, /*remove_flags=*/false);
}
```
把参数变量名也代入注释，如果用一个可读性更强的变量名更易阅读
改进三：
```cpp
int main(int argc, char* argv[]) {
  const bool remove_flags = false;
  ParseCommandLineFlags(&argc, &argv, remove_flags);
}
```
改进三：
引入enum
```cpp
enum ShouldRemoveFlags { kDontRemoveFlags, kRemoveFlags };

void ParseCommandLineFlags(int* argc, char*** argv, ShouldRemoveFlags remove_flags);

int main(int argc, char* argv[]) {
  ParseCommandLineFlags(&argc, &argv, kDontRemoveFlags);
}
```
或者用enum class
```cpp
enum class ShouldRemoveFlags { kNo, kYes };
…
int main(int argc, char* argv[]) {
  ParseCommandLineFlags(&argc, &argv, ShouldRemoveFlags::kNo);
}
```

## 101. Return Values, References, and Lifetimes

```cpp
const string& name = obj.GetName();
std::unique_ptr<Consumer> consumer(new Consumer(name));
```
这个&引用是否合理，怎么保证，它是否有问题？
case by case，讨论以下数据返回以及如何被保存
三个重要问题
1. 返回类型是啥(GetName())
2. 存储类型是啥(name)
3. 如果返回的是引用，引用的生命期？

以下按类型具体讨论
1. 返回string，保存为string，一般编译器都会RVO，最差也就是一个move
2. 返回string&或者const string&，保存为string，这是个拷贝，按照Tip77的解释，
对于一个引用，那么它有一个名字，然后保存为另一个名字，那么就有两个名字，存在拷贝
3. 返回string，保存为string&，编译不过，引用一个临时变量非法
4. 返回const string&，保存为string&，const限定丢弃，编译不过
5. 返回const string&，保存为const string&，这种方式没有cost(消耗)，但是生命期却是个坑，
因为返回的引用有可能是一个成员的引用，那就得到包含这个成员的对象的生命期了
6. 返回string&，保存为string&，跟5一样，但有所不同，因为没有const，所以任意更改
都会影响到返回前所引用的数据，以及被保存的数据
7. 返回string&，保存为const string&，跟5一样
8. 返回string，保存为const string&，这是个特例，C++支持这种方式，如果对一个临时对象
进行const T&引用，那么该临时对象不会马上析构，直到这个引用离开它的scope。

所以，如果做code review，一般看几个点
1. GetName返回的是引用还是值
2. Consumer构造函数使用的string，const string&，string_view
3. 构造函数对参数是否有生命期要求(如果传入的不是string)
如果把name声明为string，因为RVO关系，并不会比引用效率更低，且更安全，没有引用生命期考虑
另外如果返回的是引用，要考虑生命期，也是name声明为string更好，因为这样切断返回值和name对于
所引用对象的生命期考量。

总的，避免拷贝想法是好的，如果在没有让事情变得复杂，比如考虑引用对象的生命期。
但如果在没有拷贝的情况下，比如RVO，让代码复杂不是个好的权衡。


## 107. Reference Lifetime Extension

Tip101介绍了一点引用和生命期管理，这个tip更深入一点
```cpp
string Foo::GetName();

const string& name = obj.GetName();  // Is this safe/legal?
```
临时对象的生命期延长条件有：
用const T&初始化一个表达式的结果，这个结果返回的是一个临时对象T，或者一个临时T子对象，
比如一个包含T的struct。
具体展开：
1. 必须是const T&，T&不可以
2. 不可以有类型转换，比如用string去初始化const std::string_view&无效
3. 间接获得T子对象也不行，因为编译器不会通过函数来查看，只有直接地初始化临时的的公共成员变量可行
4. 允许当T是U的父类，从U的临时对象赋值给T&，但最好不这样使用，这样使用只会更困惑

如果一个临时对象的生命期被延长，那么将会延长到引用变量离开它的scope。
正如Tip101所建议的，少用这种范式，它不会带来性能优势，相反这种操作是微妙脆弱的，会给code reviewers
和维护者带来烦恼。

以下是该Tip所使用的一些代码展示
```cpp
std::vector<int> GetInts();
for (int i : GetInts()) { }  // lifetime extension on the vector is important

// Return string_views of size 1 for each char in this string.
std::vector<absl::string_view> Explode(const string& s);

// Lifetime extension kicks in on the vector, but *not* on the temporary string!
for (absl::string_view s : Explode(StrCat("oo", "ps"))) { }  // WRONG
```
对于Explode所返回的临时对象vector可以extension lifetime，但是里面所存储的临时对象则不会，因为存的是string_view

以下是错误的
```cpp
MyProto GetProto();

// Lifetime extension *doesn't work* here: sub_protos (a repeated field)
// is destroyed by MyProto going out of scope, and the lifetime extension rules
// don't kick in here to magically lifetime extend the MyProto returned by
// GetProto().  The sub-object lifetime extension only works for simple
// is-a-member-of relationships: the compiler doesn't see that sub_protos()
// itself returning a reference to an sub-object of the outer temporary.
for (const SubProto& p : GetProto().sub_protos()) { }  // WRONG
```
代码解释了以上第3条，糟糕的翻译，编译器没看到sub_protos返回一个外部临时子对象的引用。


## 49. Argument-Dependent Lookup

如果没有加::的函数，比如func(a, b, c)，成为unqualified函数，对于此类函数，编译器会进行
函数声明匹配搜索，当加入了函数参数的命名空间搜索路径，那么往往结果令人疑惑。针对函数参数
命名空间的Argument-Dependent Lookup简称为ADL。

Name Lookup Basics

一个函数调用必须对应一个函数定义，分两步，首先，搜索匹配函数名一堆overloads，重载决议会基于这些
overloads选择一个最优的参数匹配函数。Name lookup只会做搜索，但不会决议哪个函数匹配，甚至它都不会
查看函数参数是否匹配，仅做名字搜索。

```cpp
namespace b {
void func();
namespace internal {
void test() { func(); } // ok: finds b::func().
} // b::internal
} // b
```
from local function scope, to class scope, enclosing class scope, base classes, namespace scope, 
out into enclosing namespaces, finally global ::namespace.
在任何一个scope找到名字匹配便会stop搜索，而并不管此时的函数参数是否匹配。
当在一个scope发现不少于1种匹配函数名，那么就会在这个scope进行重载决议。

```cpp
namespace b {
void func(const string&);  // b::func
namespace internal {
void func(int);  // b::internal::func
namespace deep {
void test() {
  string s("hello");
  func(s);  // error: finds only b::internal::func(int).
}
}  // b::internal::deep
}  // b::internal
}  // b
```

Argument-Dependent Lookup (ADL)
如果函数带参数，那么参数所在的命名空间也会平行搜索，不同的是这些参数所在命名空间搜索不会再外扩至更上层。
函数名字搜索和ADL搜索合并结果，再进行重载决议。

```cpp
namespace aspace {
struct A {};
void func(const A&);  // found by ADL name lookup on 'a'.
}  // namespace aspace

namespace bspace {
void func(int);  // found by lexical scope name lookup
void test() {
  aspace::A a;
  func(a);  // aspace::func(const aspace::A&)
}
}  // namespace bspace
```
如果并行搜索出来的匹配有最优，则重载决议挑选这个最优匹配，如果没有最佳匹配，那么就会报错。
复杂一点例子：
```cpp
namespace aspace {
struct A {};
template <typename T> struct AGeneric {};
void func(const A&);
template <typename T> void find_me(const T&);
}  // namespace aspace

namespace bspace {
typedef aspace::A AliasForA;
struct B : aspace::A {};
template <typename T> struct BGeneric {};
void test() {
  // ok: base class namespace searched.
  func(B());
  // ok: template parameter namespace searched.
  find_me(BGeneric<aspace::A>());
  // ok: template namespace searched.
  find_me(aspace::AGeneric<int>());
}
}  // namespace bspace
```

特殊情况，Type Aliases
```cpp
namespace cspace {
// ok: note that this searches aspace, not bspace.
void test() {
  func(bspace::AliasForA());
}
}  // namespace cspace
```
对于func的参数ADL，搜索的回事aspace，因为AliasForA是typedef aspace::A AliasForA;
还有一个特殊情况，迭代器
```cpp
namespace d {
int test() {
  std::vector<int> vec(a);
  // maybe this compiles, maybe not!
  return count(vec.begin(), vec.end(), 0);
}
}  // namespace d
```
这要看迭代器std::vector<int>::iterator的类型是int\*还是其他类型（如果该类型的命名空间含有count重载函数）
所以调用qualified函数更佳,std::count。
再有一个特殊情况，Overloaded Operators
对于std::cout << obj;函数声明一般是operator<<(std::ostream&, const O::Obj&)，这将会搜索参数std::ostream所在
的std命名空间，还有O::Obj所在的O命名空间。
比较好的作法是把这种函数定义放在O所在的命名空间中，加入把operator<<放在全局::中，那么O命名空间中随意添加一个
另外类型的operator<<就会干扰搜索。
所以一般遵循这样规则，把操作符函数和非成员函数放在同个命名空间的类型定义旁边。

注意的是，int double这种类型并不是全局命名相关，相反，它们没有命名空间一说，也就不参与ADL，指针和数组则跟它们
的对象或数组成员相关。

再注意代码重构，如果对一个unqualifed的函数参数类型修改，可能会影响重载决议，把一个类型移进一个命名空间，并在老
的命名空间里用一个typedef保持兼容性，这只会让问题更难被诊断。同样把一个函数移进一个新的命名空间，然后留下一个
using 声明，可能会造成unqualifed函数找不到其名字搜索，或者更绝的是，它可能找到一个非你所愿的匹配函数。

最后的最后：
C++用了13页规则来阐述名字搜索，特例情况，friend函数，函数所在类scope等搜索。只有掌握以上基础概念才会理解函数决议。


## 135. Test the Contract, not the Implementation

C++可以通过public,protected,private,friend来做访问控制，测试代码也有类似规则，如GooleTest使用FRIEND_TEST宏。

Test the Contract


## 123. absl::optional and std::unique_ptr

How to Store Values
```cpp
#include <memory>
#include "third_party/absl/types/optional.h"
#include ".../bar.h"

class Foo {
  ...
 private:
  Bar val_;
  absl::optional<Bar> opt_;
  std::unique_ptr<Bar> ptr_;
};
```
As a Bare Object
简单、安全，不会为null。但有几个地方不灵活：
1 val\_的生命期跟Foo对象一致，有时候并非所愿。比如若Bar支持move或者swap，那么val\_的数据可以
通过这些操作替换，但是如果有指针或者引用指向val\_，那么所指向的数据可能都会被改变
2 Bar构造函数所需要的参数都要通过Foo类的构造函数进入，一些复杂的expressions可能会比较麻烦

As std::optional<Bar>
介于Bare Object的simplicity和std::unique_ptr的灵活，std::optional可以为空，它可以通过赋值
(opt\_ = ...)或通过(opt\_.emplace(...))来构造。
因为optional内数据是存储在栈上的，所以要注意避免大型对象超过栈大小，同时，哪怕是一个空的optional，
跟一个有数据的optional占用一样大的内存。
相对于Bare Object，optional有两个劣势：
1 构造和析构不是那么明显
2 可能会访问一个不存在的数据(empty)

As std::unique_ptr<Bar>
可以为空，可以用来transfer对象的所有权，从一个裸指针接管所有权(ptr\_ = absl::WrapUnique(...)
如果一个unique_ptr为空，那么没有任何对象分配，仅占用一个指针大小。
用unique_ptr的劣势：
1 存储的对象是Bar还是Bar的子类
2 构造和析构何时发生更难找，因为unique_ptr会被transferred
3 可能会访问到null ptr
4 带来一个间接层，not friendly to CPU caches
5 Bar如果可以copy，但是std::unique_ptr<Bar>不可以，同时它阻止了Foo类可拷贝

[Locality of reference](https://en.wikipedia.org/wiki/Locality_of_reference)

总结:
Prefer bare object, if it works for your case. Otherwise, try absl::optional. As a last resort, use std::unique_ptr.

  . | Bar | absl::optional<Bar> | std::unique_ptr<Bar>
----|-----|---------------------|---------------------
Supports delayed construction |		| ✓	| ✓
Always safe to access 	      |	✓ 	|	| 
Can transfer ownership of Bar |	 	|	| ✓
Can store subclasses of Bar   |		|	| ✓
Movable			      | If Bar is movable     |	If Bar is movable	| ✓
Copyable		      |	If Bar is copyable    | If Bar is copyable      |	
Friendly to CPU caches        |	✓	| ✓	| 
No heap allocation overhead   |	✓	| ✓	| 
Memory usage		      |sizeof(Bar)  |	sizeof(Bar) + sizeof(bool)2  |	sizeof(Bar*) when null, sizeof(Bar*) + sizeof(Bar) otherwise
Object lifetime		      | Same as enclosing scope	| Restricted to enclosing scope	| Unrestricted
Call f(Bar\*)		      | f(&val\_) | f(&opt\_.value()) or f(&\*opt\_)	| f(ptr\_.get()) or f(&\*ptr\_)
Remove value		      | N/A	| opt_.reset(); or opt_ = absl::nullopt; |	ptr_.reset(); or ptr_ = nullptr;


## 131. Special Member Functions and `= default`

What Does =default Do, and Why Would We Use It?
1 可以更换函数访问等级，变为virtual destructor，比如设置copy构造，恢复被抑制的函数，
比如玩家有声明自己的构造函数，你还是可以通过default来推出编译器生成的默认构造函数。
2 编译器生成的拷贝移动构造函数，不用因为添加删除一个成员，而每次都要更新代码
3 如果编译器生成的函数是trivial，那么生成的代码会更快更安全。
4 如果成员支持聚合默认构造函数，那么就可以通过aggregate initialization构造，而如果自己定义的
构造函数则不行。
5 如果显式的写出default可以给我们增加文档注释的地方，相比较于编译器隐式的生成这些函数
6 In a class template, =default is a simple way to declare an operation conditionally, 
based on whether some underlying type provides it.

注意：
当我们在一个特殊成员函数的初始声明上使用=default时，编译器将检查它是否可以为该函数合成一个inline定义。
如果可以，它就会这样做。如果不能，该函数实际上被声明为删除，就像我们写了=delete一样。

如果一个函数的初始声明使用了=default，或者编译器声明了一个不是用户声明的特殊成员函数，
那么就会推导出一个合适的noexcept，从而有可能让代码更快。

How Does It Work?
C++11之前
```cpp
class A {
 public:
  A() {}  // User-provided, non-trivial constructor makes A a non-aggregate.
};
```
C++11之后
```cpp
class C {
 public:
  C() = default;  // misleading: C has a deleted default constructor
 private:
  const int i;  // const => must always be initialized.
};

class D {
 public:
  D() = default;  // unsurprising, but not explicit: D has a default constructor
 private:
  std::unique_ptr<int> p;  // std::unique_ptr has a default constructor
};
```
当在类外使用=default时，默认的函数将不会是trivial的：trivial是由第一个声明决定的
（这样所有的客户端都同意操作是否trivial）。

If you don’t need your class to be an aggregate and you don’t need the constructor to be trivial 
then defaulting the constructor outside of the class definition, like examples E and F below, 
is often a good choice. 
```cpp
class E {
 public:
  E();  // promises to have a default constructor, but...
 private:
  const int i;  // const => must always be initialized.
};
inline E::E() = default;  // compilation error here: would not initialize `i`

class F {
 public:
  F();  // promises to have a default constructor
 private:
  std::unique_ptr<int> p;  // std::unique_ptr has a default constructor
};
inline F::F() = default;  // works as expected
```

推荐：
Prefer =default over writing an equivalent implementation by hand, even if that implementation is just {}. 
Optionally, omit =default from the initial declaration and provide a separate defaulted implementation.

Outside of templates, write =delete instead if =default wouldn’t provide an implementation.


## 42. Prefer Factory Functions to Initializer Methods

```cpp
// foo.h
class Foo {
 public:
  // Factory method: creates and returns a Foo.
  // May return null on failure.
  static std::unique_ptr<Foo> Create();

  // Foo is not copyable.
  Foo(const Foo&) = delete;
  Foo& operator=(const Foo&) = delete;

 private:
  // Clients can't invoke the constructor directly.
  Foo();
};

// foo.c
std::unique_ptr<Foo> Foo::Create() {
  // Note that since Foo's constructor is private, we have to use new.
  return absl::WrapUnique(new Foo());
}
```
对于Foo::Create()，我们可以返回一个完全初始化的对象，也可以通知初始化失败，也可以返回子类型。
这允许你在不改变用户代码的情况下换入不同的实现，甚至可以根据用户的输入动态地选择实现类。

举了个例子，然后说明了优点，现在翻译下为什么要引入这样的范式。
如果允许一个类做一些初始化操作会失败，那么一般会有一个initializer method，它会通过返回值来告知失败，
这种函数一般紧跟类的构造函数，如果初始化失败，那么会再紧跟一个析构函数来销毁。
所以这就存在问题了，有可能调用者会在初始化操作之前调用其他函数，或者在初始化失败后也调用一些其他函数。
那么维护这个类就会有两三个状态，initialized,uninitialized, and initialization-failed.这么多状态就需要
很强的使用准则，类的每个方法都要规定它可以在什么状态下被调用，用户必须遵守这些规则。如果这种纪律失误，
开发人员会倾向于写任何碰巧能用的代码，而不管你打算支持什么。当这种情况开始发生时，可维护性就会急剧下降，
因为你的实现必须支持客户开始依赖的任何预初始化方法调用组合。
最终你的实现反而变成了你的接口。[Hyrum's Law](https://www.hyrumslaw.com/)

用Factory functions有些缺点，因为它返回的是堆对象，所以不适合值语意。
当派生类的构造函数需要初始化它的基类时，工厂函数也不能使用，
所以在基类的protected API中有时需要使用initializer methods。
不过public API仍然可以使用工厂函数。


# 74. Delegating and Inheriting Constructors

如果一个类有多个构造函数，这些构造函数一般会有个共用的初始化，为了避免重复，一般会定义一个private的
SharedInit()。
```cpp
class C {
 public:
  C(int x, string s) { SharedInit(x, s); }
  explicit C(int x) { SharedInit(x, ""); }
  explicit C(string s) { SharedInit(0, s); }
  C() { SharedInit(0, ""); }
 private:
  void SharedInit(int x, string s) { … }
};
```
C++11提供了一个新机制，delegating constructors来处理这些情况。
```cpp
class C {
 public:
  C(int x, string s) { … }
  explicit C(int x) : C(x, "") {}
  explicit C(string s) : C(0, s) {}
  C() : C(0, "") {}
};
```
如果使用delegating constructors，那么就不能使用成员初始化列表，所有的初始化都会由
delegating constructors来处理。如果delegating constructors仅仅是设置成员，单独的
成员初始化列表或类内初始化可能更清晰。

另外一种不太常见的构造函数重复情况
```cpp
class D : public C {
 public:
  void NewMethod();
};
```
仅仅是继承了一个类，新加一个接口，那么D的构造函数怎么写？C++11引入了一个新的机制inheriting constructors。
```cpp
class D : public C {
 public:
  using C::C;  // inherit all constructors from C
  void NewMethod();
};
```
当然，只有派生类没有添加新的成员需要显式初始化，才可以用inheriting constructors，事实上google style guide也
告诫不要用inheriting constructors，除非新成员有类内初始化。


## 10. Splitting Strings, not Hairs

[absl/strings/str_split.h](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_split.h)
google c++团队开发了这个API
absl::StrSplit比较高效，因为内部使用了string_view，没有拷贝。

```cpp
// Splits on commas. Stores in vector of string_view (no copies).
std::vector<absl::string_view> v = absl::StrSplit("a,b,c", ',');

// Splits on commas. Stores in vector of string (data copied once).
std::vector<std::string> v = absl::StrSplit("a,b,c", ',');

// Splits on literal string "=>" (not either of "=" or ">")
std::vector<absl::string_view> v = absl::StrSplit("a=>b=>c", "=>");

// Splits on any of the given characters (',' or ';')
using absl::ByAnyChar;
std::vector<std::string> v = absl::StrSplit("a,b;c", ByAnyChar(",;"));

// Stores in various containers (also works w/ absl::string_view)
std::set<std::string> s = absl::StrSplit("a,b,c", ',');
std::multiset<std::string> s = absl::StrSplit("a,b,c", ',');
std::list<std::string> li = absl::StrSplit("a,b,c", ',');

// Equiv. to the mythical SplitStringViewToDequeOfStringAllowEmpty()
std::deque<std::string> d = absl::StrSplit("a,b,c", ',');

// Yields "a"->"1", "b"->"2", "c"->"3"
std::map<std::string, std::string> m = absl::StrSplit("a,1,b,2,c,3", ',');
```
输入参数、delimiter和返回值都可以定制。


## 3. String Concatenation and operator+ vs. StrCat()

```cpp
std::string foo = LongString1();
std::string bar = LongString2();
std::string foobar = foo + bar;

std::string foo = LongString1();
std::string bar = LongString2();
std::string foobar = absl::StrCat(foo, bar);
```
这两种写法效果基本一致
```cpp
std::string foo = LongString1();
std::string bar = LongString2();
std::string baz = LongString3();
string foobar = foo + bar + baz;

std::string foo = LongString1();
std::string bar = LongString2();
std::string baz = LongString3();
std::string foobar = absl::StrCat(foo, bar, baz);
```
第一种写法，调用了两个operator+。
效果类似于
```cpp
std::string temp = foo + bar;
std::string foobar = std::move(temp) + baz;
```
等价于std::move(temp.append(baz));但是可能第一次生成的临时对象内部分配的内存不够大，
放不了3个string的聚合，就会触发再一次分配更大空间的操作，所以最坏的情况是，
n个字符串连接的链需要O(n)次重分配。

这就是absl::StrCat适用的场景，它会计算合并的字符串长度，预留空间，然后完成合并。
如果换一种
```cpp
foobar += foo + bar + baz;
```
可用
```cpp
absl::StrAppend(&foobar, foo, bar, baz);
```
更高效。absl::StrCat和absl::StrAppend适用于int32_t, uint32_t, int64_t, uint64_t, float, double, 
const char\*, and string_view，比如
```cpp
std::string foo = absl::StrCat("The year is ", year);
```


## 36. New Join API

absl::StrJoin()
可以Join std::string, absl::string_view, int, double – any type that absl::StrCat() supports.
如果StrCat不支持的类型，也可以通过定制Formatter来Join。

```cpp
std::vector<std::string> v = {"a", "b", "c"};
std::string s = absl::StrJoin(v, "-");
// s == "a-b-c"

std::vector<absl::string_view> v = {"a", "b", "c"};
std::string s = absl::StrJoin(v.begin(), v.end(), "-");
// s == "a-b-c"

std::vector<int> v = {1, 2, 3};
std::string s = absl::StrJoin(v, "-");
// s == "1-2-3"

const int a[] = {1, 2, 3};
std::string s = absl::StrJoin(a, "-");
// s == "1-2-3"
```

以下是定制Formatter

```cpp
std::map<std::string, int> m = {\{"a", 1\}, \{"b", 2\}, \{"c", 3\}};
std::string s = absl::StrJoin(m, ";", absl::PairFormatter("="));
// s == "a=1;b=2;c=3"
```
也可以用lambda来写Formatter
```cpp
std::vector<Foo> foos = GetFoos();

std::string s = absl::StrJoin(foos, ", ", [](std::string* out, const Foo& foo) {
  absl::StrAppend(out, foo.ToString());
});
```
[absl/strings/str_join.h](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_join.h)


## 59. Joining Tuples

2013年发布的absl::StrJoin广受好评，为了丰富API，现又添加了join std::tuple。
```cpp
auto tup = std::make_tuple(123, "abc", 0.456);
std::string s = absl::StrJoin(tup, "-");
s = absl::StrJoin(std::make_tuple(123, "abc", 0.456), "-");

int a = 123;
std::string b = "abc";
double c = 0.456;

// Works, but copies all arguments.
s = absl::StrJoin(std::make_tuple(a, b, c), "-");
// No copies, but only works with lvalues.
s = absl::StrJoin(std::tie(a, b, c), "-");
// No copies, and works with lvalues and rvalues.
s = absl::StrJoin(std::forward_as_tuple(123, MakeFoo(), c), "-");
```
默认对tuple内部元素聚合采用absl::AlphaNumFormatter，同样也可以定制join Formatter。
```cpp
struct Foo {};
struct Bar {};

struct MyFormatter {
  void operator()(string* out, const Foo& f) const {
    out->append("Foo");
  }
  void operator()(string* out, const Bar& b) const {
    out->append("Bar");
  }
};

std::string s = absl::StrJoin(std::forward_as_tuple(Foo(), Bar()), "-",
                         MyFormatter());
EXPECT_EQ(s, "Foo-Bar");
```
absl::StrJoin()API的目标是使用直观和一致的语法加入任何集合、范围、列表或数据组。
我们认为加入std::tuple对象非常适合这个目标，并为API增加了更多的灵活性。


## 142. Multi-parameter Constructors and explicit

大部分构造函数应该声明为explicit
C++11之前explicit常用于单参数的构造函数。
C++11之后explicit对于尖括号的拷贝构造更有意义，比如
```cpp
void f(std::pair<int, int>) with f({1, 2})
```
比如
```cpp
std::vector<char> bad = {"hello", "world"};
```
等等，第二例子有bug，应该是
```cpp
std::vector<std::string> good = {"hello", "world"};
```
回头再讨论这个bug

先谈一个不显式explicit的好处，构造函数如果不显式explicit，编译器会将参数隐式转换为合适的类型，而不用显式写出。
```cpp
// The coordinates of a point in the Cartesian plane, w.r.t. some basis.
class Coordinate2D {
 public:
  Coordinate2D(double x, double y);
  // ...
};

// Computes the Euclidean norm of a given point `p`.
double EuclideanNorm(Coordinate2D p);

// Uses of the non-explicit constructor:
double norm = EuclideanNorm({3.0, 4.0});  // passing a function argument
Coordinate2D origin = {0.0, 0.0};         // initializing with `=`
Coordinate2D Translate(Coordinate2D p, Vector2D v) {
  return {p.x() + v.x(), p.y() + v.y()};  // returning a value from a function
}
```
声明一个没有explicit的Coordinate2D(double, double)，我们可以传入{3.0, 4.0}来代替函数的参数Coordinate2D。

有些情况就不适用，比如调用一个构造函数的input和output是不同类型，或者有前置条件
比如一个Request类，它的构造函数Request(Server\*, Connection\*)，显然Request既不是Server又不是Connection。
同样也有可能有一个Response类，它的构造函数也是Server和Connection，所以通过Server和Connection隐式转为
Request是不合理的，它就应该显式explicit。这样就禁止传入{server, connection}给一个需要Request或Response的函数。
这样做会让代码更清晰。
```cpp
// A line is defined by two distinct points.
class Line {
 public:
  // Constructs the line passing through the given points.
  // REQUIRES: p1 != p2
  explicit Line(Coordinate2D p1, Coordinate2D p2);

  // Determines whether this line contain a given point `p`.
  bool ContainsPoint(Coordinate2D p) const;
};

Line line({0, 0}, {42, 1729});

// Computes the gradient of `line`.  Returns an infinite value if `line` is
// vertical.
double Gradient(const Line& line);
```
通过explicit声明Line(Coordinate2D, Coordinate2D)，限制传入不相关的points给Gradient函数，
传入之前必须转换为Line，因为Line类会对两个点进行检查是否有效。

std::unique_ptr explicit构造转移裸指针的所有权，防止两次删除
```cpp
std::vector<std::unique_ptr<int>> v;
int* p = new int(-1);
v.push_back(p);  // error: cannot convert int* to std::unique_ptr<int>
// ...
v.push_back(p);
```
传入std::unique_ptr必须explicit写出，这样一眼可见所有权转移。

推荐作法：
* Copy constructors and move constructors should never be explicit
* Make a constructor explicit unless its arguments “are” the value of the newly created object. (实际上google style guide要求所有单参数的构造都应该explicit)
* In particular, constructors for types where identity (address) is relevant to the value should be explicit.
* Constructors that impose additional constraints on values (i.e., that have preconditions) should be explicit.  

回到开头那个bug
```cpp
std::vector<char> bad = {"hello", "world"};
```
vector有一个模板构造，可以传入一对pair，因而这两个字符串可以推导为const char\*，如果按本章tip建议，
应该设置该构造函数explicit，因为std::vector<char>不是两个迭代器。

## 88. Initialization: =, (), and {}

C++11提供了一个新的语法uniform initialization，它可以对非常多的类型进行统一的初始化，
同时避免了 Most Vexing Parse，避免了narrowing conversions。
```cpp
int x{2};
std::string foo{"Hello World"};
std::vector<int> v{1, 2, 3};
```
vs.
```cpp
int x = 2;
std::string foo = "Hello World";
std::vector<int> v = {1, 2, 3};
```
用尖括号也会有些问题，比如
```cpp
std::vector<std::string> strings{2}; // A vector of two empty strings.
std::vector<int> ints{2};            // A vector containing only the integer 2.
```
其次，尖括号初始化不够直观，其他语言没有这样做的。
google认为 For uniform initialization syntax, we don’t believe in general that the benefits outweigh the drawbacks.

Best Practices for Initialization
尽量采用赋值来初始化和拷贝构造
```cpp
int x = 2;
std::string foo = "Hello World";
std::vector<int> v = {1, 2, 3};
std::unique_ptr<Matrix> matrix = NewMatrix(rows, cols);
MyStruct x = {true, 5.0};
MyProto copied_proto = original_proto;
```
取代
```cpp
// Bad code
int x{2};
std::string foo{"Hello World"};
std::vector<int> v{1, 2, 3};
std::unique_ptr<Matrix> matrix{NewMatrix(rows, cols)};
MyStruct x{true, 5.0};
MyProto copied_proto{original_proto};
```

采用传统构造语法（括号）来初始化
```cpp
Frobber frobber(size, &bazzer_to_duplicate);
std::vector<double> fifty_pies(50, 3.14);
```
取代
```cpp
// Bad code

// Could invoke an intializer list constructor, or a two-argument constructor.
Frobber frobber{size, &bazzer_to_duplicate};

// Makes a vector of two doubles.
std::vector<double> fifty_pies{50, 3.14};
```

如果以上方法编译不过，才考虑用尖括号
```cpp
class Foo {
 public:
  Foo(int a, int b, int c) : array_{a, b, c} {}

 private:
  int array_[5];
  // Requires {}s because the constructor is marked explicit
  // and the type is non-copyable.
  EventManager em{EventManager::Options()};
};
```

不要混合尖括号和auto
```cpp
// Bad code
auto x{1};
auto y = {2}; // This is a std::initializer_list<int>!
```

Herb Sutter也有一篇关于尖括号初始化相关的文章
(GotW #1 Solution: Variable Initialization – or Is It?)[https://herbsutter.com/2013/05/09/gotw-1-solution/]


## 61. Default Member Initializers

```cpp
class Client {
 private:
  int chunks_in_flight_ = 0;
};
```
```cpp
struct Options {
  bool use_loas = true;
  bool log_pii = false;
  int timeout_ms = 60 * 1000;
  std::array<int, 4> timeout_backoff_ms = { 10, 100, 1000, 10 * 1000 };
};
```

Member Initialization Overrides
```cpp
class Frobber {
 public:
  Frobber() : ptr_(nullptr), length_(0) { }
  Frobber(const char* ptr, size_t length)
    : ptr_(ptr), length_(length) { }
  Frobber(const char* ptr) : ptr_(ptr) { }
 private:
  const char* ptr_;
  // length_ has a non-static class member initializer
  const size_t length_ = strlen(ptr_);
};
```
相当于
```cpp
class Frobber {
 public:
  Frobber() : ptr_(nullptr), length_(0) { }
  Frobber(const char* ptr, size_t length)
    : ptr_(ptr), length_(length) { }
  Frobber(const char* ptr)
    : ptr_(ptr), length_(strlen(ptr_)) { }
 private:
  const char* ptr_;
  const size_t length_;
};
```
结论：
Default member initializers won’t make your program any faster. 
They will help reduce bugs from omissions, especially when someone adds a new constructor or a new data member.


## 93. using absl::Span

在google一般用string_view来传递函数参数和函数返回值，如果不需要对字符串拥有所有权。
它让API更灵活，不做无畏的conversion来提高效率。

absl::string_view还有一个兄弟absl::span，std::span(C++20)，类似于
absl::string_view对于absl::string，absl::Span<const T>对于std::vector<T>。
	
absl::Span<const T>对应的元素（比如数组）不可修改，absl::Span<T>支持non-const访问，
但是要求T的构造函数explicit。

absl::Span的目标是像string_view那样通用的接口。

```cpp
void TakesVector(const std::vector<int>& ints);
void TakesSpan(absl::Span<const int> ints);

void PassOnlyFirst3Elements() {
  std::vector<int> ints = MakeInts();
  // We need to create a temporary vector, and incur an allocation and a copy.
  TakesVector(std::vector<int>(ints.begin(), ints.begin() + 3));
  // No copy or allocations are made when using Span.
  TakesSpan(absl::Span<const int>(ints.data(), 3));
}

void PassALiteral() {
  // This creates a temporary std::vector<int>.
  TakesVector({1, 2, 3});
  // Span does not need a temporary allocation and copy, so it is faster.
  TakesSpan({1, 2, 3});
}
void IHaveAnArray() {
  int values[10] = ...;
  // Once more, a temporary std::vector<int> is created.
  TakesVector(std::vector<int>(std::begin(values), std::end(values)));
  // Just pass the array. Span detects the size automatically.
  // No copy was made.
  TakesSpan(values);
}
```

Buffer Overflow Prevention
因为absl::Span拥有长度，所以API比C风格的指针更安全。
```cpp
// Bad code
void BadUser() {
  int src[] = {1, 2, 3};
  int dest[2];
  memcpy(dest, src, ABSL_ARRAYSIZE(src) * sizeof(int)); // oops! Dest overflowed.
}
// A simple example, but takes advantage that the sizes of the Spans are known
// and prevents the above mistake.
template <typename T>
bool SaferMemCpy(absl::Span<T> dest, absl::Span<const T> src) {
  if (src.size() > dest.size()) {
    return false;
  }
  memcpy(dest.data(), src.data(), src.size() * sizeof(T));
  return true;
}

void GoodUser() {
  int src[] = {1, 2, 3}, dest[2];
  // No overflow!
  SaferMemCpy(absl::MakeSpan(dest), absl::Span<const int>(src));
}
```

Const Correctness for Vector of Pointers
```cpp
void FrobFastWeak(const std::vector<Foo*>& v);
void FrobSlowStrong(const std::vector<const Foo*>& v);
void FrobFastStrong(absl::Span<const Foo* const> v);
```
```cpp
// fast and easy to type but not const-safe
FrobFastWeak(v);
// slow and noisy, but safe.
FrobSlowStrong(std::vector<const Foo*>(v.begin(), v.end()));
// fast, safe, and clear!
FrobFastStrong(v);
```
再举个例子：
```cpp
// Bad code
class DontDoThis {
 public:
  // Don’t modify my Foos, pretty please.
  const std::vector<Foo*>& shallow_foos() const { return foos_; }

 private:
  std::vector<Foo*> foos_;
};

void Caller(const DontDoThis& my_class) {
  // Modifies a foo even though my_class is a reference-to-const
  my_class->foos()[0]->SomeNonConstOp();
}
// Good code
class DoThisInstead {
 public:
  absl::Span<const Foo* const> foos() const { return foos_; }

 private:
  std::vector<Foo*> foos_;
};

void Caller(const DoThisInstead& my_class) {
  // This one doesn't compile.
  // my_class.foos()[0]->SomeNonConstOp();
}
```

## 141. Beware Implicit Conversions to bool

Two Kinds of Null Pointer Checks
```cpp
if (foo) {
  DoSomething(*foo);
}
```
或者
```cpp
if (foo != nullptr) {
  DoSomething(*foo);
}
```
考虑一个情况
```cpp
bool* is_migrated = ...;

// Is this checking that `is_migrated` is not null, or was the actual
// intent to verify that `*is_migrated` is true?
if (is_migrated) {
  ...
}
```
这样写会更清晰
```cpp
// Looks like a null-pointer check for a bool*
if (is_migrated != nullptr) {
  ...
}
```

Optional Values and Scoped Assignments
```cpp
absl::optional<bool> b = MaybeBool();
if (b) { ... }  // What happens when the function returns absl::optional(false)?
```
这样写会更清晰
```cpp
absl::optional<bool> b = MaybeBool();
if (b.has_value()) { ... }
```
当然如果把if判断限定一起更好
```cpp
if (absl::optional<Foo> foo = MaybeFoo()) {
  DoSomething(*foo);
}
```
C++17
```cpp
if (absl::optional<Foo> foo = MaybeFoo(); foo.has_value()) {
  DoSomething(*foo);
}
```

“Boolean-like” Enums
```cpp
void ParseCommandLineFlags(
    const char* usage, int* argc, char*** argv,
    StripFlagsMode strip_flags_mode) {
  if (strip_flags_mode) {  // Wait, which value was true again?
    ...
  }
}
```
改为
```cpp
void ParseCommandLineFlags(
    const char* usage, int* argc, char*** argv,
    StripFlagsMode strip_flags_mode) {
  if (strip_flags_mode == kPreserveFlags) {
    ...
  }
}
```
结论：
* 指针跟nullptr比对
* 容器是否为空用absl::optional<T>::has_value()，避免隐式转化，到底式判断optional为空还是数据为空
* enum判断对应的枚举数值 


## 143. C++11 Deleted Functions (= delete)

一般有4种方法来限制某些接口使用
* Provide a dummy definition consisting solely of a runtime check.(runtime checks)
* Use accessibility controls (protected/private) to make the function inaccessible.(compile-time)
* Declare the function, but intentionally omit the definition.(link time)
* Since C++11: Explicitly define the function as “deleted”.

A compile time check is better, but still flawed. 
It only works for member functions and is based on accessibility constraints

```cpp
class MyType {
 private:
  MyType(const MyType&);  // Not defined anywhere.
  MyType& operator=(const MyType&);  // Not defined anywhere.
  // ...
};
```
```cpp
class MyType : private NoCopySemantics {
  ...
};
```
```cpp
class MyType {
 private:
  DISALLOW_COPY_AND_ASSIGN(MyType);
};
```

C++11 Deleted Definitions
优点：
Any function can be explicitly defined as deleted
对于=default, which works only with special member functions
而=delete，可以用于任何函数
另外
只有在第一次声明函数的时候才可以=delete。

=delete首先是个函数定义，它也会参与name lookup和重载决议
```cpp
class MyType {
 public:
  // Disable default constructor.
  MyType() = delete;

  // Disable copy (and move) semantics.
  MyType(const MyType&) = delete;
  MyType& operator=(const MyType&) = delete;

  //...
};

// error: call to deleted constructor of 'MyType'
// note: 'MyType' has been explicitly marked deleted here
//   MyType() = delete;
MyType x;

void foo(const MyType& val) {
  // error: call to deleted constructor of 'MyType'
  // note: 'MyType' has been explicitly marked deleted here
  //   MyType(const MyType&) = delete;
  MyType copy = val;
}
```
注意把拷贝构造函数显式delete也会抑制move构造函数生成，如果想要手动召出
```cpp
MyType(MyType&&) = default;
MyType& operator=(MyType&&) = default;
```

其他用例：
```cpp
void print(int value);
void print(absl::string_view str);
```
```cpp
void print(int value);
void print(const char* str);
// Use string literals ":" instead of character literals ':'.
void print(char) = delete;
```
限制必须使用string而不是单个字符，而且不但限制了函数调用，还限制函数地址的取用
```cpp
void (*pfn1)(int) = &print;  // ok
void (*pfn2)(char) = &print; // error: attempt to use a deleted function
```
还有
```cpp
// A _very_ limited type:
//   1. Dynamic storage only.
//   2. Lives forever (can't be destructed).
//   3. Can't be a member or base class.
class ImmortalHeap {
 public:
  ~ImmortalHeap() = delete;
  //...
};
```
还有
```cpp
// Don't allow new T[].
class NoHeapArraysPlease {
 public:
  void* operator new[](std::size_t) = delete;
  void operator delete[](void*) = delete;
};

auto p = new NoHeapArraysPlease;  // OK

// error: call to deleted function 'operator new[]'
// note: candidate function has been explicitly deleted
//   void* operator new[](std::size_t) = delete;
auto pa = new NoHeapArraysPlease[10];
```

## 11. Return Policy

对于某个大型对象
```cpp
class SomeBigObject {
 public:
  SomeBigObject() { ... }
  SomeBigObject(const SomeBigObject& s) {
    printf("Expensive copy …\n", …);
    …
  }
  SomeBigObject& operator=(const SomeBigObject& s) {
    printf("Expensive assignment …\n", …);
    …
    return *this;
  }
  ~SomeBigObject() { ... }
  …
};
```
```cpp
static SomeBigObject SomeBigObjectFactory(...) {
  SomeBigObject local;
  ...
  return local;
}
```
看起来不会高效
```cpp
SomeBigObject obj = SomeBigObject::SomeBigObjectFactory(...);
```
实际上大部分主流编译器会做RVO优化。
How Can You Ensure the Compiler Performs RVO?
The called function should define a single variable for the return value:
```cpp
SomeBigObject SomeBigObject::SomeBigObjectFactory(...) {
  SomeBigObject local;
  …
  return local;
}
```
The calling function should assign the returned value to a new variable:
```cpp
// No message about expensive operations:
SomeBigObject obj = SomeBigObject::SomeBigObjectFactory(...);
```
如果不是用一个新的变量就不会RVO
```cpp
// RVO won’t happen here; prints message "Expensive assignment ...":
obj = SomeBigObject::SomeBigObjectFactory(s2);
```
如果调用的函数里返回多余1个变量时，也不会RVO
```cpp
// RVO won’t happen here:
static SomeBigObject NonRvoFactory(...) {
  SomeBigObject object1, object2;
  object1.DoSomethingWith(...);
  object2.DoSomethingWith(...);
  if (flag) {
    return object1;
  } else {
    return object2;
  }
}
```
但如果调用函数里只有一个变量，在多个地方有返回，也可以RVO
```cpp
// RVO will happen here:
SomeBigObject local;
if (...) {
  local.DoSomethingWith(...);
  return local;
} else {
  local.DoSomethingWith(...);
  return local;
}
```

One More Thing: Temporaries
RVO对于非具名的临时对象也适用
```cpp
// RVO works here:
SomeBigObject SomeBigObject::ReturnsTempFactory(...) {
  return SomeBigObject::SomeBigObjectFactory(...);
}
```

A final note: If your code needs to make copies, then make copies, whether or not the copies can be optimized away. 
Don’t trade correctness for efficiency.

Google就是Google，反反复复重复很多次，correctness/simplicity优于efficiency。


## 120. Return Values are Untouchable

```cpp
MyStatus DoSomething() {
  MyStatus status;
  auto log_on_error = RunWhenOutOfScope([&status] {
    if (!status.ok()) LOG(ERROR) << status;
  });
  status = DoA();
  if (!status.ok()) return status;
  status = DoB();
  if (!status.ok()) return status;
  status = DoC();
  if (!status.ok()) return status;
  return status;
}
```
如果重构下代码，把return status;改为return Status(); 

先上结论：
不要在返回语句运行后访问（读或写）函数的返回变量。
因为返回值在被拷贝或者移动之后会隐式的析构，所以此时再去读写就会UB。
当然本Tip讨论的是返回非引用的local变量。

回到最初的问题：
在返回语句上有2种不同的优化，可以修改我们原始代码片段的行为。NRVO和implicit move。
return status之所以可行是因为，函数返回时做了RVO，返回值status直接被构造于返回值上，
清理对象是在返回语句后看到MyStatus对象的这个唯一实例。
return Status()时没有RVO，那么status返回值是被移动到返回值上。RunWhenOutOfScope使用一个
已经被move的数值是未定义的。
he compiler also can’t do RVO if the called function uses more than one variable for the return value.

尽管如此，第一个代码也不正确，它依赖于编译器实现copy elision，并不是所有编译器保证
会做这个优化。

那么Solution:
Do not touch the return variable after the return statement.
重构下
```cpp
MyStatus DoSomething() {
  MyStatus status;
  status = DoA();
  if (!status.ok()) return status;
  status = DoB();
  if (!status.ok()) return status;
  status = DoC();
  if (!status.ok()) return status;
  return status;
}
MyStatus DoSomethingAndLog() {
  MyStatus status = DoSomething();
  if (!status.ok()) LOG(ERROR) << status;
  return status;
}
```
或者显式禁用NRVO
```cpp
MyStatus DoSomething() {
  MyStatus status_no_nrvo;
  // 'status' is a reference so NRVO and all associated optimizations
  // will be disabled.
  // The 'return status;' statements will always copy the object and Logger
  // will always see the correct value.
  MyStatus& status = status_no_nrvo;
  auto log_on_error = RunWhenOutOfScope([&status] {
    if (!status.ok()) LOG(ERROR) << status;
  });
  status = DoA();
  if (!status.ok()) return status;
  status = DoB();
  if (!status.ok()) return status;
  status = DoC();
  if (!status.ok()) return status;
  return status;
}
```
再举个例子：
```cpp
std::string EncodeVarInt(int i) {
  std::string out;
  StringOutputStream string_output(&out);
  CodedOutputStream coded_output(&string_output);
  coded_output.WriteVarint32(i);
  return out;
}
```
改为
```cpp
std::string EncodeVarInt(int i) {
  std::string out;
  {
    StringOutputStream string_output(&out);
    CodedOutputStream coded_output(&string_output);
    coded_output.WriteVarint32(i);
  }
  // At this point the streams are destroyed and they already flushed.
  // We can safely return 'out'.
  return out;
}
```

## 117. Copy Elision and Pass-by-value

```cpp
class Widget {
 public:
  …

 private:
  string name_;
};
```
构造函数怎么写
```cpp
// First constructor version
explicit Widget(const std::string& name) : name_(name) {}
```
或者
```cpp
// Second constructor version
explicit Widget(std::string name) : name_(std::move(name)) {}
```
Isn’t it horribly expensive to pass std::string by copy? 
It turns out that no, sometimes passing by value (as we’ll see, it’s not really “by copy”) 
can be much more efficient than passing by reference.
考虑一个调用代码
```cpp
Widget widget(absl::StrCat(bar, baz));
```
对于第一种构造函数，需要absl::StrCat生成临时string，传引用，然后拷贝进入name\_。
对于第二种构造函数，When the compiler sees a temporary being used to copy-construct an object, 
the compiler will simply use the same storage for both the temporary and the new object。
只有一个move操作。

但如果调用的不是一个临时对象
```cpp
string local_str;
Widget widget(local_str);
```
两种构造都会有一个拷贝，第二种再多一个move。

When to Use Copy Elision
pass by value有些缺点
First of all, it makes the function body more complex, which creates a maintenance and readability burden. 
比如move之后的变量不能误用。
比如Widget有个函数set_name，那么传引用可以复用内存，而传值可能会allocate new memory。

That principle certainly applies to this technique: passing by const reference is simpler and safer, 
so it’s still a good default choice. 


## 148. Overload Sets

敲重点：
In my opinion, one of the most powerful and insightful sentences in the C++ style guide is this: 
“Use overloaded functions (including constructors) only if a reader looking at a call site can 
get a good idea of what is happening without having to first figure out exactly which overload is being called.”

这是一个非常直接的规则：只有在不会给读者造成困惑的情况下才会重载。
然而，这一点的影响其实相当大，并且触及到了现代API设计中的一些有趣的问题。
首先让我们定义一下 "重载集 "这个术语，然后让我们看一些例子。

What is an Overload Set?
Informally, an overload set is a set of functions with the same name that differ in the number, 
type and/or qualifiers of their parameters. 
```cpp
int Add(int a, int b);
int Add(int a, int b, int c);  // Number of parameters may vary

// Return type may vary, as long as the selected overload is uniquely
// identifiable from only its parameters (types and counts).
float Add(float a, float b);

// But if two return types have the same parameter signature, they can't form a
// proper overload set; the following won't compile with the above overloads.
int Add(float a, float b);    // BAD - can't overload on return type
```

举例：
```cpp
void Process(const std::string& s) { Process(s.c_str()); }
void Process(const char*);
```
There is no behavioral difference here: in both cases, we’re accepting some form of string-ish data, 
and the inline forwarding function makes it perfectly clear that the behavior of every member of the overload set is identical.

反例：
```cpp
// remove obstacle from garage exit lane
void open(Gate& g);

// open file
void open(const char* name, const char* mode ="r");
```

经典 StrCat：
This is a good use of overload sets - it would be annoying and redundant to encode the parameter count 
in the function name, and conceptually it doesn’t matter how many things are passed to StrCat() 
- it will always be the “convert to string and concatenate” tool.

经典 Parameter Sinks:
```cpp
void push_back(const T&);
void push_back(T&&);
```

经典 Overloaded Accessors:
```cpp
vector::operator[]
或者
optional::value()
```
```cpp
T& value() &;
const T & value() const &;
T&& value() &&;
const T&& value() const &&;
```

Conclusion:
Overload sets are a simple idea conceptually, but prone to abuse when not well understood 
 don’t produce overloads where anyone might need to know which function was chosen.


## 149. Object Lifetimes vs. = delete

=delete for Lifetimes
```cpp
class Request {
  ...

  // The provided Context must live as long as the current Request.
  void SetContext(const Context& context);
};
```
有一个接口，接受const引用，但如果传入一个临时对象呢，可以用delete禁用
```cpp
class Request {
  ...

  // The provided Context must live as long as the current Request.
  void SetContext(const Context& context);
  void SetContext(Context&& context) = delete;
};
```
Was this a good idea? Why or why not?

Don’t Design In a Vacuum
在没有delete之前
```cpp
request.SetContext(Context());
```
可以编译通过，但是运行会失败，重新看下API，代码修改为
```cpp
request.SetContext(request2.context());
```
加了delete之后，编译期间直接报错
```cpp
error: call to deleted member function 'SetContext'

  request.SetContext(Context());
  ~~~~~~~~^~~~~~~~~~

:4:8: note: candidate function has been explicitly deleted
  void SetContext(Context&& context) = delete;
```
那么代码可能修改为
```cpp
Context context;
request.SetContext(context);
```
有点问题，这个传引用的变量lifetime要保证，起码不能小于request处理这个context的生命期
```cpp
class Request {
  ...

  // The provided Context must live as long as the current Request.
  void SetContext(const Context& context);
  void SetContext(Context&& context) = delete;
};
```
所以可能需要在API处加注释说明。
以这种方式删除一个重载集的成员，充其量也只是事倍功半。
是的，你避免了一类错误，但你也使API复杂化了。
依靠这样的设计肯定会得到一种虚假的安全感：C++类型系统根本无法对参数的寿命要求进行必要的细节编码。

=delete for “Optimization”
我们来颠覆一下情况：也许不是你想要避免临时对象，也许你想避免拷贝。
```cpp
future<bool> DNAScan(Config c, const std::string& sequence) = delete;
future<bool> DNAScan(Config c, std::string&& sequence);
```
How likely is it that a caller of your API will never need to keep their value? 
如果没有delete
```cpp
Config c1 = GetConfig();
Config c2 = GetConfig();
std::string s = GetDNA();

// Kick off scans for both configs.
auto scan1 = DNAScan(c1, s);
auto scan2 = DNAScan(c2, std::move(s));
```
对应两种不同的调用
```cpp
Config c1 = GetConfig();
Config c2 = GetConfig();
std::string s = GetDNA();
std::string s2 = s;

// Kick off scans for both configs.
auto scan1 = DNAScan(c1, std::move(s));
auto scan2 = DNAScan(c2, std::move(s2));
```
在极少的情况下，当你可以确定一个API必须以一种特殊的方式使用时：你可能应该在有关类型中编码。
不然，不要在std::string上操作，作为你对DNA序列的表示；
写一个DNA类，并使它只用一个明确的（容易扫描的）方式来进行昂贵的复制操作。
properties of types should be expressed in those types, not in the APIs that operate on them.

上结论：
It is tempting to try to use rvalue-references or reference qualifiers in conjunction with =delete 
to provide a more “user friendly” API, enforcing lifetimes or preventing optimization problems. 

In practice, those are usually bad temptations. 

Lifetime requirements are much more complicated than the C++ type system can express. 
API providers can rarely predict every future valid usage of their API. 
Avoiding these types of =delete tricks keeps things simple.


## 136. Unordered Containers

Introducing absl::\*\_hash_map
provide developers with direct control over their implementation and default hash functions
New code should prefer these types over std::unordered_map

For every absl::*_hash_map there is also an absl::*_hash_set
absl::flat_hash_map and absl::flat_hash_set

![Image of flat_hash_map](https://abseil.io/img/flat_hash_map.svg)

These should be your default choice. 
They store their value_type inside the main array. 
Because they move data when they rehash, elements don’t get pointer stability. 
If you need pointer stability or your values are large, consider using absl::node_hash_map instead, 
or absl::flat_hash_map<Key, std::unique_ptr<Value>>, possibly.

absl::node_hash_map and absl::node_hash_set
![Image of node_hash_map](https://abseil.io/img/node_hash_map.svg)
like std::unordered_map

We generally recommend that you use absl::flat_hash_map<K, std::unique_ptr<V>> 
instead of absl::node_hash_map<K, V>, and similarly for node_hash_set.

## 144. Heterogeneous Lookup in Associative Containers

Associative containers associate an element with a key. 
Inserting into or looking up an element from the container requires an equivalent key. 
In general, containers require the keys to be of a specific type, 
which can lead to inefficiencies at call sites that need to convert between near-equivalent types
(like std::string and absl::string_view).

为了解决不必要的转换
provide heterogeneous lookup.
This feature allows callers to pass keys of any type (as long as the user-specified comparator functor supports them) 
[std::map::find](http://en.cppreference.com/w/cpp/container/map/find)

Using Heterogeneous Lookup For Performance
常规做法：
```cpp
std::map<std::string, int> m = ...;
absl::string_view some_key = ...;
// Construct a temporary `std::string` to do the query.
// The allocation + copy + deallocation might dominate the find() call.
auto it = m.find(std::string(some_key));
```
use a transparent comparator
```cpp
struct StringCmp {
  using is_transparent = void;
  bool operator()(absl::string_view a, absl::string_view b) const {
    return a < b;
  }
};

std::map<std::string, int, StringCmp> m = ...;
absl::string_view some_key = ...;
// The comparator `StringCmp` will accept any type that is implicitly
// convertible to `absl::string_view` and says so by declaring the
// `is_transparent` tag.
// We can pass `some_key` to `find()` without converting it first to
// `std::string`. In this case, that avoids the unnecessary memory allocation
// required to construct the `std::string` instance.
auto it = m.find(some_key);
```

What Else Is It Good For?
```cpp
struct ThreadCmp {
  using is_transparent = void;
  // Regular overload.
  bool operator()(const std::thread& a, const std::thread& b) const {
    return a.get_id() < b.get_id();
  }
  // Transparent overloads
  bool operator()(const std::thread& a, std::thread::id b) const {
    return a.get_id() < b;
  }
  bool operator()(std::thread::id a, const std::thread& b) const {
    return a < b.get_id();
  }
  bool operator()(std::thread::id a, std::thread::id b) const {
    return a < b;
  }
};

std::set<std::thread, ThreadCmp> threads = ...;
// Can't construct an instance of `std::thread` with the same id, just to do the lookup.
// But we can look up by id instead.
std::thread::id id = ...;
auto it = threads.find(id);
```
thread是不能复制的，所以不好构造一个thread key，这也是一个适用情况

Ordered containers (std::{map,set,multimap,multiset}) support heterogeneous lookup starting in C++14. 
As of C++17, unordered containers still do not support it.

The new family of Swiss Tables, however, support heterogeneous lookup for both string-like types 
(std::string, absl::string_view, etc.) and smart pointers (T\*,std::shared_ptr, std::unique_ptr). 
They require both the hasher and the equality function to be tagged as transparent. 
All other key types require explicit opt-in from the user.

The B-Tree containers (absl::btree\_{set,map,multiset,multimap}) also support heterogeneous lookup.


## 45. Avoid Flags, Especially in Library Code

The common use of flags in production code, especially within libraries, is a mistake. 
Don’t use flags unless it is truly necessary. There, we said it.

Flags are global variables, only worse: you can’t know the value of that variable by reading the code. 
A flag may not only be set at startup, but could potentially be changed later through arbitrary means. 
If you run a server in your binary, there is usually no guarantee that the value of your flag will remain 
unchanged from cycle to cycle, nor any notification if it were to change, nor any means for looking for such a change.

谷歌在2012年初做的一项分析发现，据我们所知，在上述数据保留范围内，大多数C++ Flag实际上从未发生过变化。

Even given those caveats, it’s time that we all take a good hard look at our usage of flags. 
The next time you’re tempted to add a flag to your library, spend a while looking for a better way. 
Pass configuration explicitly: this is almost always easier to reason about properly, 
and is certainly easier to maintain. 
Consider making numeric flags into compile-time constants. 
If you encounter new flags in code reviews, push back. 
Every flag introduction should be justified.


## 112. emplace vs. push_back

```cpp
std::vector<string> my_vec;
my_vec.push_back("foo");     // This is OK, so...
my_vec.emplace_back("foo");  // This is also OK, and has the same result

std::set<string> my_set;
my_set.insert("foo");        // Same here: any insert call can be
my_set.emplace("foo");       // rewritten as an emplace call.
```
那么该用哪个push_back或者emplace_back？是不是一定要用emplace。

先看个例子：
```cpp
vec1.push_back(1<<20);
vec2.emplace_back(1<<20);
```
有啥却别？
第一个很显然，push_back一个1<<20(1048576)这个数，第二行就得看vec2的类型了，如果是std::vector<int>那么跟第一行意思一样，但如果是std::vector<std::vector<int>>呢，那就是构造一个包含100多万数值的vector了。


再者，如果碰到push_back和emplace_back都可用的情况，用push_back语意更清晰，因为push_back这个字面意思就表达清楚了行为。另外，push_back更安全，如果有个std::vector<std::vector<int>> vec，vec.push_back(2<<20);将会编译错误，而vec.emplace_back(2<<20);则会编译通过。

如果有隐式转换，emplace_back效率会比push_back高，比如，vec.push_back("foo");
将会构造一个临时string，然后再把该string move进vec。如果用vec.emplace_back("foo");那么就会在容器内直接构造，节省一次move。因此，如果对于大型对象，则可以用emplace_back替换push_back以获取更佳效率。但一般情况下，这种性能差异不重要，一条准则就是避免为了所谓的优化，让代码不安全不清晰，除非基准测试显示性能有重大提升。

结论，如果push_back和emplace_back都可以用，优先使用push_back。insert和emplace同理。


## 65. Putting Things in their Place

C++11增加一个函数emplace，它可以让对象在容器内直接构造，避免生成一个临时对象再拷贝，或移动进容器。对于几乎所有的对象，避免拷贝总是性能更优，而且让存储move-only对象进容器更方便。

The Old Way and the New Way
C++11之前：
```cpp
class Foo {
 public:
  Foo(int x, int y);
  …
};

void addFoo() {
  std::vector<Foo> v1;
  v1.push_back(Foo(1, 2));
}
```
构造了两个Foo对象，一个临时对象，一个vector里的对象，它由临时对象移动构造而成。
C++11
```cpp
void addBetterFoo() {
  std::vector<Foo> v2;
  v2.emplace_back(1, 2);
}
```
只有一个对象构造。

Using Emplace Methods for Move-Only Operations
emplace除了性能上的提升，还有一个适用场景，就是如果容器内只存移动类型，比如std::unique_ptr，这在以前几乎不可行。
```cpp
std::vector<std::unique_ptr<Foo>> v1;
```
如何插入一个元素？
```cpp
v1.push_back(std::unique_ptr<Foo>(new Foo(1, 2)));
```
语句可行，但笨重，或者：
```cpp
Foo *f2 = new Foo(1, 2);
v1.push_back(std::unique_ptr<Foo>(f2));
```
可以编译通过，但有隐患，f2指针随时被delete。
再考虑：
```cpp
std::unique_ptr<Foo> f(new Foo(1, 2));
v1.push_back(f);             // Does not compile!
v1.push_back(new Foo(1, 2)); // Does not compile!
```
编译不过，move-only。
此时，用emplace_back，让对象构造的时候就插入更直观。如果用push_back，就必须用move来做name-eraser。
```cpp
std::unique_ptr<Foo> f(new Foo(1, 2));
v1.emplace_back(new Foo(1, 2));
v1.push_back(std::move(f));
```
emplace还支持对指定迭代器位置构造
```cpp
v1.emplace(v1.begin(), new Foo(1, 2));
```
实践中，我们还是不要用这种方法来构造unique_ptr，而应该用std::make_unique。

结论：
当碰到unique_ptr时，emplace有更好的封装，并且堆对象的ownership变得更清晰。


## 99. Nonmember Interface Etiquette

C++类的interface不单单包括它的成员，它的定义。我们看一个类的API时，还要看类主体外的定义，这些定义和其公有成员一样成为interface的一部分。比如模板偏特化，hashers，traits，非成员函数operators，以及为了ADL的非成员函数，比如swap。

以下是个常见的非类成员函数设计
```cpp
namespace space {
class Key { ... };

bool operator==(const Key& a, const Key& b);
bool operator<(const Key& a, const Key& b);
void swap(Key& a, Key& b);

// standard streaming
std::ostream& operator<<(std::ostream& os, const Key& x);

// gTest printing
void PrintTo(const Key& x, std::ostream* os);

// new-style flag extension:
bool ParseFlag(const string& text, Key* dst, string* err);
string UnparseFlag(const Key& v);

}  // namespace space

HASH_NAMESPACE_BEGIN
template <>
struct hash<space::Key> {
  size_t operator()(const space::Key& x) const;
};
HASH_NAMESPACE_END
```

The Proper Namespace
作为接口的函数通常被设计来可以通过参数lookup(ADL)，Tip49。重载操作符函数以及swap类似的函数也是被设计为可通过ADL查找，只有函数定义在与参数类型关联的命名空间中，才会稳定可靠，相关的命名空间包括基类以及类模板参数的命名空间。一个常见的错误是把函数定义放在全局。
```cpp
namespace library {
struct Letter {};

void good(Letter);
}  // namespace library

// bad is improperly placed in global namespace
void bad(library::Letter);

namespace client {
void good();
void bad();

void test(const library::Letter& x) {
  good(x);  // ok: 'library::good' is found by ADL.
  bad(x);  // oops: '::bad' is hidden by 'client::bad'.
}

}  // namespace client
```
很微妙，也是重点，函数定义同时以及其参数定义一起，事情会简化。

A Quick Note on In-Class Friend Definitions
```cpp
namespace library {
class Key {
 public:
  explicit Key(string s) : s_(std::move(s)) {}
  friend bool operator<(const Key& a, const Key& b) { return a.s_ < b.s_; }
  friend bool operator==(const Key& a, const Key& b) { return a.s_ == b.s_; }
  friend void swap(Key& a, Key& b) {
    swap(a.s_, b.s_);
  }

 private:
  std::string s_;
};
}  // namespace library
```
这些friend函数特殊的属性就是只可以被ADL搜索，有点奇怪的是，这些函数虽然定义在所包围的namespace中，但却没有进行name lookup。这些函数必须有inline定义。

这些函数不会hide全局同名函数，也不会出现在对同名函数不相关调用的诊断信息上，它们定义简便，也可以访问类的内部结构。

The Proper Source Location
为了避免one-definition rule(ODR)，任何自定义的类型接口不应该被多次定义，这通常意味着它们应该与类型打包在同一个头文件中，如果在_test.cc文件中添加这些非成员的自定义是不合适的。

非成员函数的重载（包括操作符重载）应该定义在其参数所在头文件中。

模板偏特化也是如此，模板特化应该和模板定义打包一起，或者和特化的类型打包一起


## 109. Meaningful `const` in Function Declarations

本文讲述const什么时候在函数声明是有意义的，什么时候是没有必要的且可以忽略。
首先看声明和定义的区别
```cpp
void F(int);                     // 1: declaration of F(int)
void F(const int);               // 2: re-declaration of F(int)
void F(int) { /* ... */ }        // 3: definition of F(int)
void F(const int) { /* ... */ }  // 4: error: re-definition of F(int)
```
前两行是函数声明，函数声明告诉编译器函数的签名以及返回类型。上面例子中，函数的签名是F(int)，函数的参数类型constness可以被忽略，所以两个声明是等价的。
后两行是函数定义，声明可以有多个，但是定义必须只有一份。

尽管第三行和第四行声明和定义了同种函数，但是还是略微区别，第三行的函数参数是int，第四行的函数参数是const int。

Meaningful const in Function Declarations
并不是所有函数声明中的const是无意义的
const type-specifiers buried within a parameter type specification are significant and can be used to distinguish overloaded function declarations
```cpp
void F(const int* x);                  // 1
void F(const int& x);                  // 2
void F(std::unique_ptr<const int> x);  // 3
void F(int* x);                        // 4
```
以上函数都有个参数X，但就参数x，没有必要声明为const。但每个函数参数都不同，所以是一堆重载函数。前面三行的const都不会被忽略，因为它们是函数参数类型specification的一部分，而不是影响参数本身x的限定(top level)。
```cpp
void F(const int x);          // 1: declares F(int)
void F(int* const x);         // 2: declares F(int*)
void F(const int* const x);   // 3: declares F(const int*)
```
这个例子就清楚多了。

Rules of Thumb
* 不要在函数声明（不是定义）中使用top level的const限定，它是无意义的，且会被编译器忽略，视觉噪音，且会误导读者。
* 函数定义中是否用const由自己团队决定，你可以遵循与何时声明一个函数局部变量const相同的理由。


## 126. `make_unique` is the new `new`

两种unique的分配方式，std::make_unique和absl::WrapUnique（直接从一个裸指针接管ownership）。

Why Avoid new?
避免直接使用new
1. 在可能的情况下，所有权最好用类型系统来表示。这使得审查者几乎可以完全通过本地检查来验证正确性（没有泄漏和双重删除）。(在对性能特别敏感的代码中，这可能是可以原谅的：虽然很便宜，但由于ABI约束，通过值传递std::unique_ptr跨越函数边界有非零开销。这很少有重要到足以证明避免它的理由。)
2. std::make_unique语意清楚，直接表达其意图，而且只做一件事，分配后直接返回一个std::unique_ptr，没有类型转换或者隐藏行为。
3. std::unique_ptr<T> my_t(new T(args));也可以，但是多余的，写了两次T。
4. 如果所有的分配通过std::make_unique或者工厂函数，那么absl::WrapUnique适用于实现factory calls。有些为了和以前代码兼容（没有用到std::unqiue_ptr来所有权转移）或者有些动态分配需要用到aggregate initialization。(absl::WrapUnique(new MyStruct{3.141, "pi"}))。
5. 
std::unique_ptr<T> foo(Blah());
std::unique_ptr<T> bar(new T());
第二行一眼就看到比较安全，第一行要根据Blah返回，如果返回一个std::unique_ptr，也OK，如果返回的是一个raw指针，也OK，但如果返回的是一个随机指针（没有转移）那就回有问题。

How Should We Choose Which to Use?
1.
默认选用std::make_unique
```cpp
不要
std::unique_ptr<T> bar(new T());
而要
auto bar = std::make_unique<T>();
```
```cpp
不要
bar.reset(new T());
而要
bar = std:make_unique<T>();
```
2.在factory calls中使用非public构造函数，采用absl::WrapUnique并且返回一个std::unique_ptr。
3. 如果分配函数需要brace initialization，采用absl::WrapUnique
4. 如果调用一个以前的接口需要指针T*，那么可以采用std::make_unique构造，传入调用，再ptr.release(),或者在函数参数中直接使用new。
5. 如果调用一个以前的接口返回指针T*，那么应该用absl::WrapUnique接管。

结论：
Prefer使用std::make_unique，然后absl::WrapUnique，最后new。


## 119. Using-declarations and namespace aliases

```cpp
namespace example {
namespace makers {
namespace {

using ::otherlib::BazBuilder;
using ::mylib::BarFactory;
namespace abc = ::applied::bitfiddling::concepts;

// Private helper code here.

}  // namespace

// Interface implementation code here.

}  // namespace makers
}  // namespace example
```
不要把alias放在头文件中，因为这只是要方便实现，而不是把它当成exported name成为API一部分。

先上结论：
* 不要在头文件中声明namespace aliases或者convenience using-declarations。
* 应该把namespace aliases或者convenience using-declarations放在最innermost命名空间。
* 当声明namespace aliases and using-declarations时，使用qualified名字，除非引用一个当前命名空间的名字。
* 适当情况下使用qualified名字。

举个例子：
Scope of the Alias
```cpp
using ::foo::Quz;

namespace example {
namespace util {

using ::foo::Bar;
```
如果之后在全局范围声明一个Quz，那么第一行将会冲突，因为重新声明了Quz。如果在example命名空间或者example::util命名空间声明了example::Quz或example::util::Quz，那么unqualified lookup将会用这两个取代::foo::Quz。

一般地说，一个声明越接近使用点，能破坏你的代码的作用域集就越小。在我们的例子中，最糟糕的是Quz，它可以被任何人破坏；Bar只能被::example::util中的其他代码破坏，而一个在未命名命名命名空间中声明和使用的名字不能被任何其他作用域破坏。

Relative Qualifiers
```cpp
namespace example {
namespace util {

using foo::Bar;
```
也许作者的意图是使用::foo::Bar这个名字。然而，这可能会破坏，因为代码依赖于::foo::Bar的存在，也依赖于命名空间::example::foo和::example::util::foo的不存在。这种脆性可以通过完全限定使用的名称来避免：使用::foo::Bar。

只有一种情况不会
```cpp
namespace example {
namespace util {
namespace internal {

struct Params { /* ... */ };

}  // namespace internal

using internal::Params;  // OK, same as ::example::util::internal::Params
```

Demo:
```cpp
helper.h
namespace bar {
namespace foo {

// ...

}  // namespace foo
}  // namespace bar
```
```cpp
some_feature.h
extern int f;
```
```cpp
#include "helper.h"
#include "some_feature.h"

namespace foo {
void f();
}  // namespace foo

// Failure mode #1: Alias at a bad scope.
using foo::f;  // Error: redeclaration (because of "f" declared in some_feature.h)

namespace bar {

// Failure mode #2: Alias badly qualified.
using foo::f;  // Error: No "f" in namespace ::bar::foo (because that namespace was declared in helper.h)

// The recommended way, robust in the face of unrelated declarations:
using ::foo::f;  // OK

void UseCase() { f(); }

}  // namespace bar
```

Unnamed Namespaces
放在匿名命名空间中的使用声明可以从外层命名空间访问，反之亦然。如果你已经在文件的顶部有一个匿名命名空间，最好把所有的别名放在那里。在这个匿名命名空间中，你可以获得额外的一点健壮性，防止与外层命名空间中声明的东西发生冲突。
```cpp
namespace example {
namespace util {

namespace {

// Put all using-declarations in here. Don't spread them over the file.
using ::foo::Bar;
using ::foo::Quz;

// In here, Bar and Quz refer inalienably to your aliases.

}  // namespace

// Can use both Bar and Quz here too. (But don't declare any entities called Bar or Quz yourself now.)
```


## 130. Namespace Naming

关于命名空间命名，一直有所误解。

Name Lookup
```cpp
namespace foo {
namespace bar {
void f() {
  Baz b;
}
}
}
```
unqualified Baz首先搜索f()，然后bar命名空间，再foo命名空间，再全局搜索。
Java就不一样，直接qualified。
```cpp
public void f() {
  com.google.foo.bar.Baz b = new com.google.foo.bar.Baz();
}
或者
import com.google.foo.bar.Baz;
import com.google.foo.bar.*;
```
在任何情况下，Baz都不会在明确提供的包之外寻找：通配符不会降到子包中，搜索也不会扩展到父包中。

这就是C++和Java的差异。

问题：
在C++中一般用unqualified名字，如果在::division::section::team::subteam::project中使用std::unique_ptr，那么可能有以下选择：
::std::unique_ptr
::division::std::unique_ptr
::division::section::std::unique_ptr
::division::section::team::std::unique_ptr
::division::section::team::subteam::std::unique_ptr
::division::section::team::subteam::project::std::unique_ptr

那么建议：
Don’t nest deeply: a single top-level namespace per project gets the same result without long/complicated names, with less exposure to accidents, without causing surprise for new engineers, and without the need to build any tooling.


## 134. make_unique and private Constructors.

make_unique如果碰到private的构造函数就会编译不过。
```cpp
class Widget {
 public:
  static std::unique_ptr<Widget> Make() {
    return absl::make_unique<Widget>(GenerateId());
  }

 private:
  Widget(int id) : id_(id) {}
  static int GenerateId();

  int id_;
}
```
编译不过。
error: calling a private constructor of class 'Widget'
    { return unique_ptr<_Tp>(new _Tp(std::forward<_Args>(__args)...)); }
                                 ^
note: in instantiation of function template specialization
'absl::make_unique<Widget, int>' requested here
    return absl::make_unique<Widget>(GenerateId());
                ^
note: declared private here
  Widget(int id) : id_(id) {}

Make函数可以访问私有构造函数，absl::make_unique却不行。

推荐做法：
```cpp
    // Using `new` to access a non-public constructor.
    return absl::WrapUnique(new Widget(...));
```
或者直接把构造函数public。

Why Can’t I Just Friend absl::make_unique?
a good rule of thumb is “no long-distance friendships”. 否则，你就会创建一个竞争性的声明的朋友，一个不由所有者维护的声明。
再者，如果friend了absl::make_unique，那么任何人都可以创建这个对象，那为什么不直接public。

What About std::shared_ptr?
```cpp
class Widget {
  class Token {
   private:
    Token() {}
    friend Widget;
  };

 public:
  static std::shared_ptr<Widget> Make() {
    return std::make_shared<Widget>(Token{}, GenerateId());
  }

  Widget(Token, int id) : id_(id) {}

 private:
  static int GenerateId();

  int id_;
};
``` 

## 153. Don't Use using-directives

先上结论
Using-directives (using namespace foo) are dangerous enough to be banned by the Google style guide. Don’t use them in code that will ever need to be upgraded.

If you wish to shorten a name, you may instead use a namespace alias (namespace baz = ::foo::bar::baz;)
or a using-declaration (using ::foo::SomeName), 
both of which are permitted by the style guide in certain contexts (e.g. in *.cc files).

Using-directives at Function Scope
```cpp
namespace totw {
namespace example {
namespace {

TEST(MyTest, UsesUsingDirectives) {
  using namespace ::testing;
  Sequence seq;  // ::testing::Sequence
  WallTimer timer;  // ::WallTimer
  ...
}

}  // namespace
}  // namespace example
}  // namespace totw
```
一般以为using-directive只是把名字注入到声明所在的scope内，而实际上是注入::testing和::totw::example::annoymous共同的祖先namepsace里，那就是global namespace。
所以以上代码相当于
```cpp
using ::testing::Expectation;
using ::testing::Sequence;
using ::testing::UnorderedElementsAre;
...
// many, many more symbols are injected into the global namespace

namespace totw {
namespace example {
namespace {

TEST(MyTest, UsesUsingDirectives) {
  Sequence seq; // ::testing::Sequence
  WallTimer timer; // ::WallTimer
  ...
}

} // namespace
} // namespace example
} // namespace totw
```
一旦有人添加了某些代码，就会遭到破坏
* 例如有人定义::totw::Sequence或者::totw::example::Sequence，那么原先::testing::Sequence就会被取代
* 如果有人定义::Sequence，那么就会在global namespace出现两个，::testing::Sequence和::Sequence，编译不过
* 如果有人定义::testing::WallTimer，timer同样编译不过

Thus, a single using-directive in a function scope has placed naming restrictions on symbols in ::testing, ::totw, ::totw::example, and the global namespace. 
Allowing this using-directive, even if only in function scope, creates ample opportunities for name clashes in the global and other namespaces.

再举个例子：
```cpp
namespace totw {
namespace example {
namespace {

TEST(MyTest, UsesUsingDirectives) {
  using namespace ::testing;
  EXPECT_THAT(..., proto::Partially(...)); // ::testing::proto::Partially
  ...
}

} // namespace
} // namespace example
} // namespace totw
```
This using-directive has introduced a namespace alias proto in the global namespace, roughly equivalent to the following:
```cpp
namespace proto = ::testing::proto;

namespace totw {
namespace example {
namespace {

TEST(MyTest, UsesUsingDirectives) {
  EXPECT_THAT(..., proto::Partially(...)); // ::testing::proto::Partially
  ...
}

} // namespace
} // namespace example
} // namespace totw
```
如果后来有人定义::proto, ::totw::proto, or ::totw::example::proto，那么一样会编译不过。

This ties into the style guide’s rules on namespace naming: avoid nested namespaces, and don’t use common names for nested namespaces.

有人认为一个封闭的命名空间，其中只有很少的symbol，比如std::placeholders，包含符号 _1 ... _9。即便这样也不安全，它排除了其他命名空间引入相同名称的symbol。
using-directives defeat the modularity provided by namespaces.

Unqualified using-directives
```cpp
namespace totw {
namespace example {
namespace {

using namespace rpc;
using namespace testing;

TEST(MyTest, UsesUsingDirectives) {
  Sequence seq;  // ::testing::Sequence
  WallTimer timer;  // ::WallTimer
  RPC rpc;  // ...is this ::rpc::RPC or ::RPC?
  ...
}

}  // namespace
}  // namespace example
}  // namespace totw
```
* 上面所讨论的function scope Using-directives问题都存在，而且这次有两个，一个::testing，一个::rpc
* ::testing和::rpc命名空间里如果有同名冲突，编译不过
* 如果添加::rpc::testing，可能会编译不过。
* A newly-introduced symbol in ::totw::example, ::totw, ::testing, ::rpc, or the global namespace could collide with an existing symbol in any of those namespaces. That’s a big matrix of possibilities.

Why Do We Have This Feature, Then?
在通用库中，有一些using-directives的合法用法，但它们是如此的晦涩和罕见，以至于不值得在这里或在风格指南中提及。

using-directives就是个定时炸弹，今天编译好好的，改天添加了一些符号就会编译不过。


## 147. Use Exhaustive switch Statements Responsibly

如果编译选项加了-Werror，switch-case没有穷尽所有枚举，也没有提供default，那么会编译错误。
An exhaustive switch statement is an excellent construct for ensuring at compile time 
that every enumerator of a given enum is explicitly handled.

```cpp
std::string AnEnumToString(AnEnum an_enum) {
  switch (an_enum) {
    case kFoo:
      return "kFoo";
    case kBar:
      return "kBar";
    case kBaz:
      return "kBaz";
  }
}
```
如果传入一个非枚举定义的变量，也没有default处理，那将会UB

```cpp
std::string AnEnumToString(AnEnum an_enum) {
  switch (an_enum) {
    case kFoo:
      return "kFoo";
    case kBar:
      return "kBar";
    case kBaz:
      return "kBaz";
  }
  std::cerr << "Unexpected value for AnEnum: " << an_enum;
  return kUnknownAnEnumString;
}
```


## 161. Good Locals and Bad Locals

Bad Uses of Local Variables
```cpp
MyType value = SomeExpression(args);
return value;
```
prefer
```cpp
return SomeExpression(args);
```

```cpp
auto actual = SortedAges(args);
EXPECT_THAT(actual, ElementsAre(21, 42, 63));
```
prefer
```cpp
EXPECT_THAT(SortedAges(args), ElementsAre(21, 42, 63));
```

```cpp
myproto.mutable_submessage()->mutable_subsubmessage()->set_foo(21);
myproto.mutable_submessage()->mutable_subsubmessage()->set_bar(42);
myproto.mutable_submessage()->mutable_subsubmessage()->set_baz(63);
```
prefer
```cpp
auto& subsubmessage = *myproto.mutable_submessage()->mutable_subsubmessage();
subsubmessage.set_foo(21);
subsubmessage.set_bar(42);
subsubmessage.set_baz(63);
```

```cpp
for (const auto& name_and_age : ages_by_name) {
  if (IsDisallowedName(name_and_age.first)) continue;
  if (name_and_age.second < 18) children.insert(name_and_age.first);
}
```
prefer
```cpp
for (const auto& name_and_age : ages_by_name) {
  const auto& name = name_and_age.first;
  const auto& age = name_and_age.second;

  if (IsDisallowedName(name)) continue;
  if (age < 18) children.insert(name);
}
```
C++17特性
```cpp
for (const auto& [name, age] : ages_by_name) {
  if (IsDisallowedName(name)) continue;
  if (age < 18) children.insert(name);
}
```

## 168. inline Variables

```cpp
inline constexpr absl::string_view kHelloWorld = "Hello World.";
```
using inline here ensures that there is only one copy of kHelloWorld in the program.
guaranteed to be at the same memory address every time.

几乎每一个在头文件中定义的全局变量都应该被标记为inline--通常也应该是conexpr。
如果它们没有被标记为inline，那么每个包含头文件的.cc文件都会有一个单独的变量实例，
这可能会导致对ODR（一个定义规则）的subtle违反。

Outside of header files there is no need to mark variables as inline.

A static constexpr data member of a class is implicitly inline from C++17. 


## 166. When a Copy is not a Copy

```cpp
class BigExpensiveThing {
 public:
  static BigExpensiveThing Make() {
    // ...
    return BigExpensiveThing();
  }
  // ...
 private:
  BigExpensiveThing();
  std::array<OtherThing, 12345> data_;
};

BigExpensiveThing MakeAThing() {
  return BigExpensiveThing::Make();
}

void UseTheThing() {
  BigExpensiveThing thing = MakeAThing();
  // ...
}
```
C++17之前，这段代码可能有3次拷贝和move，每个return一次拷贝，构造的时候一个move。
当然可能也有些优化。比如几个地方都有个BigExpensiveThing，显然move BigExpensiveThing
到最终的位置比较高效。实际上，thing总是被构造，并没有move，得益于C++规则允许move
操作优化成直接构造。

C++17之后，this code is guaranteed to perform zero copies or moves.
BigExpensiveThing::Make直接构造了thing。

Generally, creation of an object is deferred until the object is given a name. 

All you need to know is: objects are not copied until after they are first given a name. 
There is no extra cost in returning by value.

Even after being given a name, local variables might still not be copied when returned from a function, 
due to the Named Return Value Optimization(NRVO). 

那么
Gritty Details: When Unnamed Objects Are Copied?
1 Constructing base classes: 
```cpp
class DerivedThing : public BigExpensiveThing {
 public:
  DerivedThing() : BigExpensiveThing(MakeAThing()) {}  // might copy data_
};
```
这是因为当类被用作基类时，其布局和表示方式可能有些不同（由于virtual base class和vpointer值）
所以直接初始化基类可能不会得到正确的表示方式。

2 Passing or returning small trivial objects:
```cpp
struct Strange {
  int n;
  int *p = &n;
};
void f(Strange s) {
  CHECK(s.p == &s.n);  // might fail
}
void g() { f(Strange{0}); }
```
直接通过寄存器，所以copy


## 146. Default vs Value Initialization

两个概念：
scalar type
an integral or floating point arithmetic object; a pointer; an enum; a pointer-to-member; nullptr_t.
aggregate type
an array or trivial class (one with only public, non-static data members, no user-provided constructors, 
no base classes, no virtual member functions, and no default member initializers).

*User-Provided Constructors*
```cpp
struct Foo {
  Foo() : v() {}

  int v;
  std::string s;
};

int main() {
  Foo default_foo;
  Foo value_foo = {};
  ...
}
```
用户提供构造函数，那么value- and default-initialization都好使。
The = {} triggers value-initialization of value_foo, which calls Foo’s default constructor.

1 Uninitialized Members in User-Provided Constructors
```cpp
struct Foo {
  Foo() {}

  int v;
};

int main() {
  Foo foo = {};
}
```
In this case, v is once more default-initialized, 
which means its value is undetermined, and it is unsafe to read.

2 Explicit Value-Initialization
```cpp
struct Foo {
  Foo() : v(0) {}

  int v;
};
```
This is called direct-initialization, which is a more specific form of value-initialization.


*Default Member Initialization*
```cpp
struct Foo {
  int v = 0;
};
```
一个比为类定义构造函数更简单的解决方案，同时还能避免default- vs value-initialization的陷阱，
就是尽可能在声明时初始化类的成员。

```cpp
int main() {
  int foo[3];
  int bar[3] = {};
  ...
}
```
Every element of foo is default-initialized, while every element of bar will be zero-initialized.

案例分析
```cpp
struct Foo {
  Foo() = default;

  int v;
};

struct Bar {
  Bar();

  int v;
};

Bar::Bar() = default;

int main() {
  Foo f = {};
  Bar b = {};
  ...
}
```
对于Foo的构造函数声明为default，所以这不算是user-provided函数，Foo是个aggregate type，
那么f.v is zero-initialized。
对于Bar的构造函数就属于user-provided，尽管是由编译器作为默认构造函数创建实现。
As this constructor does not explicitly initialize Bar::v, b.v will be default-initialized and unsafe to read.

Recommendations
* Be explicit about the value to which scalar types are being initialized instead of relying on zero-initialization.
* Treat all instances of scalar types as having indeterminate values until you explicitly initialize or assign to them.
* If a member has a sensible default, and the class has multiple constructors, use a default member initializer to ensure it isn’t left uninitialized.
Be aware that a member initializer within a constructor will override the default. 


## 108. Avoid std::bind

```cpp
void DoStuffAsync(std::function<void(absl::Status)> cb);

class MyClass {
  void Start() {
    DoStuffAsync(std::bind(&MyClass::OnDone, this));
  }
  void OnDone(absl::Status status);
};
```
有啥问题？
编译不过，如果用std::function<void()>则编译通过，但是多了个参数就编译不过。
*std::bind() doesn’t just bind the first N arguments*
You must instead specify every argument.

```cpp
std::bind(&MyClass::OnDone, this, std::placeholders::_1)
```
Ugh, that’s ugly. Is there a better way? Why yes, use absl::bind_front() instead.
```cpp
absl::bind_front(&MyClass::OnDone, this)
```
absl::bind_front() binds the first N arguments and perfect-forwards the rest: 
absl::bind_front(F, a, b)(x, y) evaluates to F(a, b, x, y).

更可怕的是
```cpp
void DoStuffAsync(std::function<void(absl::Status)> cb);

class MyClass {
  void Start() {
    DoStuffAsync(std::bind(&MyClass::OnDone, this));
  }
  void OnDone();  // No absl::Status here.
};
```
竟然编译通过，哪怕DoStuffAsync要求一个absl::Status参数，传入的却没有。

*std::bind() disables one of the basic compile time checks*
The compiler usually tells you if the caller is passing more arguments than you expect, but not for std::bind().

举个例子：
```cpp
void Process(std::unique_ptr<Request> req);

void ProcessAsync(std::unique_ptr<Request> req) {
  thread::DefaultQueue()->Add(
      ToCallback(std::bind(&MyClass::Process, this, std::move(req))));
}
```
编译不过，std::bind() doesn’t move the bound move-only argument to the target function. 
用absl::bind_front()替换可以编译通过。

再个例子：
```cpp
// F must be callable without arguments.
template <class F>
void DoStuffAsync(F cb) {
  auto DoStuffAndNotify = [](F cb) {
    DoStuff();
    cb();
  };
  thread::DefaultQueue()->Schedule(std::bind(DoStuffAndNotify, cb));
}

class MyClass {
  void Start() {
    DoStuffAsync(std::bind(&MyClass::OnDone, this));
  }
  void OnDone();
};
```
编译不过，*passing the result of std::bind() to another std::bind() is a special case.*
一般来说，std::bind(F, arg)()会解析为F(arg)，除非arg是另一个std::bind调用的结果，那样情况下
会解析为F(arg())，此时arg若是std::function<void()>，the magic behavior is lost.

*Applying std::bind() to a type you don’t control is always a bug.*
DoStuffAsync() shouldn’t apply std::bind() to the template argument. 
Either absl::bind_front() or lambda would work fine.

如果你通过编写嵌套的 std::bind() 调用来组成函数，你真的应该写一个 lambda 或一个命名函数来代替。

再看一个std::bind() used correctly的例子：
```cpp
std::bind(&MyClass::OnDone, this)
```
vs
```cpp
[this]() { OnDone(); }
```

```cpp
std::bind(&MyClass::OnDone, this, std::placeholders::_1)
```
vs
```cpp
absl::bind_front(&MyClass::OnDone, this)
```

以上讨论涵盖了99%的std::bind()用法，但是还有一些是std::bind()的用处之地。
* Ignore some of the arguments: std::bind(F, \_2).
* Use the same argument more than once: std::bind(F, \_1, \_1).
* Bind arguments at the end: std::bind(F, \_1, 42).
* Change argument order: std::bind(F, \_2, \_1).
* Use function composition: std::bind(F, std::bind(G)).

Conclusion
Avoid std::bind. Use a lambda or absl::bind_front instead.


## 163. Passing absl::optional parameters

The problem
如果要实现一个函数，函数的参数是否存在不一定，一般选择std::optional
```cpp
void MyFunc(const absl::optional<Foo>& foo);  // Copies by value
void MyFunc(absl::optional<const Foo&> foo);  // Doesn't compile
```

如果有人传入Foo给MyFunc，Foo将被拷贝进std::optional<Foo>，然后再传这个optional的引用给MyFunc，如果你的目标是避免拷贝Foo，显然没有。
第二个语句absl::optional不支持。

Recommendation
If you need a parameter that might not exist, pass it by const * and let nullptr indicate “does not exist.”

```cpp
void MyFunc(const Foo* foo);
```
跟传const reference一样高效，同时支持null ptr判断。
同样可以用std::reference_wrapper包一层来实现。

```cpp
void MyFunc(absl::optional<std::reference_wrapper<const Foo>> foo);
```
然而，这有相当多的模板，而且很难阅读。因此，我们不推荐使用。


那么问题来了
Then what on Earth is absl::optional for???
absl::optional can be used if you own the thing that’s optional.
For example, class members and function return values often work well with absl::optional. 
类成员以及函数返回值

Exceptions例外
如果对象足够小，不需要传引用，那么直接传值optional

```cpp
void MyFunc(absl::optional<int> bar);
```

如果你希望你的函数的所有调用者都已经在absl::optionl中拥有一个对象，那么你可以使用const absl::optionl&。然而，这种情况很少；
通常只有当你的函数在你自己的文件/库中是私有的时候才会发生。


## 171. Avoid Sentinel Values

举个例子：

```cpp
// Returns the account balance, or -5 if the account has been closed.
int AccountBalance();
```
那么函数返回值判断是判断负值，还是判断-5，假如后来系统支持余额为负值，那么原先表示账户关闭的负值如何调整？

检测语句
```cpp
int balance = AccountBalance();
if (balance == -5) {
  std::cerr << "account closed";
  return;
}
// use `balance` here
```
放宽点
```cpp
int balance = AccountBalance();
if (balance <= 0) {
  std::cerr << "where is my account?";
  return;
}
// use `balance` here
```
不检测
```cpp
int balance = AccountBalance();
// use `balance` here
```

Problems with Sentinel Values
* Different systems may use different sentinel values
* Sentinel values limit interface evolution, as the specific sentinel may someday be a valid value for use in that system.
* One system’s sentinel value is another’s valid value, increasing cognitive overhead and code complexity when interfacing with multiple systems.
* The sentinel values are still part of the type’s domain of valid values, so neither the caller nor the callee is forced by the type system to acknowledge that a value may be invalid. When code and comments disagree, both are usually wrong.

忘记检查指定的sentinel values是一个常见的错误。
在最好的情况下，使用一个未被检查的sentinel values会在运行时立即使系统崩溃。
更常见的情况是，一个未检查的sentinel values可能会继续在系统中传播，产生不好的结果。

Use absl::optional Instead
```cpp
// Returns the account balance, or absl::nullopt if the account has been closed.
absl::optional<int> AccountBalance();
```

optional派上用场
```cpp
absl::optional<int> balance = AccountBalance();

if (!balance.has_value()) {
  std::cerr << "Account doesn't exist";
  return;
}
// use `*balance` here
```

## 172. Designated Initializers

Designated initializers are a syntax in the draft C++20 standard for specifying the contents of a struct in a compact yet readable and maintainable manner. 

过去
```cpp
struct Point {
  double x;
  double y;
  double z;
};

Point point;
point.x = 3.0;
point.y = 4.0;
point.z = 5.0;
```
将来
```cpp
Point point = {
    .x = 3.0,
    .y = 4.0,
    .z = 5.0,
};
```

```cpp
// Make it clear to the reader (of the potentially complicated larger piece of
// code) that this struct will never change.
const Point character_position = { .x = 3.0 };
```

```cpp
std::vector<Point> points;
[...]
points.push_back(Point{.x = 3.0, .y = 3.0});
points.push_back(Point{.x = 4.0, .y = 4.0});
```

Semantics
Designated initializers are a form of aggregate initialization, and so can be used only with aggregates. 
适用于 “structs or classes with no user-provided constructors or virtual functions”

```cpp
Point point = {
    .x = 1.0,
    // y will be 0.0
    .z = 2.0,
};
```

这是一个aggregate initialization
那么除了指定x和z的值，y的值为0，
那么问题来了，除了指定成员初始化外，未赋值的字段default值是啥？

答案是：
* If the struct definition contains a default member initializer
 (i.e. the field definition looks like std::string foo = "default value";) then that is used.
* Otherwise the field is initialized as if with = {}. 
In practice this means that for plain old data types you get the zero value,
for more complicated classes you get a default-constructed instance.

Some History and Language Trivia
一些历史和语言小知识

Designated initializers have been a standard part of the C language since C99,
and have been offered by compilers as a non-standard extension since before that.
自C99以来，Designated initializers一直是C语言的标准部分。
并在此之前就已经被编译器作为非标准扩展提供。

But until recently they were not part of C++: a notable example where C is not a subset of C++. 
For this reason the Google style guide used to say not to use them.
但直到最近，它们还不是C++的一部分：所以这也是一个显著的例子来证明C不是C++的子集。

After two decades the situation has finally changed: designated initializers are now part of the draft C++20 standard.
二十年后，情况终于发生了变化：Designated initializers现在是C++20标准草案的一部分。

The C++20 form of designated initializers has some restrictions compared to the C version:
* C++20 requires fields to be listed in the designator in the same order as they are listed in the struct definition (so Point{.y = 1.0, .x = 2.0} is not legal). C does not require this.
* C allows you to mix designated and non-designated initializers (Point{1.0, .z = 2.0}), but C++20 does not.
* C supports a syntax for sparsely initializing arrays known as “array designators”. This is not part of C++20.


## 173. Wrapping Arguments in Option Structs

Designated Initializers(C++20)
Designated initializers make using option structs easier and safer since we can construct the options object in the call to the function.
这使得代码更短更安全，避免了传递option structs的临时对象生命期问题。

先上个使用案例，再分析存在问题
```cpp
struct PrintDoubleOptions {
  absl::string_view prefix = "";
  int precision = 8;
  char thousands_separator = ',';
  char decimal_separator = '.';
  bool scientific = false;
};

void PrintDouble(double value,
                 const PrintDoubleOptions& options = PrintDoubleOptions{});
std::string name = "my_value";
PrintDouble(5.0, {.prefix = absl::StrCat(name, "="), .scientific = true});
```

The Problem of Passing Many Arguments
一个函数设计如果需要传入很多参数，那么会confusing
```cpp
void PrintDouble(double value, absl::string_view prefix,  int precision,
                 char thousands_separator, char decimal_separator,
                 bool scientific);
```
调用
```cpp
PrintDouble(5.0, "my_value=", 2, ',', '.', false);
```
it is hard to read this code and know to which parameter each argument corresponds.
如果参数位置不小心调换就bug，比如
```cpp
PrintDouble(5.0, "my_value=", ',', '.', 2, false);
```

那么过去的作法是：
```cpp
PrintDouble(5.0, "my_value=",
            /*precision=*/2,
            /*thousands_separator=*/',',
            /*decimal_separator=*/'.',
            /*scientific=*/false);
```
另外这个函数可能设计为有很多默认值
```cpp
void PrintDouble(double value, absl::string_view prefix = "", int precision = 8,
                 char thousands_separator = ',', char decimal_separator = '.',
                 bool scientific = false);
```
这段代码更不好阅读
```cpp
PrintDouble(5.0, "my_value=");
```
再把注释加上
```cpp
PrintDouble(5.0, "my_value=",
            /*precision=*/8,              // unchanged from default
            /*thousands_separator=*/',',  // unchanged from default
            /*decimal_separator=*/'.',    // unchanged from default
            /*scientific=*/true);
```

解决，用一个option struct包起来传递参数
```cpp
struct PrintDoubleOptions {
  absl::string_view prefix = "";
  int precision = 8;
  char thousands_separator = ',';
  char decimal_separator = '.';
  bool scientific = false;
};

void PrintDouble(double value,
                 const PrintDoubleOptions& options = PrintDoubleOptions{});
```
函数调用
```cpp
PrintDoubleOptions options;
options.prefix = "my_value=";
PrintDouble(5.0, options);
```
不过这个解决方案也有一些问题。

先看，如果PrintDouble调用的是最初的那个接口
```cpp
void PrintDouble(double value, absl::string_view prefix,  int precision,
                 char thousands_separator, char decimal_separator,
                 bool scientific);
```
```cpp
std::string name = "my_value";
PrintDouble(5.0, absl::StrCat(name, "="));
```
absl::StrCat构造一个临时string，然后被一个string_view绑定，这个临时string的生命期
贯穿整个函数调用，所以这个string_view prefix是安全的。

再看看用传递option struct方式
```cpp
std::string name = "my_value";
PrintDoubleOptions options;
options.prefix = absl::StrCat(name, "=");
PrintDouble(5.0, options);
```
options.prefix的类型是string_view，绑定的是absl::Str临时string，到了PrintDouble该string的生命期就结束了，
所以留下了dangling string_view。

有两种方式来修复：
1. 把string_view改为string，效率降低
2. add setter member function，代码难看
```cpp
class PrintDoubleOptions {
 public:
  PrintDoubleOptions& set_prefix(absl::string_view prefix) {
    prefix_ = prefix;
    return *this;
  }

  absl::string_view prefix() const { return prefix_; }

  // Setters and getters for the other member variables.

 private:
  absl::string_view prefix_ = "";
  int precision_ = 8;
  char thousands_separator_ = ',';
  char decimal_separator_ = '.';
  bool scientific_ = false;
};
```
这样就可以安全调用
```cpp
std::string name = "my_value";
PrintDouble(5.0, PrintDoubleOptions{}.set_prefix(absl::StrCat(name, "=")));
```

更好的方案是使用designated initializers 
```cpp
class DoublePrinter {
  explicit DoublePrinter(const PrintDoubleOptions& options);
  static std::unique_ptr<DoublePrinter> Make(const PrintDoubleOptions& options);
  ...
};

auto printer1 = absl::make_unique<DoublePrinter>(
    PrintDoubleOptions({.scientific=true});
auto printer2 = DoublePrinter::Make({.scientific=true});
```

Conclusions
* 对于那些可能会被调用者混淆的多个参数的函数，
或者你想指定默认参数而不必担心顺序，
强烈考虑使用option struct来增加便利性和代码的清晰度。
* 当调用采用option struct的函数时，使用designated initializers可以使代码更短，并有可能避免临时的寿命问题。
* Designated initializers由于其简洁明了的特点，进一步使人们倾向于使用option struct的函数，
而不是有很多参数的函数。


## 175. Changes to Literal Constants in C++14 and C++17.

C++ now has some features that can make numeric literals more readable.

42有几种写法
```cpp
binary (0b00101010) 
decimal (42)
hex (0x2A)
octal (052)
```

单引号可以用来数字分隔符，比如(0xDEAD'C0DE)
二进制 0b1110'0000
1'000'000'000用来替代1000000000
单引号都没有啥限定条件
0b1'001'0001乱标单引号，最终的结果也还是145

0x2Ap12 表示 0x2A << 12(0x2A000)
0x1p-10 表示 1.0/1024


## 176. Prefer Return Values to Output Parameters

*The problem*
```cpp
// Extracts the foo spec and the bar spec from the provided doodad.
// Returns false if the input is invalid.
bool ExtractSpecs(Doodad doodad, FooSpec* foo_spec, BarSpec* bar_spec);
```
* foo_spec和bar_spec哪个是输入输出
* What happens to pre-existing data in foo_spec or bar_spec? 
Is it appended to? 
Is it overwritten? 
Does it make the function CHECK-fail? 
Does it make it return false? 
Is it undefined behavior?
* Can foo_spec be null? 
Can bar_spec? 
If they cannot, does a null pointer make the function CHECK-fail? 
Does it make it return false? 
Is it undefined behavior?
* What are the lifetime requirements on foo_spec and bar_spec? 
In other words, do they need to outlive the function call?
* If false is returned, what happens to foo_spec and bar_spec? 
Are they guaranteed to be unchanged? 
Are they “reset” in some way? 
Is it unspecified?

*The solution*
```cpp
struct ExtractSpecsResult {
  FooSpec foo_spec;
  BarSpec bar_spec;
};
// Extracts the foo spec and the bar spec from the provided doodad.
// Returns nullopt if the input is invalid.
absl::optional<ExtractSpecsResult> ExtractSpecs(Doodad doodad);
```
好处：
* It is clearer what the inputs and outputs are.
* There are no questions about pre-existing data in foo_spec and bar_spec because they are created from scratch by the function.
* There are no questions about null pointers because there are no pointers.
* There are no questions about lifetimes because everything is passed and returned by value.
* There are no questions about what happens to foo_spec and bar_spec in case of failure because they cannot even be accessed if nullopt is returned.
另外这样写，也方便把结果传递给其他函数
```cpp
SomeFunction(ExtractSpecs(...)).
```

Recommendations
* 尽可能选择返回值而不是输出参数。style guide也是这样建议的
* 使用像absl::optional这样的通用封装器来表示缺失的返回值。
考虑返回absl::variant如果你需要一个更灵活的表示方式。
* 使用一个struct包装很多个返回值
构造一个新的struct返回值是合理的
不要用std::pair std::tuple来替换


## 177. Assignability vs. Data Member Types

在实现一个类型时，先决定类型设计。
优先考虑API而不是实现细节。
一个常见的例子是类型的assignability与数据成员的限定符之间的权衡。

Deciding how to represent data members
想象一下，你正在编写一个City类，并讨论如何表示其成员变量。
你知道它是短暂的，代表城市作为时间的snapshot，所以像人口、名称和市长这样的东西可以想象是const。
```cpp
 private:
  const std::string city_name_;
  const Person mayor_;
  const int64 population_;
```
Why or why not?
"是的，让这些const "的常见建议是基于这样的想法："好吧，这些值对于一个给定的城市来说是不会改变的，
所以既然所有可以常量的东西都应该是常量，那就让它们成为常量。" 
这将使该类的维护者更容易避免意外修改这些字段。

这忽略了一个至关重要的问题：City是什么样的类型？这是一个值吗？
还是一个业务逻辑的捆绑？是希望它是可复制的，只可移动的，还是不可复制的？
你可以为City（作为一个整体）高效编写的set of operations可能会受到是否将单个成员做成const的问题的影响，
而这往往是一个糟糕的权衡。

具体来说，如果你的类有const成员，就不能对其进行赋值assigned to（无论是通过复制赋值还是移动赋值）。
语言理解这一点：如果你的类型有const成员，copy-assignment and move-assignment将不会被合成。
你仍然可以复制（或移动）构造这样一个对象，但你不能在构造后以任何方式改变它（甚至 "只是 "从另一个相同类型的对象中复制）。
即使你写了自己的赋值运算符，你也会很快发现，你（显然）不能覆盖这些const成员。

因此，问题有可能变成 "我们应该倾向于哪一种：const成员和赋值操作？" 
然而，即使是这样也是误导，因为这两个问题都由一个重要的问题来回答，
"City是什么样的类型？" 如果它打算成为一个值类型，那就规定了API(包括赋值操作)，而API在一般情况下胜过了实现方面的关注。

重要的是，这些API设计决策要优先于实现细节的选择：在一般情况下，受类型的API影响的工程师要比类型的实现多。
也就是说，一个类型的用户比该类型的维护者更多，所以应该优先考虑影响用户的设计选择，而不是影响实现者。
即使你认为该类型永远不会被维护它的团队以外的人使用，软件工程也是关于接口设计和抽象的，我们应该优先考虑好的接口。

Reference Members
同样的道理也适用于设计数据成员为引用。即使我们知道成员必须是非空的，通常还是更倾向于将T\*存储为值类型，因为引用是不可重新绑定的。
也就是说，我们不能重新指向一个T&--对这种成员的任何修改都是在修改underlying的T。

考虑一下std::vector<T>的实现。在任何 std::vector 的实现中，几乎肯定会有一个指向分配的 T\* 数据成员。
我们从 std::vector 的规范中知道，这样的分配通常必须是有效的（可能除了空vectors）。
一个总是有分配的实现可以让这个T&，对吗？(这里忽略了数组和偏移量）。

显然不是。std::vector是一个值类型，它是可以复制和分配的。如果分配的存储方式是引用到第一个成员，
而不是指针到第一个成员，我们就不能move-assign存储，而且不清楚正常resize时如何更新数据。
我们巧妙地告诉其他维护者 "这个值是非空的"，会妨碍我们为用户提供所需的API。希望能清楚这是个错误的权衡。

Non-copyable / assignable types
当然，如果你对类型设计的选择表明，City（或者你所考虑的任何类型）应该是不可复制的，那对你的实现的约束就少多了。
一个类持有const或引用成员并没有对错之分，只有当这些实现决定制约或破坏了该类所呈现的接口时，才会引起关注。
如果你已经深思熟虑并有意识地决定你的类型不需要可复制，那么你对如何表示类的数据成员做出不同的选择是非常合理的。

The Unusual Case: immutable types
有一种有用但不常见的设计可能会要求使用const成员：不可变类型。
这种类型的实例在构造后是不可改变的：没有mutating方法，没有赋值操作符。
这种类型相当罕见，但有时也很有用。特别是，这样的类型本质上是线程安全的，因为没有mutating操作。
这种类型的对象可以在线程之间自由共享，而不必担心data race或同步。
然而，作为交换，这些对象可能会有很大的运行时开销，源于需要不断复制它们。
不可变性甚至会使这些对象无法被有效地move。

设计你的类型时，几乎总是希望它是mutable的，但仍然是线程兼容的，
而不是依赖线程安全--通过不可变性。你的类型的用户通常能够更好地判断可变性的好处。
在没有非常有力的证据证明你的用例为何不寻常的情况下，不要强迫他们绕过不寻常的设计选择。

Recommendations
* 在考虑实现细节之前，先决定你的类型的设计。
* value type是常见的，推荐使用。业务逻辑类型也是如此，它们通常是non-copyable的。
* immutable的类型有时是有用的，但它们是合理的情况相当罕见。
* 优先考虑API设计和用户的需求，而不是维护者（通常是较小的）关注的问题。
* 在构建value类型或move-only类型时，避免使用const和引用数据成员。


## 5. Disappearing Act

先上错误代码
```cpp
// DON’T DO THIS
std::string s1, s2;
...
const char* p1 = (s1 + s2).c_str();             // Avoid!
const char* p2 = absl::StrCat(s1, s2).c_str();  // Avoid!
```
指向了两个临时对象

Yikes! How can you avoid this kind of problem?
Option 1: Finish Using the Temporary Object Before the End of the full-expression:
```cpp
// Safe (albeit a silly example):
size_t len1 = strlen((s1 + s2).c_str());
size_t len2 = strlen(absl::StrCat(s1, s2).c_str());
```
Option 2: Store the Temporary Object.
```cpp
// Safe (and more efficient than you might think):
std::string tmp_1 = s1 + s2;
std::string tmp_2 = absl::StrCat(s1, s2);
// tmp_1.c_str() and tmp_2.c_str() are safe.
```
Option 3: Store a Reference to the Temporary Object.
```cpp
// Equally safe:
const std::string& tmp_1 = s1 + s2;
const std::string& tmp_2 = absl::StrCat(s1, s2);
// tmp_1.c_str() and tmp_2.c_str() are safe.
// The following behavior is dangerously subtle:
// If the compiler can see you’re storing a reference to a
// temporary object’s internals, it will keep the whole
// temporary object alive.
// struct Person { string name; ... }
// GeneratePerson() returns an object; GeneratePerson().name
// is clearly a sub-object:
const std::string& person_name = GeneratePerson().name; // safe
// If the compiler can’t tell, you’re at risk.
// class DiceSeries_DiceRoll { `const string&` nickname() ... }
// GenerateDiceRoll() returns an object; the compiler can’t tell
// if GenerateDiceRoll().nickname() is a subobject.
// The following may store a dangling reference:
const std::string& nickname = GenerateDiceRoll().nickname(); // BAD!
```
Option 4: Design your functions so they don’t return objects???
很多函数都遵循这个原则；但也有很多函数没有。有时候，返回一个对象确实比要求调用者传递一个输出参数的指针更好。
任何返回指向对象内部的指针或引用的东西，在对临时对象进行操作时都有可能出现问题。 
c_str()是最明显的罪魁祸首，但protobuf getters (mutable or otherwise) and getters 同样会出现问题。


## 140. Constants: Safe Idioms
a constant is a variable or function that always evaluates to the same value.
通常最简单的方法是声明一个const或constexpr变量，如果是在头文件中，则标记为inline。
另一种方法是从函数中返回一个值，这种方法更灵活。 

*Constants in Header Files*
All of the idioms in this section are robust and recommendable.

An inline constexpr Variable
From C++17 variables can be marked as inline, ensuring that there is only a single copy of the variable. 
当与constexpr一起使用以确保安全的初始化和销毁时，这提供了另一种定义常量的方法，其值在编译时可以访问。
```cpp
// in foo.h
inline constexpr int kMyNumber = 42;
inline constexpr absl::string_view kMyString = "Hello";
```

An extern const Variable
```cpp
// Declared in foo.h
ABSL_CONST_INIT extern const int kMyNumber;
ABSL_CONST_INIT extern const char kMyString[];
ABSL_CONST_INIT extern const absl::string_view kMyStringView;
```
上面的例子声明了每个对象的一个实例。extern关键字确保了外部链接。
const关键字有助于防止值的意外修改。
这是一种很好的方式，尽管这意味着编译器不能 "看到 "常量值。
这在一定程度上限制了它们的实用性，但对于典型的用例来说并不重要。
它还需要在相关的.cc文件中定义变量。
```cpp
// Defined in foo.cc
const int kMyNumber = 42;
const char kMyString[] = "Hello";
const absl::string_view kMyStringView = "Hello";
```
ABSL_CONST_INIT 宏确保每个常量在编译时被初始化，但这就是它的全部作用。
它并不能使变量成为const，也不能阻止违反style guide的带有non-trivial的析构的变量声明。

你可能很想在.cc文件中用constexpr定义变量，但这不是目前可移植的方法。

NOTE:
absl::string_view是一个声明字符串常量的好方法。
该类型有一个constexpr构造函数和一个trivial的析构，所以将它们声明为全局变量是安全的。
因为string_view知道它的长度，所以使用它们不需要运行时调用strlen()。

A constexpr Function
一个不接受参数的constexpr函数总是返回相同的值，所以它的功能是一个常量，并且通常可以在编译时用来初始化其他常量。
因为所有的constexpr函数都是隐式inline的，所以不存在链接问题。
这种方法的主要缺点是对constexpr函数中的代码进行了限制。
其次，constexpr是API contract的一个non-trivial aspect，它有实际的后果。
```cpp
// in foo.h
constexpr int MyNumber() { return 42; }
```

An Ordinary Function
当constexpr函数不可取或不可能时，普通函数可能是一种选择。下面例子中的函数不能是constexpr函数，因为它有一个静态变量。
```cpp
inline absl::string_view MyString() {
  static constexpr char kHello[] = "Hello";
  return kHello;
}
```

A static Class Member
```cpp
// Declared in foo.h
class Foo {
 public:
  static constexpr int kMyNumber = 42;
  static constexpr char kMyHello[] = "Hello";
};
```
在C++17之前，有必要在.cc文件中为这些静态数据成员提供定义，但是对于既是静态又是conexpr的数据成员来说，这些定义现在已经没有必要了（并且已经被废弃）
```cpp
// Defined in foo.cc, prior to C++17.
constexpr int Foo::kMyNumber;
constexpr char Foo::kMyHello[];
```

Discouraged Alternatives
```cpp
#define WHATEVER_VALUE 42
```
prefer
```cpp
enum : int { kMyNumber = 42 };
```

*Approaches that Work in Source Files*
上面描述的所有方法在一个.cc文件中也能行，但没必要搞复杂。
因为在源文件中声明的常量默认情况下只有在该文件中才可见(见内部链接规则)，因此较简单的方法，如定义constexpr变量，通常是可行的。
```cpp
// within a .cc file!
constexpr int kBufferSize = 42;
constexpr char kBufferName[] = "example";
constexpr absl::string_view kOtherBufferName = "other example";
```
Within a Header File, Beware!
除非你小心使用上面所讲的idiom，否则const和constexpr对象在每个翻译单元中很可能是不同的对象。

这意味着：
Bug:任何使用常量地址的代码都会出现错误，甚至会出现可怕的 "未定义行为"。
Bloat:包括你的头文件在内的每个翻译单元都会得到它自己的副本。
对于简单的类型，如原始的数字类型，没啥问题。
但对于字符串和更大的数据结构来说就不是那么好了。

当处于命名空间范围时（即不在函数或类中），const和constexpr对象都隐含着internal linkage（与未命名空间变量和不在函数或类中的静态变量使用的链接相同）。
C++标准保证每一个使用或引用该对象的翻译单元都会得到该对象的不同 "拷贝 "或 "实例化"，每一个都在不同的地址。

在一个类中，你必须额外地将这些对象声明为static的，否则它们将是不可改变的实例变量，而不是类的每个实例共享的不可改变的类变量。

同样，在一个函数中，你必须将这些对象声明为static的，否则它们将占用堆栈的空间，并在每次调用函数时被构造。

An Example Bug
```cpp
// Declared in do_something.h
constexpr char kSpecial[] = "special";

// Does something.  Pass kSpecial and it will do something special.
void DoSomething(const char* value);
// Defined in do_something.cc
void DoSomething(const char* value) {
  // Treat pointer equality to kSpecial as a sentinel.
  if (value == kSpecial) {
    // do something special
  } else {
    // do something boring
  }
}
```
请注意，这段代码将kSpecial中第一个char的地址与value进行比较，作为函数的一种magic value。
你有时会看到代码这样做是为了short circcuit一个完整的字符串比较。

这导致了一个微妙的错误。kSpecial数组是constexpr，这意味着它是static的（有internal linkage）。
尽管我们认为 kSpecial 是 "一个常量"--但它其实不是--它是一个完整的常量系列，每个翻译单元一个！
对DoSomething(kSpecial)的调用看起来应该做同样的事情，但函数根据调用发生的位置采取不同的代码路径。

任何使用头文件中定义的常量数组的代码，或者取头文件中定义的常量的地址的代码，都足以构成这类bug。
这类bug通常出现在字符串常量上，因为它们是在头文件中定义数组的最常见原因。

An Example of Undefined Behavior
调整上面的例子，并将DoSomething作为一个inline函数移到头文件中。
bingo：现在我们已经有了未定义的行为，或者说UB。
语言要求所有的inline函数在每一个翻译单元（源文件）中都要以完全相同的方式定义---这是语言的 "一个定义规则 "的一部分。
这个特定的DoSomething的实现引用了一个静态变量，所以每个翻译单元实际上都以不同的方式定义了DoSomething，因此出现了未定义的行为。

*Other Common Mistakes*

Mistake #1: the Non-Constant Constant
```cpp
const char* kStr = ...;
const Thing* kFoo = ...;
```
prefer
```cpp
// Corrected.
const Thing* const kFoo = ...;
// This works too.
constexpr const Thing* kFoo = ...;
```

Mistake #2: the Non-Constant MyString()
```cpp
inline absl::string_view MyString() {
  return "Hello";  // may return a different value with every call
}
```
字符串常量的地址是允许每次被评估时改变的，所以上面的内容是微妙的错误，因为它返回的string_view在每次调用时可能有不同的.data()值。.
虽然在很多情况下，这不会是一个问题，但它可能会导致上面描述的错误。

把MyString()改成constexpr并不能解决这个问题，因为语言标准并没有说这样做。
有一种看法是，constexpr函数只是一个inline函数，在编译时初始化常量值时允许执行。而在在运行时，它和内联函数没有什么不同。
改成这样也是错误的
```cpp
constexpr absl::string_view MyString() {
  return "Hello";  // may return a different value with every call
}
```
除非改为
```cpp
inline absl::string_view MyString() {
  static constexpr char kHello[] = "Hello";
  return kHello;
}
```
经验法则：如果你的 "常量 "是一个数组类型，在返回它之前，先把它存储在一个函数局部静态中。
这样可以固定它的地址。

Mistake #3: Improperly Initialized Constants
1 Zero initialization. 
```cpp
const int kZero;  // this will be zero-initialized to 0
const int kLotsOfZeroes[5000];  // so will all of these
```
2 Constant initialization.
```cpp
const int kZero = 0;  // this will be constant-initialized to 0
const int kOne = 1;   // this will be constant-initialized to 1
```
3 Dynamic initialization
```cpp
// This will be dynamically initialized at run-time to
// whatever ArbitraryFunction returns.
const int kArbitrary = ArbitraryFunction();
```


## 181. Accessing the value of a StatusOr<T>

当需要访问absl::StatusOr<T>对象内部的值时，我们应该努力使这种访问安全、清晰、高效。

*Recommendation*
```cpp
// The same pattern used when handling a unique_ptr...
std::unique_ptr<Foo> foo = TryAllocateFoo();
if (foo != nullptr) {
  foo->DoBar();  // use the value object
}

// ...or an optional value...
absl::optional<Foo> foo = MaybeFindFoo();
if (foo.has_value()) {
  foo->DoBar();
}

// ...is also ideal for handling a StatusOr.
absl::StatusOr<Foo> foo = TryCreateFoo();
if (foo.ok()) {
  foo->DoBar();
}
```

```cpp
if (absl::StatusOr<Foo> foo = TryCreateFoo(); foo.ok()) {
  foo->DoBar();
}
```

*Background on StatusOr*
absl::StatusOr<T>类是一个带标签的联合体，其值语义正好表示以下情况之一:
* an object of type T is available,
* an absl::Status error (!ok()) indicating why the value is not present.

Safety, Clarity, and Efficiency
像对待智能指针一样对待StatusOr对象，可以帮助代码实现清晰，同时保持安全和高效。下面，我们将考虑一些您可能看到的访问StatusOr的其他方式，以及为什么我们更喜欢使用间接操作符的方法。

Alternative Value Accessor Safety Issues
```cpp
absl::StatusOr<Foo> foo = TryCreateFoo();
foo.value();  // Behavior depends on the build mode.
```
absl::StatusOr<T>::value()是多少？
在这里，行为取决于build模式--特别是，代码是否是在启用异常的情况下编译的，因此，读者不清楚错误状态是否会终止程序。

value()方法结合了两个操作：一个是有效性测试，另一个是对值的访问。
因此，只有在两个动作都要使用的情况下，才应该使用它（即使如此，也要三思而后行，并考虑到它的行为取决于构建模式）。
如果已经知道状态是确定的，那么你理想的访问器的语义就是简单地访问值，这正是 operator* 和 operator-> 所提供的。
除了让代码更精确地表明你的意图之外，访问的效率至少会和value()的契约一样，先测试有效性，然后再访问值。

Avoiding Multiple Names for the Same Object
错误使用
```cpp
// Without digging up the declaration of TryCreateFoo(), a reader will not
// immediately understand the types here (optional? pointer? StatusOr?).
auto maybe_foo = TryCreateFoo();
// ...compounded by the use of implicit bool rather than `.ok()`.
if (!maybe_foo) { /* handle foo not present */ }
// Now two variables (maybe_foo, foo) represent the same value.
Foo& foo = maybe_foo.value();
```

Avoiding the _or Suffix
在检查有效性后使用StatusOr变量的内在值类型（而不是为同一个值创建多个变量）的另一个好处是，我们可以为StatusOr使用最好的名称，而不需要添加前缀或后缀。
比如，不需要加前缀：
```cpp
// The type already describes that this is a unique_ptr; `foo` would be fine.
std::unique_ptr<Foo> foo_ptr;

absl::StatusOr<Foo> foo_or = MaybeFoo();
if (foo_or.ok()) {
  const Foo& foo = foo_or.value();
  foo.DoBar();
}
```
而直接用
```cpp
absl::StatusOr<Foo> foo = MaybeFoo();
if (foo.ok()) {
  MakeUseOf(*foo);
  foo->DoBar();
}
```

*Solution*
Testing the absl::StatusOr object for validity (as you would a smart pointer or optional) and accessing it using operator* or operator-> is readable, efficient, and safe.


## 76. Use absl::Status

有些人对何时以及如何使用absl::Status有疑问，所以这里有几个为什么你应该使用Status的原因，以及使用它时需要注意的一些事情。

Communicate Intent and Force the Caller to Handle Errors
使用Status强制调用者处理错误的可能性。自2013年6月起，不能简单地忽略一个返回Status对象的函数。
也就是说，这段代码会产生一个编译错误。
```cpp
absl::Status Foo();

void CallFoo1() {
  Foo();
}
```
以下为正确代码
```cpp
void CallFoo2() {
  Foo().IgnoreError();
}

void CallFoo3() {
  if (!status.ok()) std::abort();
}

void CallFoo4() {
  absl::Status status = Foo();
  if (!status.ok()) std::cerr << status;
}
```

Allow the Caller to Handle Errors Where They Have More Context
当你的代码中不清楚如何处理错误时，就使用Status。
instead的是返回一个Status，让调用者处理这个错误，因为他可能有更合适的见解。

例如，在本地进行日志记录可能会影响性能，例如在编写基础架构代码时。
如果你的代码是在一个紧密的循环中调用的，即使是调用std::cout也可能太expensive了。
在其他情况下，用户可能并不真正关心一个调用是否成功，并发现日志的垃圾信息很有干扰性。

日志记录在很多情况下是合适的，但是返回Status的函数不需要LOG来解释为什么会失败：它们可以返回失败代码和错误字符串，让调用者决定正确的错误处理响应应该是什么。

Isn’t This Just Re-Inventing Exceptions?
Google style guide中著名的禁止异常（它比其他任何禁令讨论得更多）。
这很容易让人把absl::Status看成是一个poor-man's异常机制，但有更多的开销。
虽然表面上可能有相似之处，但absl::Status的不同之处在于它需要被显式处理，而不是作为一个未处理的异常隐蔽地传递到堆栈中。
它迫使工程师决定如何处理错误，并在可编译的代码中明确地记录下来。
最后，使用absl::Status返回错误比抛出和捕获异常要快很多。
这些特性在编写代码时可能显得很繁琐，但其结果对每个必须阅读这些代码的人来说都是win，对整个Google来说也是如此。

Conclusion
错误处理是最容易出错的事情之一：这些都是本质上的边缘情况。
像Status这样的实用工具，可以为跨API边界、项目、流程和语言的错误处理增加一致性，帮助我们将一大类 "我们的错误处理有问题 "的错误降到最低。
如果你正在设计一个需要表达失败可能性的接口，如果你没有相当强大的理由，请使用Status。


## 187. std::unique_ptr Must Be Moved

A std::unique_ptr is used for expressing transfer of ownership. 
If you never pass ownership elsewhere, the std::unique_ptr abstraction is rarely necessary or appropriate.
简单明了，且硬核。
If it is never std::moved, it likely should not be a std::unique_ptr.

Common Anti-Pattern: Avoiding \&
```cpp
int ComputeValue() {
  auto data = absl::make_unique<Data>();
  ModifiesData(data.get());
  return data->GetValue();
}
```
这个例子中data不需要声明为unique_ptr，因为没有发生所有权转移，直接用栈数据更合适
```cpp
int ComputeValue() {
  Data data;
  ModifiesData(&data);
  return data.GetValue();
}
```

Common Anti-Pattern: Delayed Initialization
```cpp
class MyTest : public testing::Test {
 public:
  void SetUp() override {
    thing_ = absl::make_unique<Thing>(data_);
  }

 protected:
  Data data_;
  // Initialized in `SetUp()`, so we're using `std::unique_ptr` as a
  // delayed-initialization mechanism.
  std::unique_ptr<Thing> thing_;
};
```
Once again，没有发生所有权转移，代码改为
```cpp
class MyTest : public testing::Test {
 public:
  MyTest() : thing_(data_) {}

 private:
  Data data_;
  Thing thing_;
};
```
如果延迟初始化实在不可避免，考虑使用optional和emplace
```cpp
class MyTest : public testing::Test {
 public:
  MyTest() {
    Initialize(&data_);
    thing_.emplace(data_);
  }

 private:
  Data data_;
  absl::optional<Thing> thing_;
};
```

注意：
当然有些例外可以使用unique_ptr，但是文档必须讲清楚

Large, rarely used objects.
optional存的是栈上的数据，有大小限制，如果对空间敏感，那么可以选择采用unique_ptr。

Legacy APIs
一些过去的代码会返回指针，所以可以用unique_ptr包起来，解决内存泄露问题
过去的接口
```cpp
Widget *CreateLegacyWidget() { return new Widget; }

int func() {
  Widget *w = CreateLegacyWidget();
  return w->num_gadgets();
}  // Memory leak!
```
改为
```cpp
int func() {
  std::unique_ptr<Widget> w(CreateLegacyWidget());
  return w->num_gadgets();
}  // `w` is properly destroyed.
```

## 116. Keeping References on Arguments

Const References vs. Pointers to Const
作为函数的参数，const 引用与指向 const 的指针相比有几个优势：它们不能为空，而且很明显函数并没有取得对象的所有权。
但它们还有其他的不同之处，有时可能会出现问题：它们更隐含（即在call site上没有任何东西显示我们正在获取一个引用），
而且它们可以被绑定到一个临时的对象上。

Risk of a Dangling Reference in Classes
```cpp
class Foo {
 public:
  explicit Foo(const std::string& content) : content_(content) {}
  const std::string& content() const { return content_; }

 private:
  const std::string& content_;
};
```
bug
```cpp
void Func() {
  Foo foo("something");
  std::cout << foo.content();  // BOOM!
}
```
foo.content\_指向的是一个临时string对象

A Solution: Use Pointers
```cpp
class Foo {
 public:
  // Do not forget this comment:
  // Does not take ownership of content, which must refer to a valid string that
  // outlives this object.
  explicit Foo(const std::string* content) : content_(content) {}
  const std::string& content() const { return *content_; }

 private:
  const std::string* const content_;  // not owned, can't be null
};
```

```cpp
std::string GetString();
void Func() {
  Foo foo1(&GetString());  // error: taking the address of a temporary of
                           // type 'std::string'
  Foo foo2(&"something");  // error: no matching constructor for initialization
                           // of 'Foo'
}
```
```cpp
void Func2() {
  std::string content = GetString();
  Foo foo(&content);
}
```

One Step Further, One Less Comment: Storing a Reference
上面的例子里，你可能已经注意到了，强调了两次指针不能是空的，也不能拥有它，
一次是在构造函数的文档中，然后是在实例变量的注释中。这有必要吗？考虑一下这个。
```cpp
class Baz {
 public:
  // Does not take any ownership, and all pointers must refer to valid objects
  // that outlive the one constructed.
  explicit Baz(const Arg1* arg1, Arg2* arg2) : arg1_(*arg1), arg2_(*arg2) {}

 private:
  // It is now clear that we do not have ownership and that the references can't
  // be null.
  const Arg1& arg1_;
  Arg2& arg2_;  // Yes, non-const references are style-compliant!
};
```
引用类型的成员的一个缺点是你不能reassign它们，这意味着你的类不会有一个assignment operator（copy constructors还是可以的），
但为了遵守rule three，显式删除它可能是有意义的。 如果你的类应该是assignable的，你将需要非const指针，仍然有可能是const对象。

Conclusion
如果参数是复制的，或者只是在构造函数中使用，而在构造对象中没有保留对它的引用，那么通过const引用向构造函数传递参数还是可以的。
在其他情况下，可以考虑通过指针传递参数（无论是否传递给const）。
另外，请记住，如果你实际上是在转移一个对象的所有权，应该以std::unique_ptr的形式传递。

最后，这里讨论的内容并不限于构造函数：任何以某种方式保留其参数的别名的函数，
无论是通过将指针放在缓存中还是将参数绑定在分离的函数中，都应该通过指针来获取该参数。


## 165. if and switch statements with initializers

C++17支持
```cpp
if (init; cond) { /* ... */ }
switch (init; cond) { /* ... */ }
```
```cpp
if (auto it = m.find("key"); it != m.end()) {
  return it->second;
} else {
  return absl::NotFoundError("Entry not found");
}
```

```cpp
int w;

if (int x, y, z; int y = g()) {   // error: y redeclared, first declared in initializer
  int x;                          // error: x redeclared, first declared in initializer
  int w;                          // OK, shadows outer variable
  {
    int x, y;                     // OK, shadowing in nested scope is allowed
  }
} else {
  int z;                          // error: z redeclared, first declared in initializer
}

if (int w; int q = g()) {         // declaration of "w" OK, shadows outer variable
  int q;                          // error: q redeclared, first declared in condition
  int w;                          // error: w redeclared, first declared in initializer
}
```

Interaction with structured bindings
C++17 also introduces structured bindings
```cpp
if (auto [iter, ins] = m.try_emplace(key, data); ins) {
  use(iter->second);
} else {
  std::cerr << "Key '" << key << "' already exists.";
}
```
另一个例子来自于使用C++17的node handle，它允许真正的在map或set之间移动元素而不需要复制。
这个特性定义了一个可解构的insert-return-type，它是插入节点句柄的结果。
```cpp
if (auto [iter, ins, node] = m2.insert(m1.extract(k)); ins) {
  std::cout << "Element with key '" << k << "' transferred successfully";
} else if (!node) {
  std::cerr << "Key '" << k << "' does not exist in first map.";
} else {
  std::cerr << "Key '" << k << "' already in m2; m2 unchanged; m1 changed.";
}
```
extract is the only way to change a key of a map element without reallocation:
```cpp
map<int, string> m{{1, "mango"}, {2, "papaya"}, {3, "guava"}};
auto nh = m.extract(2);
nh.key() = 4;
m.insert(move(nh));
// m == {{1, "mango"}, {3, "guava"}, {4, "papaya"}}
```

## 186. Prefer to Put Functions in the Unnamed Namespace

如果添加一个函数，放在.cc文件的unnamed namespace中，成为一个非成员函数是一个好的选择。

先谈Benefits
1 比函数声明在头文件中好在：
* reader更容易找到函数定义
* 可以把文档、声明、定义都放在一个位置，相比于头文件声明，.cc文件定义
* 和其他源文件隔离，便于重构
* 不用纠结meaningful const，因为它没有分离的声明(Tips 109)
* 和其他代码实现放在同个位置，便于使用aliases和local types(Tips 119)
* 一个type也可以移到unnamed namespace，如果这个type只在这个源文件中使用
2 比私有函数好在：
* inputs和outputs通过arguments和return value描述比较清晰。私有函数可以读取
任何成员变量，一个非const函数可以修改任何非const成员，相反，一个非成员函数
只能read，且通过interface修改数据。
* 类的API更简单，简短，更易阅读，非必需的私有函数让继承相关的私有声明不容易找到。

当然有些时候non-member函数不适用
* 如果一个函数需要在多个源文件中使用，那就需要在头文件中声明
* 如果函数需要和类或对象有很多交互，比如，函数要读取很多字段且需要修改内部状态，通过return value
不是很方便，直接写个函数更加适用，例如在一个成员函数中使用mutex
* 函数是API的一部分

Alternative: static Non-Member Functions
给一个非成员函数加上static跟在unnamed namespace定义非成员函数具有一样的效果，目的是隔离其他
翻译单元代码，在unnamed namespaces的好处是可以对函数和对象进行统一的处理，但有些人喜欢在函数的声明中明确地写上static，
以表明它是一个翻译单元的本地函数，而不必检查包围的未命名命名空间。

给了三个链接，因为这三个链接讲的都不是很全，所以有这个tip
[The unnamed namespaces section](https://google.github.io/styleguide/cppguide.html#Unnamed_Namespaces_and_Static_Variables)
[The inputs and outputs section](https://google.github.io/styleguide/cppguide.html#Inputs_and_Outputs)
[The local variables section](https://google.github.io/styleguide/cppguide.html#Local_Variables)

总结：
File-local函数简化依赖，提高locality。非成员函数提高封装，简化函数定义，依赖更显式，如果写一个函数，可以设计为file-local且非成员函数
，比如放入.cc中的unnamed namespace。
