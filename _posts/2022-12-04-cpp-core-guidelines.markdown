---
layout: post
title:  "Cpp Core Guidelines"
date: 2022-12-04 14:00:00 +0800
categories: jekyll update
---

**Major sections**
- [P:哲学](#p哲学)
- [I:接口](#i接口)
- [F:函数](#f函数)
- [C:类和类的层次结构](#c类和类的层次结构)
- [E:枚举](#e枚举)
- [R:资源管理](#r资源管理)
- [ES:表达式和语句](#es表达式和语句)
- [Per:性能](#per性能)
- [CP:并发和并行](#cp并发和并行)
- [E:错误处理](#e错误处理)
- [Con:常量和不变性](#con常量和不变性)
- [T:模版和泛型编程](#t模版和泛型编程)
- [CPL:C风格编程](#cplc风格编程)
- [SF:源文件](#sf源文件)
- [SL:标准库](#sl标准库)

**Supporting sections**
- [A:架构理念](#a架构理念)
- [NR:非规则和神话](#nr非规则和神话)
- [Pro:Profiles](#proprofiles)
- [NL:命名和布局建议](#nl命名和布局建议)
- [Appendix C:Discussion](#appendix-cdiscussion)

<br/>
# **P:哲学**

哲学规则摘要：
- [P.1:直接用代码表达想法](#p1直接用代码表达想法)
- [P.2:使用ISO标准C++](#p2使用iso标准c)
- [P.3:表达意图](#p3表达意图)
- [P.4:理想情况下一个程序应该是静态类型安全的](#p4理想情况下一个程序应该是静态类型安全的)
- [P.5:优先使用编译检查而不是运行检查](#p5优先使用编译检查而不是运行检查)
- [P.6:不能在编译时检查的东西应该在运行时检查](#p6不能在编译时检查的东西应该在运行时检查)
- [P.7:运行时应尽早捕获错误](#p7运行时应尽早捕获错误)
- [P.8:不要泄漏资源](#p8不要泄漏资源)
- [P.9:不要浪费时间和空间](#p9不要浪费时间和空间)
- [P.10:优先使用不可变数据而不是可变数据](#p10优先使用不可变数据而不是可变数据)
- [P.11:封装混乱的结构，而不是任其散布代码](#p11封装混乱的结构而不是任其散布代码)
- [P.12:适当使用辅助工具](#p12适当使用辅助工具)
- [P.13:适当使用辅助库](#p13适当使用辅助库)

<br/>
### **P.1:直接用代码表达想法**

**Example**
```cpp
class Date {
public:
    Month month() const;  // do
    int month();          // don't
    // ...
};
```
第一个声明是明确的，即返回一个Month且不改变Date内部状态。

**Example,bad**
```cpp
void f(vector<string>& v)
{
    string val;
    cin >> val;
    // ...
    int index = -1;                    // bad, plus should use gsl::index
    for (int i = 0; i < v.size(); ++i) {
        if (v[i] == val) {
            index = i;
            break;
        }
    }
    // ...
}
```
**Example,good**
```cpp
void f(vector<string>& v)
{
    string val;
    cin >> val;
    // ...
    auto p = find(begin(v), end(v), val);  // better
    // ...
}
```

**Example,bad**
```cpp
change_speed(double s);   // bad: what does s signify?
// ...
change_speed(2.3);
```
**Example,good**
```cpp
change_speed(Speed s);    // better: the meaning of s is specified
// ...
change_speed(2.3);        // error: no unit
change_speed(23_m / 10s);  // meters per second
```

<br/>
### **P.2:使用ISO标准C++**

<br/>
### **P.3:表达意图**

**Example**
```cpp
gsl::index i = 0;
while (i < v.size()) {
    // ... do something with v[i] ...
}
```
这里没有表达仅仅在元素上循环的意图，索引的实现细节被暴露出来，所以它可能被误用，另外i也可能超出范围。改为：
```cpp
for (const auto& x : v) { /* do something with the value of x */ }
```
如果想要可修改的：
```cpp
for (auto& x : v) { /* modify x */ }
```

**Example**
```cpp
draw_line(int, int, int, int);  // obscure
draw_line(Point, Point);        // clearer
```

<br/>
### **P.4:理想情况下一个程序应该是静态类型安全的**

**原因** 理想情况下，一个程序应该是完全静态的类型安全的（编译时），但这不可能，以下例外：
- unions
- casts
- 数组退化
- 范围错误
- narrowing conversions

**Enforcement**
- unions用variant替换(C++17)
- casts尽量少用，用模版
- 数组退化-使用span
- 范围错误-使用span
- narrowing conversions-使用GSL的narrow or narrow\_cast

<br/>
### **P.5:优先使用编译检查而不是运行检查**

**原因** 代码清晰以及性能。你不需要编写编译期间的错误处理。

**Example**
```cpp
// Int is an alias used for integers
int bits = 0;         // don't: avoidable code
for (Int i = 1; i; i <<= 1)
    ++bits;
if (bits < 32)
    cerr << "Int too small\n";
```
左移可能溢出导致未定义。应该做个静态检查。
```cpp
// Int is an alias used for integers
static_assert(sizeof(Int) >= 4);    // do: compile-time check
```
或者更好的是使用类型系统，用int32\_t替换int。
**Example**
```cpp
void read(int* p, int n);   // read max n integers into *p

int a[100];
read(a, 1000);    // bad, off the end
```
改为：
```cpp
void read(span<int> r); // read into the range of integers r

int a[100];
read(a);        // better: let the compiler figure out the number of elements
```

<br/>
### **P.6:不能在编译时检查的东西应该在运行时检查**

**Example,bad**
```cpp
// separately compiled, possibly dynamically loaded
extern void f(int* p);

void g(int n)
{
    // bad: the number of elements is not passed to f()
    f(new int[n]);
}
```

**Example,bad**
```cpp
// separately compiled, possibly dynamically loaded
extern void f2(int* p, int n);

void g2(int n)
{
    f2(new int[n], m);  // bad: a wrong number of elements can be passed to f()
}
```

**Example,bad**
```cpp
// separately compiled, possibly dynamically loaded
// NB: this assumes the calling code is ABI-compatible, using a
// compatible C++ compiler and the same stdlib implementation
extern void f3(unique_ptr<int[]>, int n);

void g3(int n)
{
    f3(make_unique<int[]>(n), m);    // bad: pass ownership and size separately
}
```

**Example**
```cpp
extern void f4(vector<int>&);   // separately compiled, possibly dynamically loaded
extern void f4(span<int>);      // separately compiled, possibly dynamically loaded
                                // NB: this assumes the calling code is ABI-compatible, using a
                                // compatible C++ compiler and the same stdlib implementation

void g3(int n)
{
    vector<int> v(n);
    f4(v);                     // pass a reference, retain ownership
    f4(span<int>{v});          // pass a view, retain ownership
}
```
这种设计将元素的数量作为对象的一部分，因此不太可能出错，动态检查总是可行的。

**Example**
```cpp
vector<int> f5(int n)    // OK: move
{
    vector<int> v(n);
    // ... initialize v ...
    return v;
}

unique_ptr<int[]> f6(int n)    // bad: loses n
{
    auto p = make_unique<int[]>(n);
    // ... initialize *p ...
    return p;
}

owner<int*> f7(int n)    // bad: loses n and we might forget to delete
{
    owner<int*> p = new int[n];
    // ... initialize *p ...
    return p;
}
```

<br/>
### **P.7:运行时应尽早捕获错误**

**Example**
```cpp
void increment1(int* p, int n)    // bad: error-prone
{
    for (int i = 0; i < n; ++i) ++p[i];
}

void use1(int m)
{
    const int n = 10;
    int a[n] = {};
    // ...
    increment1(a, m);   // maybe typo, maybe m <= n is supposed
                        // but assume that m == 20
    // ...
}
```

```cpp
void increment2(span<int> p)
{
    for (int& x : p) ++x;
}

void use2(int m)
{
    const int n = 10;
    int a[n] = {};
    // ...
    increment2({a, m});    // maybe typo, maybe m <= n is supposed
    // ...
}
```

```cpp
void use3(int m)
{
    const int n = 10;
    int a[n] = {};
    // ...
    increment2(a);   // the number of elements of a need not be repeated
    // ...
}
```

**Example,bad**
```cpp
Date read_date(istream& is);    // read date from istream

Date extract_date(const string& s);    // extract date from string

void user1(const string& date)    // manipulate date
{
    auto d = extract_date(date);
    // ...
}

void user2()
{
    Date d = read_date(cin);
    // ...
    user1(d.to_string());
    // ...
}
```

<br/>
### **P.8:不要泄漏资源**

**Example,bad**
```cpp
void f(char* name)
{
    FILE* input = fopen(name, "r");
    // ...
    if (something) return;   // bad: if something == true, a file handle is leaked
    // ...
    fclose(input);
}
```
应该用RAII:
```cpp
void f(char* name)
{
    ifstream input {name};
    // ...
    if (something) return;   // OK: no leak
    // ...
}
```

**注意** 强制执行生命期安全配置文件可以消除泄漏。当与RAII提供的资源安全相结合时，它消除了垃圾回收的需求（因为它不生产垃圾）。将此与
类型和边界配置文件执行结合起来，将会得到完整的类型和资源安全，这可以由工具来保证。

<br/>
### **P.9:不要浪费时间和空间**

**Example,bad**
```cpp
struct X {
    char ch;
    int i;
    string s;
    char ch2;

    X& operator=(const X& a);
    X(const X&);
};

X waste(const char* p)
{
    if (!p) throw Nullptr_error{};
    int n = strlen(p);
    auto buf = new char[n];
    if (!buf) throw Allocation_error{};
    for (int i = 0; i < n; ++i) buf[i] = p[i];
    // ... manipulate buffer ...
    X x;
    x.ch = 'a';
    x.s = string(n);    // give x.s space for *p
    for (gsl::index i = 0; i < x.s.size(); ++i) x.s[i] = buf[i];  // copy buf into x.s
    delete[] buf;
    return x;
}

void driver()
{
    X x = waste("Typical argument");
    // ...
}
```
X的布局导致了至少6个字节（甚至更多）被浪费，拷贝构造函数禁用了移动语意，所以返回操作很慢。对于buf的new和delete是多余的，如果真的需要一个局部字符串对象，
那么可以使用string。

**Example,bad**
```cpp
void lower(zstring s)
{
    for (int i = 0; i < strlen(s); ++i) s[i] = tolower(s[i]);
}
```
每一个loop都将会计算strlen，当字符串内容变化时，我们认为tolower不会影响strlen，所以最好在循环外获取缓存大小，这样就不要在每个循环里产生成本。

<br/>
## **P.10:优先使用不可变数据而不是可变数据**

<br/>
## **P.11:封装混乱的结构，而不是任其散布代码**

**Example**
```cpp
int sz = 100;
int* p = (int*) malloc(sizeof(int) * sz);
int count = 0;
// ...
for (;;) {
    // ... read an int into x, exit loop if end of file is reached ...
    // ... check that x is valid ...
    if (count == sz)
        p = (int*) realloc(p, sizeof(int) * sz * 2);
    p[count++] = x;
    // ...
}
```
可以换为：
```cpp
vector<int> v;
v.reserve(100);
// ...
for (int x; cin >> x; ) {
    // ... check that x is valid ...
    v.push_back(x);
}
```

<br/>
## **P.12:适当使用辅助工具**

- 静态分析工具
- 并发工具
- 测试工具

<br/>
## **P.13:适当使用辅助库**

**Example**
```cpp
std::sort(begin(v), end(v), std::greater<>());
```

<br/>
# **I:接口**

接口规则摘要：
- [I.1:显式明确接口](#i1显式明确接口)
- [I.2:避免使用非const全局变量](#i2避免使用非const全局变量)
- [I.3:避免使用单例](#i3避免使用单例)
- [I.4:接口设计精确且强类型化](#i4接口设计精确且强类型化)
- [I.5:说明先决条件（如果有的话）](#i5说明先决条件如果有的话)
- [I.6:优先使用Expects来表达先决条件](#i6优先使用expects来表达先决条件)
- [I.7:说明后置条件](#i7说明后置条件)
- [I.8:优先使用Ensures来表达后置条件](#i8优先使用ensures来表达后置条件)
- [I.9:如果一个接口是模版，那么用concepts来记录它的参数](#i9如果一个接口是模版那么用concepts来记录它的参数)
- [I.10:用异常来表示未能执行所需任务](#i10用异常来表示未能执行所需任务)
- [I.11:不要通过裸指针T\*或T&来转移所有权](#i11不要通过裸指针t或t来转移所有权)
- [I.12:将一个不为空的指针声明为not\_null](#i12将一个不为空的指针声明为not_null)
- [I.13:不要把数组作为一个单一的指针来传递](#i13不要把数组作为一个单一的指针来传递)
- [I.22:避免全局对象的复杂初始化](#i22避免全局对象的复杂初始化)
- [I.23:函数参数个数尽量少](#i23函数参数个数尽量少)
- [I.24:避免相邻的参数可以由相同的参数以任意顺序调用，但意义不同](#i24避免相邻的参数可以由相同的参数以任意顺序调用但意义不同)
- [I.25:优先使用空的抽象类作为类层次的接口](#i25优先使用空的抽象类作为类层次的接口)
- [I.26:如果你想要一个跨编译器的ABI，请使用C风格子集](#i26如果你想要一个跨编译器的abi请使用c风格子集)
- [I.27:为了稳定的库ABI，考虑用Pimpl](#i27为了稳定的库abi考虑用pimpl)
- [I.30:封装反规则的行为](#i30封装反规则的行为)

**还有：**
- [F:函数](#f函数)
- [C.concrete:实体类型](#cconcrete实体类型)
- [C.hier:类分层](#chier类分层)
- [C.over:重载运算符](#cover重载运算符)
- [C.con:容器及其他资源句柄](#ccon容器及其他资源句柄)
- [E:错误处理](#e错误处理)
- [T:模版和元编程](#t模版和元编程)

<br/>
### **I.1:显式明确接口**

**原因** 正确性。接口中没有明确的假设很容易被忽视，也很难测试。

**Example,bad** 通过全局（命名空间范围）变量（调用模式）来控制一个函数的行为是隐式的，且造成混乱。
```cpp
int round(double d)
{
    return (round_up) ? ceil(d) : d;    // don't: "invisible" dependency
}
```
对于调用者，两次调用round(7.2)可能会得到不同的结果。这一点并不明显。

**Example,bad** 通过非局部变量(errno)的报告很容易被忽略。
```cpp
// don't: no test of printf's return value
fprintf(connection, "logging: %d %d %d\n", x, y, s);
```

<br/>
### **I.2:避免使用非const全局变量**

**原因** 非const全局变量隐藏了依赖关系，并使依赖关系受到不可预测变化的影响。

**Example**
```cpp
struct Data {
    // ... lots of stuff ...
} data;            // non-const data

void compute()     // don't
{
    // ... use data ...
}

void output()     // don't
{
    // ... use data ...
}
```
还有谁可能修改了数据？

**注意** 全局对象的初始化不是有序的。如果你使用一个全局对象，就用一个常量来初始化它。注意，即使是一个常量对象，也可能是一个为定义的初始化顺序。

**例外** 一个全局对象往往比单例更好。

**注意** 全局常量是有用的。

**替换** 如果你使用全局数据来避免复制，可以考虑将数据用const引用传递。另一个解决方案是将数据定义为一个对象的状态，将操作定义为成员函数。

**警告** 注意data races。如果一个线程可以访问非局部数据（或通过引用传递），而另外一个线程执行calle，那么就会出现data races。每个指针或可变数据的引用都是一个潜在的data race。

**Enforcement** 所有在命名空间的非const变量以及指向非const数据的全局指针或全局引用都应该被标记出来。

<br/>
### **I.3:避免使用单例**

**原因** 单例是伪装的复杂全局对象

**Example**
```cpp
class Singleton {
    // ... lots of stuff to ensure that only one Singleton object is created,
    // that it is initialized properly, etc.
};
```

**注意** 如果你不希望全局对象发生变化，那么就声明为const或constexpr。

**例外** 可以使用最简单的单例模式，在第一次使用的时候获得初始化，如果有的话。
```cpp
X& myX()
{
    static X my_x {3};
    return my_x;
}
```
这是解决与初始化顺序相关问题最有效的办法之一。在多线程环境中，静态对象的初始化不会引入data race，除非你在构造函数中访问一个共享对象。
注意局部静态对象的初始化不会有data race，然而如果x的销毁涉及到需要同步的操作，我们必须使用一个不太简单的解决方案。
```cpp
X& myX()
{
    static auto p = new X {3};
    return *p;  // potential leak
}
```
现在需要有人负责线程安全地delete对象。这很容易出错，所以我们不使用这种技术，除非：
- myX是支持多线程代码
- X对象需要销毁
- X析构函数需要同步

<br/>
### **I.4:接口设计精确且强类型化**

**原因** 类型是最简单和最好的文档，由于其定义明确的含义，提高了可读性，并在编译时进行检查。另外，精确的类型化代码通常会被更好地优化。

**Example,bad**
```cpp
void pass(void* data);    // weak and under qualified type void* is suspicious
```
调用者不确定接口允许什么类型数据，以及数据是否可被修改，因为没有指定const。所有的指针类型都可以隐式地转换为void\*。

**Example,bad**
```cpp
draw_rect(100, 200, 100, 500); // what do the numbers specify?

draw_rect(p.x, p.y, 10, 20); // what units are 10 and 20 in?
```
使用类型系统：
```cpp
void draw_rectangle(Point top_left, Point bottom_right);
void draw_rectangle(Point top_left, Size height_width);

draw_rectangle(p, Point{10, 20});  // two corners
draw_rectangle(p, Size{10, 20});   // one corner and a (height, width) pair
```

**Example,bad**
```cpp
set_settings(true, false, 42); // what do the numbers specify?
```
改为更显式，安全以及可读性
```cpp
alarm_settings s{};
s.enabled = true;
s.displayMode = alarm_settings::mode::spinning_light;
s.frequency = alarm_settings::every_10_seconds;
set_settings(s);
```
对于一组bool情况，可以考虑使用enum，它可以表达一组bool值。
```cpp
enable_lamp_options(lamp_option::on | lamp_option::animate_state_transitions);
```

**Example,bad**
```cpp
time_to_blink means: Seconds? Milliseconds?
void blink_led(int time_to_blink) // bad -- the unit is ambiguous
{
    // ...
    // do something with time_to_blink
    // ...
}

void use()
{
    blink_led(2);
}
```
**Example,good**
```cpp
void blink_led(milliseconds time_to_blink) // good -- the unit is explicit
{
    // ...
    // do something with time_to_blink
    // ...
}

void use()
{
    blink_led(1500ms);
}
```
还有一种可以接受更多时间单位：
```cpp
template<class rep, class period>
void blink_led(duration<rep, period> time_to_blink) // good -- accepts any unit
{
    // assuming that millisecond is the smallest relevant unit
    auto milliseconds_to_blink = duration_cast<milliseconds>(time_to_blink);
    // ...
    // do something with milliseconds_to_blink
    // ...
}

void use()
{
    blink_led(2s);
    blink_led(1500ms);
}
```

<br/>
### **I.5:说明先决条件（如果有的话）**

**原因** 参数的意义可能会制约它们在被调用者中的正确使用。

**Example**
```cpp
double sqrt(double x);
```
x应该是大等于0的。类型系统无法表达这个，所以可以人为加个注释。
```cpp
double sqrt(double x); // x must be non-negative
```
也可以加个assertions
```cpp
double sqrt(double x) { Expects(x >= 0); /* ... */ }
```
理论上Expects应该属于接口的一部分，但是这很难做到。那么我们把它放入定义（函数体）。

**注意** 优先使用正式的需求说明。比如Expects，如果这不可行，那么可以使用注释，比如// [p:q)表达一个有序队列，用<排序。

<br/>
### **I.6:优先使用Expects来表达先决条件**

**原因** 明确该条件是个前提条件，且工具得以使用。

**Example**
```cpp
int area(int height, int width)
{
    Expects(height > 0 && width > 0);            // good
    if (height <= 0 || width <= 0) my_error();   // obscure
    // ...
}
```

**注意** 前提条件可以用很多方式来表述，包括注释、if语句和assert()。这可能会使它们与普通代码难以区分，难以更新，难以被工具操作，而且可能会有错误的语义（你总是想在调试模式下中止，而在产品运行中什么都不检查吗）。

**注意** 前提条件应该是接口的一部分，而不是实现的一部分，但我们还没有语言设施来做到这一点。一旦语言支持变得可用，我们将采用标准版本的前提条件、后置条件和断言。

**注意** Expects()也可以用来检测算法中的一个条件。

<br/>
### **I.7:说明后置条件**

**原因** 检测对结果的误解，并指出错误的实现。

**Example,bad**
```cpp
int area(int height, int width) { return height * width; }  // bad
```
这里我们没有显式限定前置条件，比如宽和高都应该大于0。我们也忽略了后置条件限定，比如相乘的面积可能超过int最大值造成溢出。
```cpp
int area(int height, int width)
{
    auto res = height * width;
    Ensures(res > 0);
    return res;
}
```

**Example,bad**
```cpp
void f()    // problematic
{
    char buffer[MAX];
    // ...
    memset(buffer, 0, sizeof(buffer));
}
```
这里没有使用后置条件判断，造成优化器认定memset是多余的，那memset就会被优化掉。
改为：
```cpp
void f()    // better
{
    char buffer[MAX];
    // ...
    memset(buffer, 0, sizeof(buffer));
    Ensures(buffer[0] == 0);
}
```

**注意** 后置条件通常在非正式的注释中出现。使用Ensures可以更系统，可见以及可检查性。

**Example**
```cpp
mutex m;

void manipulate(Record& r)    // don't
{
    m.lock();
    // ... no m.unlock() ...
}
```
这里我们忽略了说明mutex应该被释放，直接加个后置条件声明会更清晰。
```cpp
void manipulate(Record& r)    // postcondition: m is unlocked upon exit
{
    m.lock();
    // ... no m.unlock() ...
}
```
最好使用RAII:
```cpp
void manipulate(Record& r)    // best
{
    lock_guard<mutex> _ {m};
    // ...
}
```

**注意** 理想情况下，后置条件要在接口/声明中说明，以便用户容易看到他们。只有与用户相关的后置条件可以在接口中说明。\
至于内部状态相关的后置条件在定义/实现中。

<br/>
### **I.8:优先使用Ensures来表达后置条件**

**原因** 明确该条件是后置条件，且工具得以使用。

**Example**
```cpp
void f()
{
    char buffer[MAX];
    // ...
    memset(buffer, 0, MAX);
    Ensures(buffer[0] == 0);
}
```

<br/>
### **I.9:如果一个接口是模版，那么用concepts来记录它的参数**

**原因** 接口更精确地约定，以及编译期间就可以检查。

**Example**
```cpp
template<typename Iter, typename Val>
  requires input_iterator<Iter> && equality_comparable_with<iter_value_t<Iter>, Val>
Iter find(Iter first, Iter last, Val v)
{
    // ...
}
```

<br/>
### **I.10:用异常来表示未能执行所需任务**

**原因** 不应该忽略错误，这会导致系统或计算处于一个为定义的状态。

**Example,bad**
```cpp
int printf(const char* ...);    // bad: return negative number if output fails

template<class F, class ...Args>
// good: throw system_error if unable to start the new thread
explicit thread(F&& f, Args&&... args);
```

**替代方案** 如果你无法使用异常（例如代码中很多老式的裸指针使用），考虑使用返回一对值。
```cpp
int val;
int error_code;
tie(val, error_code) = do_something();
if (error_code) {
    // ... handle the error or exit ...
}
// ... use val ...
```
C++17:
```cpp
auto [val, error_code] = do_something();
if (error_code) {
    // ... handle the error or exit ...
}
// ... use val ...
```

<br/>
### **I.11:不要通过裸指针T*或T&来转移所有权**

**原因** 如果对调用者或者被调用者是否拥有一个对象有疑问，就可能发生泄漏或过早销毁。

**Example**
```cpp
X* compute(args)    // don't
{
    X* res = new X{};
    // ...
    return res;
}
```
谁来负责释放X？如果compute返回一个引用将更难分析，此时可以返回值，如果返回对象比较大，可以使用移动语意。
```cpp
vector<double> compute(args)  // good
{
    vector<double> res(10000);
    // ...
    return res;
}
```

**替代方案** 使用智能指针来传递所有权，unique\_ptr独占所有权，shared\_ptr共享所有权。但直接返回对象本身更优雅，更高效。
只有在需要引用语意的情况下才使用智能指针。

**替代方案** 有时为了ABI兼容，老代码不能被修改，这时候应该使用GSL的owner。
```cpp
owner<X*> compute(args)    // It is now clear that ownership is transferred
{
    owner<X*> res = new X{};
    // ...
    return res;
}
```
这将告诉分析工具res是所有者，返回的值应该被释放或者被转移到其他所有者。

**注意** 每一个作为裸指针被传递的对象都被认为是由调用者拥有所有权，所以它的生命周期也由调用者管理。换个角度看：与指针传递的API相比，
所有权转移的API相当罕见，所以默认是没有所有权转移。

<br/>
### **I.12:将一个不为空的指针声明为not_null**

**原因** 避免解引用nullptr，避免多余的nullptr检查。

**Example**
```cpp
int length(const char* p);            // it is not clear whether length(nullptr) is valid

length(nullptr);                      // OK?

int length(not_null<const char*> p);  // better: we can assume that p cannot be nullptr

int length(const char* p);            // we must assume that p can be nullptr
```
通过在源码中说明意图，实现者和工具可以提供更好的诊断，例如通过静态分析来发现某些类别的错误，并进行优化，例如删除分支和判空测试。

**注意** char字符指针是隐式的，潜在的混淆和错误来源，优先使用zstring，而不是const char\*。
```cpp
// we can assume that p cannot be nullptr
// we can assume that p points to a zero-terminated array of characters
int length(not_null<zstring> p);
```

<br/>
### **I.13:不要把数组作为一个单一的指针来传递**

**原因** 指针风格的接口易错。一个指向数组的指针必须依靠一些其他数据来确定被调用者大小。

**Example**
```cpp
void copy_n(const T* p, T* q, int n); // copy from [p:p+n) to [q:q+n)
```
如果q指向的数组元素个数少于n？如果p指向的数组元素个数少于n?

**替代方案**
```cpp
void copy(span<const T> r, span<T> r2); // copy r to r2
```

另外一个例子：
**Example,bad**
```cpp
Example, bad Consider:
void draw(Shape* p, int n);  // poor interface; poor code
Circle arr[10];
// ...
draw(arr, 10);
```
传10可能是个错误，因为常见的范围是[0,10)。draw无法安全地遍历数组，因为它没法知道元素大小。Circle隐式退化成Shape。

**替代方案**
```cpp
void draw2(span<Circle>);
Circle arr[10];
// ...
draw2(span<Circle>(arr));  // deduce the number of elements
draw2(arr);    // deduce the element type and array size

void draw3(span<Shape>);
draw3(arr);    // error: cannot convert Circle[10] to span<Shape>
```
draw2传入的信息跟draw一样多，而且更显式列出需要一个Circle的range。

**例外** 使用zstring和czstring来表示C风格的，以零结尾的字符串。但是当你这样做的时候，请使用std::string\_view或GSL中的span<char>来防止range错误。

<br/>
### **I.22:避免全局对象的复杂初始化**

**原因** 复杂初始化会导致执行顺序的不确定性。

**Example**
```cpp
// file1.c

extern const X x;

const Y y = f(x);   // read x; write y

// file2.c

extern const Y y;

const X x = g(y);   // read y; write x
```
x和y在不同的翻译单元，f()和g()的调用顺序是未定义的。可能会访问一个未初始化的const。

**注意** 初始化顺序在并发代码中特别难以处理。通常最好避免全局对象（命名范围空间内）。

**Enforcement**
- 调用非const的全局变量初始化应该标记出
- 访问extern的全局变量初始化也应该标记出

<br/>
### **I.23:函数参数个数尽量少**

**原因** 多参数容易混淆，开销也更大。

**讨论** 两个常见原因：
- 缺少抽象。
- 违反one function，one responsibility。

**Example** 
```cpp
template<class InputIterator1, class InputIterator2, class OutputIterator, class Compare>
OutputIterator merge(InputIterator1 first1, InputIterator1 last1,
                     InputIterator2 first2, InputIterator2 last2,
                     OutputIterator result, Compare comp);
```
缺少抽象，STL没有传递一个range，而是传递了迭代器pair。这里我们有4个模版参数和6个函数参数。
首先可以简化的是使用默认比较符<。
```cpp
emplate<class InputIterator1, class InputIterator2, class OutputIterator>
OutputIterator merge(InputIterator1 first1, InputIterator1 last1,
                     InputIterator2 first2, InputIterator2 last2,
                     OutputIterator result);
```
这并没有减少总的复杂度，为了真正减少参数数量，我们需要将参数更高抽象。
```cpp
template<class In1, class In2, class Out>
  requires mergeable<In1, In2, Out>
Out merge(In1 r1, In2 r2, Out result);
```
再举个例子：
```cpp
void f(int* some_ints, int some_ints_length);  // BAD: C style, unsafe
```
改为：
```cpp
void f(gsl::span<int> some_ints);              // GOOD: safe, bounds-checked
```
这种抽象更安全，更健壮，而且自然地减少参数个数。

**注意** 函数参数最好低于4个。

**替代方案** 使用更高的抽象。将参数分组为有意义的分组，并传递对象（传值或传引用）。

**替代方案** 使用默认参数或重载，允许用最少的参数完成最常用形式的调用。

**Enforcement** 如果一个函数声明有两个相同类型的迭代器，而不是range或view，那应当发出警告。

<br/>
### **I.24:避免相邻的参数可以由相同的参数以任意顺序调用，但意义不同**

**原因** 同一类型的相邻参数容易被错误调换。

**Example,bad**
```cpp
void copy_n(T* p, T* q, int n);  // copy from [p:p + n) to [q:q + n)
```
这是个糟糕的K&R式接口的变种，它很容易把from和to参数颠倒。
把from改为const:
```cpp
void copy_n(const T* p, T* q, int n);  // copy from [p:p + n) to [q:q + n)
```
**例外** 如果函数参数顺序无所谓：
```cpp
int max(int a, int b);
```

**替代方案** 不要传数组给参数指针，应该传一个对象来表示范围。
```cpp
void copy_n(span<const T> p, span<T> q);  // copy from p to q
```

**替代方案** 定义一个struct，并把参数字段定义为成员。
```cpp
struct SystemParams {
    string config_file;
    string output_path;
    seconds timeout;
};
void initialize(SystemParams p);
```
这会让未来的读者更清晰的看到这些调用，因为参数往往在调用栈中按名字填写。

<br/>
### **I.25:优先使用空的抽象类作为类层次的接口**

**原因** 空的抽象类（没有非静态成员数据）比有状态的基类更稳定。

**Example,bad**
```cpp
class Shape {  // bad: interface class loaded with data
public:
    Point center() const { return c; }
    virtual void draw() const;
    virtual void rotate(int);
    // ...
private:
    Point c;
    vector<Point> outline;
    Color col;
};
```
每一个派生类都要计算center，即使是non-trivial的，而center从未被使用。同样的，不是每一个shape都有color，许多shapes最好不要用定义为points的序列轮廓来表示。\
使用一个抽象类会更好。
```cpp
class Shape {    // better: Shape is a pure interface
public:
    virtual Point center() const = 0;   // pure virtual functions
    virtual void draw() const = 0;
    virtual void rotate(int) = 0;
    // ...
    // ... no data members ...
    // ...
    virtual ~Shape() = default;
};
```

<br/>
### **I.26:如果你想要一个跨编译器的ABI，请使用C风格子集**

**原因** 不同编译器实现了类、异常处理、函数名称等的各自不同的二进制布局。

**例外** 一些平台出现了通用的ABI，从而让你从严厉的限制中解放。

**注意** 如果你使用单一的编译器，你可以在接口中使用完整的C++。这在编译器升级的时候需要重新编译。

<br/>
### **I.27:为了稳定的库ABI，考虑用Pimpl**

**原因** 私有数据成员参与了类的布局，私有成员函数参与了重载决议，对这些数据修改都会让使用这些类的用户重新编译代码。
一个持有指向实现的指针的非多态接口类（Pimpl）可以将一个类的用户和它的实现分离，代价是一个间接性。

**Example** 接口(widget.h)
```cpp
class widget {
    class impl;
    std::unique_ptr<impl> pimpl;
public:
    void draw(); // public API that will be forwarded to the implementation
    widget(int); // defined in the implementation file
    ~widget();   // defined in the implementation file, where impl is a complete type
    widget(widget&&); // defined in the implementation file
    widget(const widget&) = delete;
    widget& operator=(widget&&); // defined in the implementation file
    widget& operator=(const widget&) = delete;
};
```
实现(widget.cpp)
```cpp
class widget::impl {
    int n; // private data
public:
    void draw(const widget& w) { /* ... */ }
    impl(int n) : n(n) {}
};
void widget::draw() { pimpl->draw(*this); }
widget::widget(int n) : pimpl{std::make_unique<impl>(n)} {}
widget::widget(widget&&) = default;
widget::~widget() = default;
widget& widget::operator=(widget&&) = default;
```

<br/>
### **I.30:封装反规则的行为**

**Example**
```cpp
bool owned;
owner<istream*> inp;
switch (source) {
case std_in:        owned = false; inp = &cin;                       break;
case command_line:  owned = true;  inp = new istringstream{argv[2]}; break;
case file:          owned = true;  inp = new ifstream{argv[2]};      break;
}
istream& in = *inp;
```
没有初始化变量，违反所有权规则，而且还必须记得
```cpp
if (owned) delete inp;
```
```cpp
class Istream { [[gsl::suppress(lifetime)]]
public:
    enum Opt { from_line = 1 };
    Istream() { }
    Istream(zstring p) : owned{true}, inp{new ifstream{p}} {}            // read from file
    Istream(zstring p, Opt) : owned{true}, inp{new istringstream{p}} {}  // read from command line
    ~Istream() { if (owned) delete inp; }
    operator istream&() { return *inp; }
private:
    bool owned = false;
    istream* inp = &cin;
};
```

<br/>
# **F:函数**

函数规则摘要：

函数定义规则：
- [F.1:将有意义的操作打包成精心命名的函数](#f1将有意义的操作打包成精心命名的函数)
- [F.2:一个函数应该执行一个单一的逻辑操作](#f2一个函数应该执行一个单一的逻辑操作)
- [F.3:函数应该短小简单](#f3函数应该短小简单)
- [F.4:如果函数能在编译时求值，那应该声明为constexpr](#f4如果函数能在编译时求值那应该声明为constexpr)
- [F.5:如果函数非常小，而且时间要求很高，那应该声明为inline](#f5如果函数非常小而且时间要求很高那应该声明为inline)
- [F.6:如果函数不会抛出异常，那应该声明为noexcept](#f6如果函数不会抛出异常那应该声明为noexcept)
- [F.7:一般情况下，传T\*或T&，而不是智能指针](#f7一般情况下传t或t而不是智能指针)
- [F.8:优先使用纯函数](#f8优先使用纯函数)
- [F.9:未使用的参数应该是未命名的](#f9未使用的参数应该是未命名的)
- [F.10:如果一个操作可以被重复使用，就该给它一个名字](#f10如果一个操作可以被重复使用就该给它一个名字)
- [F.11:如果你只在一个地方需要一个简单的函数对象，那应该使用未命名的lambda](#f11如果你只在一个地方需要一个简单的函数对象那应该使用未命名的lambda)

参数传递表达式规则：
- [F.15:优先使用简单和传统的信息传递方式](#f15优先使用简单和传统的信息传递方式)
- [F.16:传入参数，cheap参数传值，其他则传const引用](#f16传入参数cheap参数传值其他则传const引用)
- [F.17:传入传出参数，传递非const引用](#f17传入传出参数传递非const引用)
- [F.18:对于可以被移动的参数，通过X&&和std::move参数来传递](#f18对于可以被移动的参数通过x和stdmove参数来传递)
- [F.19:对于可以被forward的参数，通过TP&&传递，且只用std::forward来转发参数](#f19对于可以被forward的参数通过tp传递且只用stdforward来转发参数)
- [F.20:对于需要传出的数值，优先通过返回值来输出参数](#f20对于需要传出的数值优先通过返回值来输出参数)
- [F.21:如果需要返回很多数值，优先使用tuple或者struct](#f21如果需要返回很多数值优先使用tuple或者struct)
- [F.60:如果无参数是个有效的选项时，优先选择T\*而不是T&](#f60如果无参数是个有效的选项时优先选择t而不是t)

参数传递语意规则：
- [F.22:使用T*或owner\<T*\>来指定单个对象](#f22使用t或ownert来指定单个对象)
- [F.23:使用non_null\<T\>来表示null不是一个有效值](#f23使用not_nullt来表示null不是一个有效值)
- [F.24:使用span\<T\>或span_p\<T\>来指定一个半开序列](#f24使用spant或span_pt来指定一个半开序列)
- [F.25:使用zstring或not_null\<zstring\>来指定一个C风格字符串](#f25使用zstring或not_nullzstring来指定一个c风格字符串)
- [F.26:需要指针的地方使用unique_ptr\<T\>来转移所有权](#f26需要指针的地方使用unique_ptrt来转移所有权)
- [F.27:使用shared_ptr\<T\>来共享所有权](#f27使用shared_ptrt来共享所有权)

返回值语意规则：
- [F.42:返回T\*来表示一个位置（仅仅）](#f42返回t来表示一个位置仅仅)
- [F.43:绝不返回指向局部对象的指针或引用](#f43绝不返回指向局部对象的指针或引用)
- [F.44:当复制不可取，且不需要返回无对象时，返回T&](#f44当复制不可取且不需要返回无对象时返回T)
- [F.45:不要返回T&&](#f45不要返回t)
- [F.46:main函数返回类型为int](#f46main函数返回类型为int)
- [F.47:赋值运算符返回T&](#f47赋值运算符返回t)
- [F.48:不要返回std::move(local)](#f48不要返回stdmovelocal)

其他函数规则：
- [F.50:当函数不能做的时候（捕捉局部变量，编写局部函数），使用lambda](#f50当函数不能做的时候捕捉局部变量编写局部函数使用lambda)
- [F.51:如果可以选择，首选默认参数而不是重载](#f51如果可以选择首选默认参数而不是重载)
- [F.52:倾向于在局部使用的lambdas中通过引用来捕获，包括传递给算法](#f52倾向于在局部使用的lambdas中通过引用来捕获包括传递给算法)
- [F.53:避免在将被非本地使用的lambdas中通过引用来捕获，包括返回、存储在堆上或传递给另一个线程。](#f53避免在将被非本地使用的lambdas中通过引用来捕获包括返回存储在堆上或传递给另一个线程)
- [F.54:如果要捕捉this，显式捕获所有变量（没有默认捕获）](#f54如果要捕捉this显式捕获所有变量没有默认捕获)
- [F.55:不要使用va\_arg参数](#f55不要使用va_arg参数)
- [F.56:避免不必要的条件嵌套](#f56避免不必要的条件嵌套)

<br/>
### **F.1:将有意义的操作打包成精心命名的函数**

<br/>
### **F.2:一个函数应该执行一个单一的逻辑操作**

**Example**
```cpp
void read_and_print()    // bad
{
    int x;
    cin >> x;
    // check for errors
    cout << x << "\n";
}
```
这是一个特定输入关联的单体，且永远没有其他用途。可以把函数分解成合适的逻辑部分并进行参数化。
```cpp
int read(istream& is)    // better
{
    int x;
    is >> x;
    // check for errors
    return x;
}

void print(ostream& os, int x)
{
    os << x << "\n";
}
```
```cpp
void read_and_print()
{
    auto x = read(cin);
    print(cout, x);
}
```
如果有更进一步需求，可以将read和print模版化。
```cpp
auto read = [](auto& input, auto& value)    // better
{
    input >> value;
    // check for errors
};

auto print(auto& output, const auto& value)
{
    output << value << "\n";
}
```

<br/>
### **F.3:函数应该短小简单**

<br/>
### **F.4:如果函数能在编译时求值，那应该声明为constexpr**

**原因** constexpr告诉编译器，允许编译期间估值。

**Example**
```cpp
constexpr int fac(int n)
{
    constexpr int max_exp = 17;      // constexpr enables max_exp to be used in Expects
    Expects(0 <= n && n < max_exp);  // prevent silliness and overflow
    int x = 1;
    for (int i = 2; i <= n; ++i) x *= i;
    return x;
}
```
这是C++14，对于C++11，使用fac()的递归表述。

**注意** constexpr并不保证编译期间肯定会被估值；它只是保证在编译期间可以被优化为可估值，如果程序员需要或者编译器决定优化情况下。
```cpp
constexpr int min(int x, int y) { return x < y ? x : y; }

void test(int v)
{
    int m1 = min(-1, 2);            // probably compile-time evaluation
    constexpr int m2 = min(-1, 2);  // compile-time evaluation
    int m3 = min(-1, v);            // run-time evaluation
    constexpr int m4 = min(-1, v);  // error: cannot evaluate at compile time
}
```

<br/>
### **F.5:如果函数非常小，而且时间要求很高，那应该声明为inline**

**原因** 有些编译器没有程序员提示下也很擅长inline，但不能依赖它。过去40年编译器承诺在没有人类提示下，可以比人类更好地inline，我们仍然在等待。\
显式指出inline或类内成员函数（隐式的）可以让编译器做的更好。

**Example**
```cpp
inline string cat(const string& s, const string& s2) { return s + s2; }
```

**例外** 除非确定不会改变，否则不要把inline函数放在本应是一个稳定的接口中。内联函数是ABI的一部分。

**注意** constexpr意味着内联。

**注意** 类中的成员函数默认为内联。

**例外** 函数模版（包括类模版成员函数A<T>::function()和成员函数模版A::function<T>()）通常定义在头文件中，是内联的。

<br/>
### **F.6:如果函数不会抛出异常，那应该声明为noexcept**

**原因** 如果一个异常不应该被抛出，那就假定程序不能应对这个错误，那程序应该尽快终止。声明一个函数为noexcept可以让优化器减少替代执行路径数量。\
还能加快失败后的退出处理速度。

**Example** 所有用C语言或其他不支持except语言的代码都应该标为noexcept，C++标准库对C标准库中的所有函数都隐式地noexcept。

**注意** constexpr函数在运行时也可能抛出异常，所以可能需要对其中一些函数设置条件性的noexcept。

**Example** 甚至你可以对可能抛出异常的函数标记为noexcept
```cpp
vector<string> collect(istream& is) noexcept
{
    vector<string> res;
    for (string s; is >> s;)
        res.push_back(s);
    return res;
}
```
如果collect耗尽内存，程序应该终止。除非程序在内存耗尽下也要存活，否则这可能是正确的做法，terminate()可能会产生合适的错误日志信息，但是\
在内存耗尽后很难再做什么聪明的事情了。

**注意** 在决定是否给一个函数标记noexcept时，你必须意识到你的代码所运行的执行环境，特别是因为抛出和分配的问题。那些打算完全通用的代码（比如标准库和其他类似的实用代码）需要支持那些可以有意义地处理bad\_alloc异常的环境。然而，大多数程序和执行环境不能有意义地处理分配失败，在这些情况下，中止程序是对分配失败最干净和最简单的反应。如果你知道你的程序代码不能对分配失败做出反应，那么即使在分配的函数上添加noexcept也是合适的。
换个角度来说。在大多数程序中，大多数函数都可以抛出（例如，因为它们使用了new，调用了可以抛出的函数，或者使用了通过抛出报告失败的库函数），所以不要不考虑是否可以处理可能的异常就到处撒noexcept。

noexcept对于经常使用的low-level函数是最有用的（也是最明显正确的）。

**注意** 析构函数、swap、move函数以及默认构造函数等不应该抛出异常

<br/>
### **F.7:一般情况下，传T\*或T&，而不是智能指针**

**原因** 传递智能指针可以转移或共享所有权，只有在使用所有权语义的时候才使用智能指针。一个不操作生命期的函数应该传裸指针或引用来代替。

传递一个共享的智能指针，意味着runtime时的开销。

**Example**
```cpp
// accepts any int*
void f(int*);

// can only accept ints for which you want to transfer ownership
void g(unique_ptr<int>);

// can only accept ints for which you are willing to share ownership
void g(shared_ptr<int>);

// doesn't change ownership, but requires a particular ownership of the caller
void h(const unique_ptr<int>&);

// accepts any int
void h(int&);
```

**Example,bad**
```cpp
// callee
void f(shared_ptr<widget>& w)
{
    // ...
    use(*w); // only use of w -- the lifetime is not used at all
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(my_widget);

widget stack_widget;
f(stack_widget); // error
```

**Example,good**
```cpp
// callee
void f(widget& w)
{
    // ...
    use(w);
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(*my_widget);

widget stack_widget;
f(stack_widget); // ok -- now this works
```

**注意** 我们可以静态捕获许多常见的悬空指针情况。函数参数自然在函数调用内生命期安全。

<br/>
### **F.8:优先使用纯函数**

**Example**
```cpp
template<class T>
auto square(T t) { return t * t; }
```

<br/>
### **F.9:未使用的参数应该是未命名的**

**原因** 可读性。抑制未使用变量编译警告。

**Example**
```cpp
widget* find(const set<widget>& s, const widget& w, Hint);   // once upon a time, a hint was used
```

**注意**
1980年代开始引入允许参数不具名就是为了解决这个问题。
如果参数是有条件不使用的，就用[[may\_unused]]属性声明它们。
```cpp
template <typename Value>
Value* find(const set<Value>& s, const Value& v, [[maybe_unused]] Hint h)
{
    if constexpr (sizeof(Value) > CacheSize)
    {
        // a hint is used only if Value is of a certain size
    }
}
```

<br/>
### **F.10:如果一个操作可以被重复使用，就该给它一个名字**

**Example**
```cpp
struct Rec {
    string name;
    string addr;
    int id;         // unique identifier
};

bool same(const Rec& a, const Rec& b)
{
    return a.id == b.id;
}

vector<Rec*> find_id(const string& name);    // find all records for "name"

auto x = find_if(vr.begin(), vr.end(),
    [&](Rec& r) {
        if (r.name.size() != n.size()) return false; // name to compare to is in n
        for (int i = 0; i < r.name.size(); ++i)
            if (tolower(r.name[i]) != tolower(n[i])) return false;
        return true;
    }
);
```

```cpp
bool compare_insensitive(const string& a, const string& b)
{
    if (a.size() != b.size()) return false;
    for (int i = 0; i < a.size(); ++i) if (tolower(a[i]) != tolower(b[i])) return false;
    return true;
}

auto x = find_if(vr.begin(), vr.end(),
    [&](Rec& r) { compare_insensitive(r.name, n); }
);
```
或者避免n参数的隐式绑定：
```cpp
auto cmp_to_n = [&n](const string& a) { return compare_insensitive(a, n); };

auto x = find_if(vr.begin(), vr.end(),
    [](const Rec& r) { return cmp_to_n(r.name); }
);
```

<br/>
## **F.11:如果你只在一个地方需要一个简单的函数对象，那应该使用未命名的lambda**

```cpp
auto earlyUsersEnd = std::remove_if(users.begin(), users.end(),
                                    [](const User &a) { return a.id > 100; });
```

<br/>
### **F.15:优先使用简单和传统的信息传递方式**

<br/>
### **F.16:传入参数，cheap参数传值，其他则传const引用**

**原因** 两个都让调用者知道传入参数不会被修改，且都允许通过右值初始化。

什么是便宜的拷贝取决于机器结构，但是两三个word，比如double，指针，引用最好传值。当复制比较便宜，通过复制代码检查且安全，对于小对象（最多两三个word），它也比通过引用传递更快。因为在函数中不需要额外的间接访问。

**Example**
```cpp
void f1(const string& s);  // OK: pass by reference to const; always cheap

void f2(string s);         // bad: potentially expensive

void f3(int x);            // OK: Unbeatable

void f4(const int& x);     // bad: overhead on access in f4()
```
对于高阶用途（仅仅），对于input-only参数确实需要对右值进行优化。
- 如果函数无条件的从参数move过来，那么传递T&&。
- 如果函数需要保留一个参数的副本，除了用const T&传递，还可以增加一个重载，用&&传递参数（右值），并在函数内用std::move。
- 特殊情况下，input+copy参数可以使用perfect forwarding。

**Example**
```cpp
int multiply(int, int); // just input ints, pass by value

// suffix is input-only but not as cheap as an int, pass by const&
string& concatenate(string&, const string& suffix);

void sink(unique_ptr<widget>);  // input only, and moves ownership of the widget
```
避免使用 "深奥的技术"，例如将参数作为T&&传递，"以提高效率"。大多数关于通过&&传递的性能优势的传言都是错误的，或者说是脆弱的（但见F.18和F.19）。

**注意** 一个引用可以被认为是指向一个有效对象（语言规则），不存在合法的空引用。如果你需要一个optional值，可以使用指针，std::optional，或者一个用于表示“无值”的特殊值。

<br/>
### **F.17:传入传出参数，传递非const引用**

**原因** 让调用者清楚地知道对象是可修改的。

**Example**
```cpp
void update(Record& r);  // assume that update writes to r
```
一些用户定义的和标准库的类型，如span<T>或迭代器，复制起来很便宜，可以通过值传递，而这样做具有可变（in-out）的引用语义。
```cpp
void increment_all(span<int> a)
{
  for (auto&& e : a)
    ++e;
}
```

<br/>
### **F.18:对于可以被移动的参数，通过X&&和std::move参数来传递**

**原因** 高效且消除调用栈错误。X&&与右值绑定，如果传入左值，那么需要显式std::move转换为右值。

**Example**
```cpp
void sink(vector<int>&& v)  // sink takes ownership of whatever the argument owned
{
    // usually there might be const accesses of v here
    store_somewhere(std::move(v));
    // usually no more use of v here; it is moved-from
}
```

**例外** unique\_ptr仅可move，且move也很cheap，但它可以通过传值达到同样的效果。传值确实会产生一个额外的move操作，但是更简单和清晰。
```cpp
template<class T>
void sink(std::unique_ptr<T> p)
{
    // use p ... possibly std::move(p) onward somewhere else
}   // p gets destroyed
``` 

**Enforcement**
- 所有传入X&&参数（X不是模版参数名字），函数内没有std::move该参数都应该被标记。
- 所有对已被move走的对象访问都应该被标记出。
- 不要有条件地move对象。

<br/>
### **F.19:对于可以被forward的参数，通过TP&&传递，且只用std::forward来转发参数**

**原因** 如果对象被传递给其他代码，而不是直接被这个函数调用，我们要让函数参数跟const和右值无关。
在这种情况下，也只有在这种情况下，参数TP&&，TP为模版参数，它即忽略又保持了const以及右值。因此任何代码使用TP&&，隐式地声明它自己不关心变量的const和右值性（因为它自己忽略掉），但是
它负责转发const和右值性给其他代码用（因为它保持了这些属性）。一个TP&&类型参数在函数体内一般通过std::forward来传递属性。

**Example**
```cpp
template<class F, class... Args>
inline auto invoke(F f, Args&&... args)
{
    return f(forward<Args>(args)...);
}

??? calls ???
```

<br/>
### **F.20:对于需要传出的数值，优先通过返回值来输出参数**

**Example**
```cpp
// OK: return pointers to elements with the value x
vector<const int*> find_all(const vector<int>&, int x);

// Bad: place pointers to elements with value x in-out
void find_all(const vector<int>&, vector<const int*>& out, int x);
```

**注意** 不建议返回一个常量，这样的老建议已经过时，它没有增加价值，却干扰了移动语义。
```cpp
const vector<int> fct();    // bad: that "const" is more trouble than it is worth

void g(vector<int>& vx)
{
    // ...
    fct() = vx;   // prevented by the "const"
    // ...
    vx = fct(); // expensive copy: move semantics suppressed by the "const"
    // ...
}
```

**例外**
- 对于非具体类型，比如继承链路上的类型，通过shared\_ptr或shared\_ptr返回对象。
- 如果一个类型的移动是昂贵的，比如array<BigPOD>，可以考虑从堆中分配，并返回一个句柄，比如unique\_ptr。或者传递非const引用，然后来填充。
- 在内循环中多次调用函数时重用携带容量的对象（string，vector），应将其视为输入/输出参数，传引用。

**Example**
```cpp
Matrix operator+(const Matrix& a, const Matrix& b)
{
    Matrix res;
    // ... fill res with the sum ...
    return res;
}

Matrix x = m1 + m2;  // move constructor

y = m3 + m3;         // move assignment
```
```cpp
struct Package {      // exceptional case: expensive-to-move object
    char header[16];
    char load[2024 - 16];
};

Package fill();       // Bad: large return value
void fill(Package&);  // OK

int val();            // OK
void val(int&);       // Bad: Is val reading its argument
```

<br/>
### **F.21:如果需要返回很多数值，优先使用tuple或者struct**

**Example**
```cpp
// BAD: output-only parameter documented in a comment
int f(const string& input, /*output only*/ string& output_data)
{
    // ...
    output_data = something();
    return status;
}

// GOOD: self-documenting
tuple<int, string> f(const string& input)
{
    // ...
    return {status, something()};
}
```
C++98就已经可以支持这种风格，因为pair就是两个元素的tuple。比如：
```cpp
// C++98
result = my_set.insert("Hello");
if (result.second) do_something_with(result.first);    // workaround
```
C++11之后可以这样写：
```cpp
Sometype iter;                                // default initialize if we haven't already
Someothertype success;                        // used these variables for some other purpose

tie(iter, success) = my_set.insert("Hello");   // normal return value
if (success) do_something_with(iter);
```
C++17开始的结构绑定：
```cpp
if (auto [ iter, success ] = my_set.insert("Hello"); success) do_something_with(iter);
```

许多情况下，返回一个特定的、用户定义的类型可能是有用的。
```cpp
struct Distance {
    int value;
    int unit = 1;   // 1 means meters
};

Distance d1 = measure(obj1);        // access d1.value and d1.unit
auto d2 = measure(obj2);            // access d2.value and d2.unit
auto [value, unit] = measure(obj3); // access value and unit; somewhat redundant
                                    // to people who know measure()
auto [x, y] = measure(obj4);        // don't; it's likely to be confusing
```
只有返回的值代表独立的实体，而不是一个抽象时，才返回pair或tuple。

<br/>
### **F.60:如果无参数是个有效的选项时，优先选择T\*而不是T&**

**原因** 指针T\*可能为空，但引用不会，空引用是无效的。有时用nullptr来表示一个概念不存在对象，如果没有此类需求，引用会更简单，更有效。

**Example**
```cpp
string zstring_to_string(zstring p) // zstring is a char*; that is a C-style string
{
    if (!p) return string{};    // p might be nullptr; remember to check
    return string{p};
}

void print(const vector<int>& r)
{
    // r refers to a vector<int>; no check needed
}
```

**注意**
```cpp
T* p = nullptr; T& r = *p;
```
这种不常见但却可能发生，这不是有效的C++。

**注意** 如果更喜欢用指针，那么用not\_null<T*>来提供跟T&一样的保证。

<br/>
### **F.22:使用T\*或owner\<T*\>来指定单个对象**

**原因** 可读性：它仅仅清晰地表示一个简单指针。可以实现重要的工具支持。

**注意** 传统的C和C++代码，普通的T\*被用于表达许多弱相关目的：
- 识别一个（单一）对象，不会被这个函数delete。
- 指向一个在free store分配的对象（并在之后delete）。
- 持有nullptr。
- 识别一个C风格的字符串（以零结尾的字符串）。
- 识别一个指定长度的数组。
- 识别数组中一个位置。
这些让人们很难理解，也让检查和工具支持复杂化。

**Example**
```cpp
void use(int* p, int n, char* s, int* q)
{
    p[n - 1] = 666; // Bad: we don't know if p points to n elements;
                    // assume it does not or use span<int>
    cout << s;      // Bad: we don't know if that s points to a zero-terminated array of char;
                    // assume it does not or use zstring
    delete q;       // Bad: we don't know if *q is allocated on the free store;
                    // assume it does not or use owner
}
```
better:
```cpp
void use2(span<int> p, zstring s, owner<int*> q)
{
    p[p.size() - 1] = 666; // OK, a range error can be caught
    cout << s; // OK
    delete q;  // OK
}
```

<br/>
### **F.23:使用not_null\<T\>来表示null不是一个有效值**

**原因** 明确性。一个带not\_null\<T\>参数的函数清楚地表明，函数调用者应该负责对于nullptr检查。一个返回值为not\_null\<T\>的函数清楚地表明，函数调用者不需要检查nullptr。

**Example** not\_null\<T\>使得读者（人类或机器）清楚，解引用前不需要检查nullptr。此外，在调试时，owner\<T\*\>和not\_null\<T\>可以用来检查正确性。
```cpp
int length(Record* p);
```
调用length函数是否需要先检查p是否为空？length的实现是否需要检查p为空？
```cpp
// it is the caller's job to make sure p != nullptr
int length(not_null<Record*> p);

// the implementor of length() must assume that p == nullptr is possible
int length(Record* p);
```

**注意** 一个not\_null\<T\>被认为不是nullptr，一个T\*可能是nullptr，两者都可以在内存中表示为T\*，（没有运行时隐式开销）

**注意** not\_null\<T\>不仅仅适用于内置指针，也适用于unique\_ptr、shared\_ptr和其他类似指针类型。

<br/>
### **F.24:使用span\<T\>或span_p\<T\>来指定一个半开序列**

**原因** 非正式或非明确的范围是错误的来源。

**Example**
```cpp
X* find(span<X> r, const X& v);    // find v in r

vector<X> vec;
// ...
auto p = find({vec.begin(), vec.end()}, X{});  // find X{} in vec
```
range在C++中非常常见，通常情况下，它是隐式的，正确性很难保证。特别是，一对参数(p, n)指定一个数组[p:p+n)，一般来说，不能知道p之后是否有n个元素可以访问。\
span<T>和span_p<T>是简单的辅助类，分别指定一个[p:q)和一个以p开始以predicate为真结束的范围。

**Example**
```cpp
void f(span<int> s)
{
    // range traversal (guaranteed correct)
    for (int x : s) cout << x << '\n';

    // C-style traversal (potentially checked)
    for (gsl::index i = 0; i < s.size(); ++i) cout << s[i] << '\n';

    // random access (potentially checked)
    s[7] = 9;

    // extract pointers (potentially checked)
    std::sort(&s[0], &s[s.size() / 2]);
}
```

**注意** span不拥有它的元素，且非常小，可以传值。
传递span就像传递两个指针或者传递一个指针和一个int一样高效。

<br/>
### **F.25:使用zstring或not_null\<zstring\>来指定一个C风格字符串**

**原因** C风格字符串无处不在。照惯例定义，它们是以零结尾的字符数组。我们需要将C风格的字符串与指向单个字符的指针或指向字符数组的老式指针区分开。
如果不需要以空结尾，使用string\_view。

**Example**
```cpp
int length(const char* p);
```
调用length时，是否先检查p为空？length的实现是否需要先检查p为空？
```cpp
// the implementor of length() must assume that p == nullptr is possible
int length(zstring p);

// it is the caller's job to make sure p != nullptr
int length(not_null<zstring> p);
```

<br/>
### **F.26:需要指针的地方使用unique_ptr\<T\>来转移所有权**

**原因** 使用unique\_ptr是最廉价的保证传指针安全。

**Example**
```cpp
unique_ptr<Shape> get_shape(istream& is)  // assemble shape from input stream
{
    auto kind = read_header(is); // read header and identify the next shape on input
    switch (kind) {
    case kCircle:
        return make_unique<Circle>(is);
    case kTriangle:
        return make_unique<Triangle>(is);
    // ...
    }
}
```

**注意** 如果你通过接口传递的是一个类层次结构中的对象，那么应该传指针而不是传对象。

<br/>
### **F.27:使用shared_ptr\<T\>来共享所有权**

**Example**
```cpp
shared_ptr<const Image> im { read_image(somewhere) };

std::thread t0 {shade, args0, top_left, im};
std::thread t1 {shade, args1, top_right, im};
std::thread t2 {shade, args2, bottom_left, im};
std::thread t3 {shade, args3, bottom_right, im};

// detach threads
// last thread to finish deletes the image
```

<br/>
### **F.42:返回T\*来表示一个位置（仅仅）**

**原因** 这就是指针的用处。返回T\*来表示所有权转移是误用。

**Example**
```cpp
Node* find(Node* t, const string& s)  // find s in a binary tree of Nodes
{
    if (!t || t->name == s) return t;
    if ((auto p = find(t->left, s))) return p;
    if ((auto p = find(t->right, s))) return p;
    return nullptr;
}
```
find返回如果不为空，返回的指针表示一个持有s的Node，重要的是，这并不意味着指向对象的所有权转移给了调用者。

**注意** 位置可以通过迭代器、索引和引用来转移。如果不需要使用nullptr或者被引用的对象没有改变，那么引用是比指针更好的选择。

<br/>
### **F.43:绝不返回指向局部对象的指针或引用**

**原因** 避免这种悬空指针导致崩溃和数据损坏。

**Example,bad**
```cpp
int* f()
{
    int fx = 9;
    return &fx;  // BAD
}

void g(int* p)   // looks innocent enough
{
    int gx;
    cout << "*p == " << *p << '\n';
    *p = 999;
    cout << "gx == " << gx << '\n';
}

void h()
{
    int* p = f();
    int z = *p;  // read from abandoned stack frame (bad)
    g(p);        // pass pointer to abandoned stack frame to function (bad)
}
```
这里一个常见的实现得到的结果是
```cpp
*p == 999
gx == 999
```
我认为是因为g()重用了f()所丢弃的栈空间，所以\*p指向被gx占用的空间。
- 设想一下，fx和gx类型不同会发生什么？
- 设想一下，fx和gx是一个具有不可变的类型会发生什么？
- 设想一下，更多的悬空指针在更大范围的函数集中传递，会发生什么？
- 设想一下，破解者可以用这个悬空指针做什么？
幸运的是，绝大部分现代编译器能捕获并警告这种简单情况。

引用也一样。
```cpp
int& f()
{
    int x = 7;
    // ...
    return x;  // Bad: returns reference to object that is about to be destroyed
}
```

**注意** 这只适用于非static局部变量。所有static变量都是静态分配的，所以它们的指针不会悬空。

**Example,bad**
```cpp
int* glob;       // global variables are bad in so many ways

template<class T>
void steal(T x)
{
    glob = x();  // BAD
}

void f()
{
    int i = 99;
    steal([&] { return &i; });
}

int main()
{
    f();
    cout << *glob << '\n';
}
```
指向局部变量的指针泄漏并非都很明显。

<br/>
### **F.44:当复制不可取，且不需要返回无对象时，返回T&**

**Example**
```cpp
class Car
{
    array<wheel, 4> w;
    // ...
public:
    wheel& get_wheel(int i) { Expects(i < w.size()); return w[i]; }
    // ...
};

void use()
{
    Car c;
    wheel& w0 = c.get_wheel(0); // w0 has the same lifetime as c
}
```

<br/>
### **F.45:不要返回T&&**

**原因** T&&要求返回一个被销毁的临时对象的引用，&&是临时对象的一个磁铁。

**Example,bad**
一个右值引用返回值在表达式结束后，就离开了其生命期。
```cpp
auto&& x = max(0, 1);   // OK, so far
foo(x);                 // Undefined behavior
```

对于传入参数（通过普通引用或完美转发）并希望返回值的函数，使用简单的自动返回类型推导（不是自动&&）。
```cpp
template<class F>
auto&& wrapper(F f)
{
    log_call(typeid(f)); // or whatever instrumentation
    return f();          // BAD: returns a reference to a temporary
}
```
better:
```cpp
template<class F>
auto wrapper(F f)
{
    log_call(typeid(f)); // or whatever instrumentation
    return f();          // OK
}
```

**例外** std::move和std::forward确实也返回&&，但它们仅仅做cast。

<br/>
### **F.46:main函数返回类型为int**

**Example**
```cpp
    void main() { /* ... */ };  // bad, not C++

    int main()
    {
        std::cout << "This is the way to do it\n";
    }
```
尽管main函数的返回值不是void，但它不需要明确的返回语句。为什么要提出这一节，因为社区里有很多误用。

<br/>
### **F.47:赋值运算符返回T&**

**原因** 操作符重载惯例是由operator=(const T&)来执行赋值，然后返回非const的\*this，这确保了与标准库一致，并遵循照着int类型一致。

**注意** 历史上有一些建议赋值操作符返回const T&，主要是避免像(a = b) = c这种情况。这种代码不常见，不应该破坏与标准库一致的做法。

**Example**
```cpp
class Foo
{
 public:
    ...
    Foo& operator=(const Foo& rhs)
    {
      // Copy members.
      ...
      return *this;
    }
};
```

<br/>
### **F.48:不要返回std::move(local)**

**原因** 为保证拷贝优化，在返回语句中使用std::move几乎总是悲观的做法。 

**Example,bad**
```cpp
S f()
{
  S result;
  return std::move(result);
}
```

**Example,good**
```cpp
S f()
{
  S result;
  return result;
}
```

<br/>
### **F.50:当函数不能做的时候（捕捉局部变量，编写局部函数），使用lambda**

**原因** 函数不能捕获局部变量，也不能在局部范围内定义。如果需要这些，尽可能选择lambda，如果不需要，就选择手写的函数对象。另外，
lambda和函数对象不能重载，如果你需要重载，最好使用函数。

**Example**
```cpp
// writing a function that should only take an int or a string
// -- overloading is natural
void f(int);
void f(const string&);

// writing a function object that needs to capture local state and appear
// at statement or expression scope -- a lambda is natural
vector<work> v = lots_of_work();
for (int tasknum = 0; tasknum < max; ++tasknum) {
    pool.run([=, &v] {
        /*
        ...
        ... process 1 / max - th of v, the tasknum - th chunk
        ...
        */
    });
}
pool.join();
```

<br/>
### **F.51:如果可以选择，首选默认参数而不是重载**

**原因** 默认参数只提供了一个单一实现的替代接口，一组重载函数不能保证都实现了同一语意，使用默认参数可以避免代码复制。

**Example**
```cpp
void print(const string& s, format f = {});
```
与之对应的是
```cpp
void print(const string& s);  // use default format
void print(const string& s, format f);
```
当一组函数用来做语意相同的操作时，就没有选择。
```cpp
void print(const char&);
void print(int);
void print(zstring);
```

<br/>
### **F.52:倾向于在局部使用的lambdas中通过引用来捕获，包括传递给算法**

**原因** 为了效率以及正确性，使用局部lambda时，你几乎总是想通过引用来捕获。包括编写及调用并行算法时，这些算法是局部的，因为它们在返回前都会join。

**Example**
在这里一个大型对象被传递给迭代算法，如果复制消息是没有效率的。
```cpp
std::for_each(begin(sockets), end(sockets), [&message](auto& socket)
{
    socket.send(message);
});
```

**Example**
这是一个简单的三个stage并行流水线，每个stage封装了一个线程和一个队列。process函数来加入队列，析构函数自动等待所有队列处理完为空。
```cpp
void send_packets(buffers& bufs)
{
    stage encryptor([](buffer& b) { encrypt(b); });
    stage compressor([&](buffer& b) { compress(b); encryptor.process(b); });
    stage decorator([&](buffer& b) { decorate(b); compressor.process(b); });
    for (auto& b : bufs) { decorator.process(b); }
}  // automatically blocks waiting for pipeline to finish
```

<br/>
### **F.53:避免在将被非本地使用的lambdas中通过引用来捕获，包括返回、存储在堆上或传递给另一个线程。**

**Example,bad**
```cpp
int local = 42;

// Want a reference to local.
// Note, that after program exits this scope,
// local no longer exists, therefore
// process() call will have undefined behavior!
thread_pool.queue_work([&] { process(local); });
```
**Example,good**
```cpp
int local = 42;
// Want a copy of local.
// Since a copy of local is made, it will
// always be available for the call.
thread_pool.queue_work([=] { process(local); });
```

<br/>
### **F.54:如果要捕捉this，显式捕获所有变量（没有默认捕获）**

**原因** 这令人困惑。成员函数中[=]似乎是按值捕获，而实际却是按引用捕获数据成员，因为捕获了一个不可见的this指针，如果真的想捕获this，就明确写出。

**Example**
```cpp
class My_class {
    int x = 0;
    // ...

    void f()
    {
        int i = 0;
        // ...

        auto lambda = [=] { use(i, x); };   // BAD: "looks like" copy/value capture
        // [&] has identical semantics and copies the this pointer under the current rules
        // [=,this] and [&,this] are not much better, and confusing

        x = 42;
        lambda(); // calls use(0, 42);
        x = 43;
        lambda(); // calls use(0, 43);

        // ...

        auto lambda2 = [i, this] { use(i, x); }; // ok, most explicit and least confusing

        // ...
    }
};
```

<br/>
### **F.55:不要使用va_arg参数**

**Example**
```cpp
int sum(...)
{
    // ...
    while (/*...*/)
        result += va_arg(list, int); // BAD, assumes it will be passed ints
    // ...
}

sum(3, 2); // ok
sum(3.14159, 2.71828); // BAD, undefined

template<class ...Args>
auto sum(Args... args) // GOOD, and much more flexible
{
    return (... + args); // note: C++17 "fold expression"
}

sum(3, 2); // ok: 5
sum(3.14159, 2.71828); // ok: ~5.85987
```

**注意** 声明...参数有时对不涉及实际参数传递的技术很有用，特别是声明"take-anything"函数，以便在重载函数集中禁用"everything else"，或者在模版元编程中表达一种全面情况。

<br/>
### **F.56:避免不必要的条件嵌套**

**原因** 浅层嵌套的条件语句更容易理解，努力将重要的代码放在最外层范围。

**Example**
```cpp
// Bad: Deep nesting
void foo() {
    ...
    if (x) {
        computeImportantThings(x);
    }
}

// Bad: Still a redundant else.
void foo() {
    ...
    if (!x) {
        return;
    }
    else {
        computeImportantThings(x);
    }
}

// Good: Early return, no redundant else
void foo() {
    ...
    if (!x)
        return;

    computeImportantThings(x);
}
```

**Example**
```cpp
// Bad: Unnecessary nesting of conditions
void foo() {
    ...
    if (x) {
        if (y) {
            computeImportantThings(x);
        }
    }
}

// Good: Merge conditions + return early
void foo() {
    ...
    if (!(x && y))
        return;

    computeImportantThings(x);
}
```

<br/>
# **C:类和类的层次结构**

类是用户定义的类型，程序员可以为其定义表示、操作和接口。类层次结构将相关类组织成层次结构。

类的规则摘要：
- [C.1:将相关数据组织成class或struct](#c1将相关数据组织成class或struct)
- [C.2:如果类有一个不变量，则使用class；如果数据成员可以独立变化，则使用struct](#c2如果类有一个不变量则使用class如果数据成员可以独立变化则使用struct)
- [C.3:用一个类来表示接口和实现的区别](#c3用一个类来表示接口和实现的区别)
- [C.4:只有当一个函数需要直接访问一个类的表示时，才让它成为成员](#c4只有当一个函数需要直接访问一个类的表示时才让它成为成员)
- [C.5:将辅助函数放在与它们支持的类相同的命名空间中](#c5将辅助函数放在与它们支持的类相同的命名空间中)
- [C.7:不要在同一语句中定义一个类或枚举并声明其类型的变量](#c7不要在同一语句中定义一个类或枚举并声明其类型的变量)
- [C.8:如果每个成员都不是public，那么应该用class而不是struct](#c8如果每个成员都不是public那么应该用class而不是struct)
- [C.9:最大限度减少成员暴露](#c9最大限度减少成员暴露)

分节：
- [C.concrete:实体类型](#cconcrete实体类型)
- [C.ctor:构造、赋值和析构](#cctor构造赋值和析构)
- [C.con:容器及其它资源句柄](#ccon容器及其它资源句柄)
- [C.lambdas:函数对象和lambda](#clambdas函数对象和lambda)
- [C.hier:类分层(OOP)](#chier类的分层oop)
- [C.over:重载运算符](#cover重载运算符)
- [C.union:Unions](#cunionunions)

<br/>
### **C.1:将相关数据组织成class或struct**

**Example**
```cpp
void draw(int x, int y, int x2, int y2);  // BAD: unnecessary implicit relationships
void draw(Point from, Point to);          // better
```
<br/>
### **C.2:如果类有一个不变量，则使用class；如果数据成员可以独立变化，则使用struct**

**原因** 可读性。易于理解。类的使用提醒程序员需要一个不变量。这是一个有用的惯例。

**Example**
```cpp
struct Pair {  // the members can vary independently
    string name;
    int volume;
};
```
```cpp
class Date {
public:
    // validate that {yy, mm, dd} is a valid date and initialize
    Date(int yy, Month mm, char dd);
    // ...
private:
    int y;
    Month m;
    char d;    // day
};
```

**注意** 如果一个类有任何私有数据，用户不使用构造函数就不能完全初始化一个对象。因此，类的定义者将提供一个构造函数，并且必须指定其含义。这实际上意味着定义者需要定义一个不变量。

<br/>
### **C.3:用一个类来表示接口和实现的区别**

**原因** 明确区分接口和实现，可以提高可读性并简化维护。

**Example**
```cpp
class Date {
public:
    Date();
    // validate that {yy, mm, dd} is a valid date and initialize
    Date(int yy, Month mm, char dd);

    int day() const;
    Month month() const;
    // ...
private:
    // ... some representation ...
};
```
比如，我们可以修改Date的表示，而不影响用户，当然代码还需要重编。

**注意** 以这种方式使用一个类来表示接口和实现的区别，不是唯一的方法，我们可以在同个命名空间中使用一组独立函数的声明，一个抽象的基类，或者一个带有concept的函数模版来表示一个接口。更重要的是，明确区分一个接口和它的实现细节。理想情况下，通常情况下，一个接口比它的实现稳定多了。

<br/>
### **C.4:只有当一个函数需要直接访问一个类的表示时，才让它成为成员**

**原因** 越少成员函数耦合度越低，越少的函数会越少因为修改了对象状态而引起麻烦。

**Example**
```cpp
class Date {
    // ... relatively small interface ...
};

// helper functions:
Date next_weekday(Date);
bool operator==(Date, Date);
```
这些辅助函数不需要直接访问Date的表示。

**例外** 语言要求虚函数必须是成员函数，不是所有的虚函数都直接访问数据，特别是抽象类的成员很少这样做。

**例外** 语言要求操作符函数=, (), [], ->需要是成员函数。

**例外** 一个重载函数集可以有一些不直接访问私有数据的成员函数。
```cpp
class Foobar {
public:
    void foo(long x) { /* manipulate private data */ }
    void foo(double x) { foo(std::lround(x)); }
    // ...
private:
    // ...
};
```

**例外** 设计为同一链条上处理的函数。
```cpp
x.scale(0.5).rotate(45).set_color(Color::red);
```

<br/>
### **C.5:将辅助函数放在与它们支持的类相同的命名空间中**

**原因** 辅助函数是不需要直接访问类的表示的函数，通常被视为类的有用接口一部分，将他们放在相同命名空间内，使得它们的关系显而易见，
并允许它们通过参数依赖查找可找到。

**Example**
```cpp
namespace Chrono { // here we keep time-related services

    class Time { /* ... */ };
    class Date { /* ... */ };

    // helper functions:
    bool operator==(Date, Date);
    Date next_weekday(Date);
    // ...
}
```

**Enforcement** 如果有全局函数的参数来自于单一的命名空间应该标记出。

<br/>
### **C.7:不要在同一语句中定义一个类或枚举并声明其类型的变量**

**原因** 在同一个声明中混合类型定义和其他实体定义是混乱和不必要的。

**Example,bad**
```cpp
struct Data { /*...*/ } data{ /*...*/ };
```

**Example,good**
```cpp
struct Data { /*...*/ };
Data data{ /*...*/ };
```

<br/>
### **C.8:如果每个成员都不是public，那么应该用class而不是struct**

**原因** 可读性。为了清楚地表明某些东西被隐藏/抽象化了。这是一个有用的惯例。

**Example,bad**
```cpp
struct Date {
    int d, m;

    Date(int i, Month m);
    // ... lots of functions ...
private:
    int y;  // year
};
```
语言规则上这样写没有问题，设计角度看，所有的东西都是错的。私有数据被隐藏在远离公共数据的地方。数据被分割在类声明的不同部分。数据的不同部分有不同的访问权限。所有这些都降低了可读性，并使维护变得复杂。

<br/>
### **C.9:最大限度减少成员暴露**

**原因** 封装。信息隐藏。最大限度地减少非故意访问的机会。这简化了维护工作。

**Example**
```cpp
template<typename T, typename U>
struct pair {
    T a;
    U b;
    // ...
};
```
无论我们在//-部分做什么，一个pair的任意用户可以任意地、独立地改变它的a和b。在一个大的代码库中，我们不容易找到哪个代码对pair的成员做什么。
这可能正是我们想要的，但如果我们想在成员之间强制执行一种关系，我们需要使它们成为私有的，并通过构造函数和成员函数强制执行这种关系（不变量）。
```cpp
class Distance {
public:
    // ...
    double meters() const { return magnitude*unit; }
    void set_unit(double u)
    {
            // ... check that u is a factor of 10 ...
            // ... change magnitude appropriately ...
            unit = u;
    }
    // ...
private:
    double magnitude;
    double unit;    // 1 is meters, 1000 is kilometers, 0.001 is millimeters, etc.
};
```

**Example**
类有两个接口，一个可以被继承类使用(protected)，一个可以被普通对象使用(public)。继承类可能会跳过运行时的check，因为它已经保证了正确性。
```cpp
class Foo {
public:
    int bar(int x) { check(x); return do_bar(x); }
    // ...
protected:
    int do_bar(int x); // do some operation on the data
    // ...
private:
    // ... data ...
};

class Dir : public Foo {
    // ...
    int mem(int x, int y)
    {
        /* ... do something ... */
        return do_bar(x + y); // OK: derived class can bypass check
    }
};

void user(Foo& x)
{
    int r1 = x.bar(1);      // OK, will check
    int r2 = x.do_bar(2);   // error: would bypass check
    // ...
}
```

**注意** protected数据不是一个好主意。

**注意** 一般先public成员再protected，最后private。

<br/>
## **C.concrete:实体类型**

实体类型规则摘要：
- [C.10:优先考虑实体类型而不是类分层](#c10优先考虑实体类型而不是类分层)
- [C.11:让实体类型有规可循](#c11让实体类型有规可循)
- [C.12:不要让数据成员成为常量或引用](#c12不要让数据成员成为常量或引用)

<br/>
### **C.10:优先考虑实体类型而不是类分层**

**原因** 实体类型比分层类简单，容易设计，容易实现，容易使用，小而快，如果要使用分层类则需要个理由。

**Example**
```cpp
class Point1 {
    int x, y;
    // ... operations ...
    // ... no virtual functions ...
};

class Point2 {
    int x, y;
    // ... operations, some virtual ...
    virtual ~Point2();
};

void use()
{
    Point1 p11 {1, 2};   // make an object on the stack
    Point1 p12 {p11};    // a copy

    auto p21 = make_unique<Point2>(1, 2);   // make an object on the free store
    auto p22 = p21->clone();                // make a copy
    // ...
}
```
如果一个类有分层结构，我们必须通过指针或引用来操作它的对象，这意味着更多的内存开销，更多的分配和回收，以及更多的运行时开销来执行这其中的间接性。

**注意** 实体类型可以在栈上分配，并且可以是其他类成员。

<br/>
### **C.11:让实体类型有规可循**

**原因** 常规类型容易理解和推理，非常规的类型需要额外的学习成本来理解和使用。C++内置类型是常规类型，标准库中的类也是，比如string，vector，map等。
实体类型如果没有赋值函数和比较函数是罕见的。

**Example**
```cpp
struct Bundle {
    string name;
    vector<Record> vr;
};

bool operator==(const Bundle& a, const Bundle& b)
{
    return a.name == b.name && a.vr == b.vr;
}

Bundle b1 { "my bundle", {r1, r2, r3}};
Bundle b2 = b1;
if (!(b1 == b2)) error("impossible!");
b2.name = "the other bundle";
if (b1 == b2) error("No!");
```
特别是，如果一个实体类型是可复制的，最好也给配一个比较符函数，来确保a==b同时b==a。

**注意** 如果打算跟C代码交互，给struct定义操作符operator==并不可行。

**注意** 资源句柄类通常是不可复制的，比如scoped\_ptr，mutex等，不能被复制，但是可以move，所以它们不是常规类型，通常设计为move-only。

<br/>
### **C.12:不要让数据成员成为常量或引用**

**原因** 这样做没啥用处，而且很微妙，它是类不能被复制或者部分不能被复制，使得类很难用。

**Example,bad**
```cpp
class bad {
    const int i;    // bad
    string& s;      // bad
    // ...
};
```
拷贝构造函数可行，拷贝赋值函数不可行。

**Enforcement** 成员数据如果是const，&或者&&都应该标记出。

<br/>
## **C.ctor:构造、赋值和析构**
这些函数影响对象生命期，创建、拷贝、移动和析构。定义构造函数保证和简化类的初始化。\
几种常规操作符：
- 默认构造：X()
- 拷贝构造：X(const X&)
- 拷贝赋值：operator=(const X&)
- 移动构造：X(X&&)
- 移动赋值：operator=(X&&)
- 析构函数：~X()

默认情况下，如果使用到，编译器会自动生成对应操作符实现，但也可能会抑制默认的自动生成。\
这些默认操作符是一组相关的函数，它们共同实现了对一个对象的生命期语义。默认情况下，C++将类视为value类型，但并不是所有类型都是value类型。

默认操作符规则：
- [C.20:如果能避免定义任何默认操作，那就不要定义](#c20如果能避免定义任何默认操作那就不要定义)
- [C.21:如果你定义或=delete掉任何拷贝、移动或析构函数，那就定义或=delete所有](#c21如果你定义或delete掉任何拷贝移动或析构函数那就定义或delete所有)
- [C.22:让所有默认操作符保持一致](#c22让所有默认操作符保持一致)

析构规则：
- [C.30:如果对象需要显式操作，那么就要定义一个析构函数](#c30如果对象需要显式操作那么就要定义一个析构函数)
- [C.31:一个类如果构造函数获得资源，那么析构函数应该释放资源](#c31一个类如果构造函数获得资源那么析构函数应该释放资源)
- [C.32:如果一个类有T\*或者T&，那么考虑它是否拥有所有权](#c32如果一个类有t或者t那么考虑它是否拥有所有权)
- [C.33:如果一个类有一个拥有所有权的指针，那么要定义一个析构函数](#c33如果一个类有一个拥有所有权的指针那么要定义一个析构函数)
- [C.35:基类的析构函数应该是public和virtual的，或者是protected和non-virtual的](#c35基类的析构函数应该是public和virtual的或者是protected和non-virtual的)
- [C.36:析构函数不能失败](#c36析构函数不能失败)
- [C.37:析构函数应该noexcept](#c37析构函数应该noexcept)

构造规则：
- [C.40:如果类有一个不变量，那么定义一个构造函数](#c40如果类有一个不变量那么定义一个构造函数)
- [C.41:一个构造函数应该创建一个完全初始化的对象](#c41一个构造函数应该创建一个完全初始化的对象)
- [C.42:如果一个构造函数不能构造一个有效的对象，则抛出一个异常](#c42如果一个构造函数不能构造一个有效的对象则抛出一个异常)
- [C.43:确保一个可复制类有一个默认构造函数](#c43确保一个可复制类有一个默认构造函数)
- [C.44:倾向于默认构造函数是简单的且不会抛异常](#c44倾向于默认构造函数是简单的且不会抛异常)
- [C.45:不要定义一个只初始化数据成员的默认构造函数；使用初始化成员函数](#c45不要定义一个只初始化数据成员的默认构造函数使用初始化成员函数)
- [C.46:默认情况下，单参数构造函数应该explicit声明](#c46默认情况下单参数构造函数应该explicit声明)
- [C.47:按照成员声明的顺序定义和初始化成员变量](#c47按照成员声明的顺序定义和初始化成员变量)
- [C.48:对于常量初始化构造函数，优先选择类内初始化而不是成员初始化](#c48对于常量初始化构造函数优先选择类内初始化而不是成员初始化)
- [C.49:构造函数中优先考虑初始化而不是赋值](#c49构造函数中优先考虑初始化而不是赋值)
- [C.50:如果你在初始化过程中需要virtual行为，请使用factory函数](#c50如果你在初始化过程中需要virtual行为请使用factory函数)
- [C.51:使用代理函数来表示一个类所有构造函数所共有部分](#c51使用代理函数来表示一个类所有构造函数所共有部分)
- [C.52:使用继承构造函数将构造函数导入到派生类中，而不需要进一步显式初始化](#c52使用继承构造函数将构造函数导入到派生类中而不需要进一步显式初始化)

拷贝和移动规则：
- [C.60:拷贝赋值函数应该为non-virtual，用const&传参数，且返回non-const&](#c60拷贝赋值函数应该为non-virtual用const传参数且返回non-const)
- [C.61:一个拷贝操作就应该有拷贝](#c61一个拷贝操作就应该有拷贝)
- [C.62:拷贝赋值操作时应该保证自我赋值安全](#c62拷贝赋值操作时应该保证自我赋值安全)
- [C.63:移动赋值函数应该为non-virtual，用&&传参，且返回non-const&](#c63移动赋值函数应该为non-virtual用传参且返回non-const)
- [C.64:一个移动操作应该发生移动并使其来源处于有效状态](#c64一个移动操作应该发生移动并使其来源处于有效状态)
- [C.65:移动赋值操作时应该保证自我赋值安全](#c65移动赋值操作时应该保证自我赋值安全)
- [C.66:移动操作应该设为noexcept](#c66移动操作应该设为noexcept)
- [C.67:多态类应该抑制public的拷贝或移动](#c67多态类应该抑制public的拷贝或移动)

其他默认操作符规则：
- [C.80:如果你显式使用默认语义的话使用=default](#c80如果你显式使用默认语义的话使用default)
- [C.81:如果你想禁用默认语义且不希望有其他替代的话=delete](#c81如果你想禁用默认语义且不希望有其他替代的话delete)
- [C.82:不要在构造函数和析构函数中调用virtual函数](#c82不要在构造函数和析构函数中调用virtual函数)
- [C.83:对于值类型，考虑提供noexcept的swap函数](#c83对于值类型考虑提供noexcept的swap函数)
- [C.84:swap函数不能失败](#c84swap函数不能失败)
- [C.85:swap函数应该设为noexcept](#c85swap函数应该设为noexcept)
- [C.86:在操作类型和noexcept应该对==有对称性](#c86在操作类型和noexcept应该对有对称性)
- [C.87:谨防在基类上==](#c87谨防在基类上)
- [C.89:hash函数应该设为noexcept](#c89hash函数应该设为noexcept)
- [C.90:依赖构造函数和赋值操作符，而不是memset和memcpy](#c90依赖构造函数和赋值操作符而不是memset和memcpy)

<br/>
### **C.20:如果能避免定义任何默认操作，那就不要定义**

**原因** 这是最简单的，并给出了最简洁的语义。

**Example**
```cpp
struct Named_map {
public:
    // ... no default operations declared ...
private:
    string name;
    map<int, int> rep;
};

Named_map nm;        // default construct
Named_map nm2 {nm};  // copy construct
```
因为string和map包含所有这些特殊函数，所以不需要再额外多做什么。

**注意** 这就是有名的the rule of zero。

**Enforcement** （无法强制执行）。虽然不可强制执行，但一个好的静态分析工具可以发现一些模式，这些模式可以被进一步改进以满足本条规则。
举个例子，如果一个类有成员（指针以及大小），并且有析构函数来释放指针所指对象，那么可以考虑用vector替换。 

<br/>
### **C.21:如果你定义或=delete掉任何拷贝、移动或析构函数，那就定义或=delete所有**

**原因** 拷贝、移动和析构语义是密切相关的，如果其中之一需要被声明，那其他的也可能需要。\
声明任何拷贝、移动、析构函数，即使是=default或=delete，也会抑制移动构造和移动赋值运算符的隐式声明。\
声明一个移动构造函数或移动赋值运算符，即使是=default或=delete，也会导致隐式生成的拷贝构造函数或拷贝赋值运算符函数被定义为delete。
因此，一旦声明了这几个函数中的一个，其他函数都应该被声明，以避免不必要的影响。

**Example,bad**
```cpp
struct M2 {   // bad: incomplete set of copy/move/destructor operations
public:
    // ...
    // ... no copy or move operations ...
    ~M2() { delete[] rep; }
private:
    pair<int, int>* rep;  // zero-terminated set of pairs
};

void use()
{
    M2 x;
    M2 y;
    // ...
    x = y;   // the default assignment
    // ...
}
```
隐式默认生成的拷贝和移动赋值操作符不大可能正确，比如拷贝后两次delete。

**注意** 这就是知名的the rule of five。

**注意** 如果你想要一个编译器自动生成的实现，同时自己定义其他几个函数，那么可以直接=default表明你想让编译器替你实现。
如果你不需要一个默认的生成函数，那么直接=delete来抑制自动生成。

**Example,good** 如果一个析构函数需要声明且为virtual，那么它可以定义为default。
```cpp
class AbstractBase {
public:
    virtual ~AbstractBase() = default;
    // ...
};
```
为了避免C.67所述的切片，应该让拷贝和移动操作符设为protected或=delete，然后再增加一个clone函数。
```cpp
class ClonableBase {
public:
    virtual unique_ptr<ClonableBase> clone() const;
    virtual ~ClonableBase() = default;
    CloneableBase() = default;
    ClonableBase(const ClonableBase&) = delete;
    ClonableBase& operator=(const ClonableBase&) = delete;
    ClonableBase(ClonableBase&&) = delete;
    ClonableBase& operator=(ClonableBase&&) = delete;
    // ... other constructors and functions ...
};
```
在这里只定义移动操作或只定义拷贝操作会其同样效果，但显式写出每个特殊函数会让读者更清楚。

**注意** 在一个有析构函数的类中依赖隐式生成的拷贝操作是被废弃的。

**注意**
```cpp
class X {
public:
    // ...
    virtual ~X() = default;            // destructor (virtual if X is meant to be a base class)
    X(const X&) = default;             // copy constructor
    X& operator=(const X&) = default;  // copy assignment
    X(X&&) = default;                  // move constructor
    X& operator=(X&&) = default;       // move assignment
};
```
编写这些函数容易出错，注意它们的参数类型，一个小错误，比如拼写错误、遗漏const、使用&而不是&&，或者遗漏一个特殊函数，都可能导致错误或警告。
为了避免繁琐的工作和出错可能性，尽量遵循the rule of zero。

<br/>
### **C.22:让所有默认操作符保持一致**

**原因** 默认操作在概念是一个匹配集合，它们的语义是相互关联的。如果拷贝/移动函数和拷贝/移动赋值操作符在逻辑上做不同的事情，用户就会感到惊讶。
如果构造函数和析构函数没有一致的资源管理view，用户也会感到惊讶。如果拷贝和移动不能反应构造和析构函数的工作方式，用户也会感到惊讶。

**Example,bad**
```cpp
class Silly {   // BAD: Inconsistent copy operations
    class Impl {
        // ...
    };
    shared_ptr<Impl> p;
public:
    Silly(const Silly& a) : p(make_shared<Impl>()) { *p = *a.p; }   // deep copy
    Silly& operator=(const Silly& a) { p = a.p; }   // shallow copy
    // ...
};
```
这些操作在拷贝语义上不一致，造成混乱和错误。

<br/>
## **C.dtor:析构**

需要给类声明一个析构函数么？这是一个令人惊讶而有洞察力的设计问题。对于大多数人来说答案是“不”，因为这个类不拥有资源，或者析构函数由the rule of zero处理。
也就是，它的成员可以自行处理析构问题。如果答案是“是”，那么这个类的大部分设计都要处理，the rule of five。

<br/>
### **C.30:如果对象需要显式操作，那么就要定义一个析构函数**

**原因** 一个析构函数在对象生命期结束时被隐式调用。如果默认的析构函数可满足，就使用默认的。只有一个类需要执行不属于其成员的析构的代码时，才定义一个非默认的析构函数。

**Example**
```cpp
template<typename A>
struct final_action {   // slightly simplified
    A act;
    final_action(A a) : act{a} {}
    ~final_action() { act(); }
};

template<typename A>
final_action<A> finally(A act)   // deduce action type
{
    return final_action<A>{act};
}

void test()
{
    auto act = finally([] { cout << "Exit test\n"; });  // establish exit action
    // ...
    if (something) return;   // act done here
    // ...
} // act done here
```
final\_action类的目的就是在析构的时候执行一段代码，通常是lambda。

**注意** 一般来说有两种情况，类需要用户自定义析构函数：
- 一个资源类，它还没有被代表为一个具有析构函数的类，例如一个vector类或交易类。
- 一个主要为了析构时要执行一些操作，比如tracer或final\_action。

**Example,bad**
```cpp
class Foo {   // bad; use the default destructor
public:
    // ...
    ~Foo() { s = ""; i = 0; vi.clear(); }  // clean up
private:
    string s;
    int i;
    vector<int> vi;
};
```
编译器默认生成的析构就可以完成，而且它更高效，更不容易出错。

**注意** 如果需要默认的析构函数，但是它的生成被抑制了（比如类定义了一个移动构造函数），那么可以用=default显式声明。

<br/>
### **C.31:一个类如果构造函数获得资源，那么析构函数应该释放资源**

**原因** 避免资源泄漏，特别是在错误情况下。

**注意** 对于资源类且有完整的默认操作集，这些操作将自动发生。

**Example**
```cpp
class X {
    ifstream f;   // might own a file
    // ... no default operations defined or =deleted ...
};
```
X在析构的时候，隐式关闭了任何它所打开的文件。

**Example,bad**
```cpp
class X2 {     // bad
    FILE* f;   // might own a file
    // ... no default operations defined or =deleted ...
};
```
X2可能会泄漏。

**注意** 如果类中有成员是指针或引用，但它们不是所有者，显然类的析构函数不能delete所指对象。
```cpp
Preprocessor pp { /* ... */ };
Parser p { pp, /* ... */ };
Type_checker tc { p, /* ... */ };
```
这p只是指向pp，但不拥有所有权。

**Enforcement** 
- 如果一个类有成员是指针或引用，且它们拥有所有权，那它们应该在析构函数中被引用。
- 当没有显式的所有权声明时，应该确定下指针或引用成员变量是否是所有者。

<br/>
### **C.32:如果一个类有T\*或者T&，那么考虑它是否拥有所有权**

**原因** 有很多代码对所有权不是具体的。

**Example**
```cpp
class legacy_class
{
    foo* m_owning;   // Bad: change to unique_ptr<T> or owner<T*>
    bar* m_observer; // OK: keep
}
```

**注意** 拥有指针的新代码和重构的旧代码应该明确所有权[R.3](#r3t不拥有所有权)，[R.20](#r20用unique_ptr和shared_ptr来表达所有权)。
引用不拥有所有权[R.4](#r4t不拥有所有权)

<br/>
### **C.33:如果一个类有一个拥有所有权的指针，那么要定义一个析构函数**

**原因** 一个包含拥有所有权的指针对象必须在析构时删除所指对象。

**Example** 一个指针成员可以代表一个资源，一个T\*不这样，但在老代码里，这很常见。考虑把T\*当成可能的所有者。
```cpp
template<typename T>
class Smart_ptr {
    T* p;   // BAD: vague about ownership of *p
    // ...
public:
    // ... no user-defined default operations ...
};

void use(Smart_ptr<int> p1)
{
    // error: p2.p leaked (if not nullptr and not owned by some other code)
    auto p2 = p1;
}
```
注意如果你定义了一个析构函数，你必须定义或delete所有默认函数。
```cpp
template<typename T>
class Smart_ptr2 {
    T* p;   // BAD: vague about ownership of *p
    // ...
public:
    // ... no user-defined copy operations ...
    ~Smart_ptr2() { delete p; }  // p is an owner!
};

void use(Smart_ptr2<int> p1)
{
    auto p2 = p1;   // error: double deletion
}
```
默认拷贝操作只是拷贝了p1.p到p2.p，那可能导致p1.p两次析构。因此要显式指出所有权。

```cpp
template<typename T>
class Smart_ptr3 {
    owner<T*> p;   // OK: explicit about ownership of *p
    // ...
public:
    // ...
    // ... copy and move operations ...
    ~Smart_ptr3() { delete p; }
};

void use(Smart_ptr3<int> p1)
{
    auto p2 = p1;   // OK: no double deletion
}
```

**注意** 通常最简单的做法是用一个智能指针来替换裸指针，让编译器隐式地进行正确的析构。

**Enforcement**
- 一个类如果有成员是指针，那么这是可疑的。
- 一个类如果有owner\<T\>，那么应该定义一组默认操作函数。

<br/>
### **C.35:基类的析构函数应该是public和virtual的，或者是protected和non-virtual的**

**原因** 避免未定义。
如果析构函数是public的，那么调用者可以通过基类指针来析构继承类对象，如果基类析构函数不是virtual的，那么这个行为是未定义的。
如果析构函数是protected的，调用者无法通过基类指针进行析构，并且析构函数不需要是virtual的。
如果析构函数是protected的，而不是private的，那么派生类的析构函数可以调用它。
一般来说，基类代码编写者不知道析构函数要做什么合适的动作。

**Example,bad**
```cpp
struct Base {  // BAD: implicitly has a public non-virtual destructor
    virtual void f();
};

struct D : Base {
    string s {"a resource needing cleanup"};
    ~D() { /* ... do some cleanup ... */ }
    // ...
};

void use()
{
    unique_ptr<Base> p = make_unique<D>();
    // ...
} // p's destruction calls ~Base(), not ~D(), which leaks D::s and possibly more
```

**注意** 虚函数定义了一个派生类的接口，可以在不看派生类情况下调用。如果该接口允许析构，那这样做应该是安全的。

**注意** 虚构函数必须是非私有的，不然将阻止使用该类型。
```cpp
class X {
    ~X();   // private destructor
    // ...
};

void use()
{
    X a;                        // error: cannot destroy
    auto p = make_unique<X>();  // error: cannot destroy
}
```

**Enforcement**
- 如果一个类有虚函数，那么它的析构函数必须是public且virtual的，要么就是protected的且non-virtual的。
- 如果一个类public继承一个基类，那么基类应该有一个析构函数，它要么是public且virtual的，要么就是protected的且non-virtual的。

<br/>
### **C.36:析构函数不能失败**

**原因** 一般来说，如果析构函数失败，我们不知道如果编写无错误代码。
标准库要求所有与之打交道的类，析构函数都不以抛出异常退出。

**Example**
```cpp
class X {
public:
    ~X() noexcept;
    // ...
};

X::~X() noexcept
{
    // ...
    if (cannot_release_a_resource) terminate();
    // ...
}
```

**注意** 析构函数声明为noexcept，这将确保析构函数要么正常完成，要么立刻终止程序。

<br/>
### **C.37:析构函数应该noexcept**

**原因** 如果析构函数抛出异常，这是个糟糕的设计，程序最好立刻终止。

**注意** 析构函数无论是用户定义的还是编译器自动生成的，如果类的所有成员都有noexcept的析构函数，那么它就会隐式地声明为noexcept的，这与析构函数的主体代码无关。
通过显式地声明为noexcept，作者可以防止类通过添加或修改类成员而成为隐式的noexcept(false)。

**Example** 并非所有的析构函数都是noexcept的，一个会抛出异常的成员就会导致非noexcept。
```cpp
struct X {
    Details x;  // happens to have a throwing destructor
    // ...
    ~X() { }    // implicitly noexcept(false); aka can throw
};
```

**Enforcement** 如果析构函数可以throw，那么应该声明为noexcept。

<br/>
## **C.ctor:构造函数**

<br/>
### **C.40:如果类有一个不变量，那么定义一个构造函数**

**原因** 这就是构造函数的作用。

**Example**
```cpp
class Date {  // a Date represents a valid date
              // in the January 1, 1900 to December 31, 2100 range
    Date(int dd, int mm, int yy)
        :d{dd}, m{mm}, y{yy}
    {
        if (!is_valid(d, m, y)) throw Bad_date{};  // enforce invariant
    }
    // ...
private:
    int d, m, y;
};
```
一般来说在构造函数中Ensures一个不变量。

**注意** 即使没有不变量，也可以用构造函数来提供方便。
```cpp
struct Rec {
    string s;
    int i {0};
    Rec(const string& ss) : s{ss} {}
    Rec(int ii) :i{ii} {}
};

Rec r1 {7};
Rec r2 {"Foo bar"};
```

**注意** C++11初始化列表规则可以消除许多构造函数。
```cpp
struct Rec2{
    string s;
    int i;
    Rec2(const string& ss, int ii = 0) :s{ss}, i{ii} {}   // redundant
};

Rec2 r1 {"Foo", 7};
Rec2 r2 {"Bar"};
```
Rec2构造函数是多余的。

**Enforcement** 
- 如果有用户自定义的拷贝操作但是没有定义构造函数，那么应该标记出来，因为用户定义拷贝操作通常意味着这个类又一个不变量。

<br/>
### **C.41:一个构造函数应该创建一个完全初始化的对象**

**原因** 构造函数为类建立不变性，类的用户应该假定构造过的对象是可用的。

**Example,bad**
```cpp
class X1 {
    FILE* f;   // call init() before any other function
    // ...
public:
    X1() {}
    void init();   // initialize f
    void read();   // read from f
    // ...
};

void f()
{
    X1 file;
    file.read();   // crash or bad read!
    // ...
    file.init();   // too late
    // ...
}
```

**例外** 有些对象不能方便地由构造函数构造出，比如factory函数。

**Enforcement**
- 所有构造函数应该初始化所有成员变量（显式地、代理构造或默认构造）。
- 如果一个构造函数有Ensures，尝试看下后置条件是否成立。

**注意** 如果构造函数获得资源，那资源应该由析构函数来释放，这种被称为RAII。

<br/>
### **C.42:如果一个构造函数不能构造一个有效的对象，则抛出一个异常**

**原因** 留下一个无效对象就是自寻麻烦。

**Example**
```cpp
class X2 {
    FILE* f;
    // ...
public:
    X2(const string& name)
        :f{fopen(name.c_str(), "r")}
    {
        if (!f) throw runtime_error{"could not open" + name};
        // ...
    }

    void read();      // read from f
    // ...
};

void f()
{
    X2 file {"Zeno"}; // throws if file isn't open
    file.read();      // fine
    // ...
}
```

**Example,bad**
```cpp
class X3 {     // bad: the constructor leaves a non-valid object behind
    FILE* f;   // call is_valid() before any other function
    bool valid;
    // ...
public:
    X3(const string& name)
        :f{fopen(name.c_str(), "r")}, valid{false}
    {
        if (f) valid = true;
        // ...
    }

    bool is_valid() { return valid; }
    void read();   // read from f
    // ...
};

void f()
{
    X3 file {"Heraclides"};
    file.read();   // crash or bad read!
    // ...
    if (file.is_valid()) {
        file.read();
        // ...
    }
    else {
        // ... handle error ...
    }
    // ...
}
```

**注意** 对于一个变量定义（堆栈上的或作为另外一个对象的成员），没有显式的函数调用可以返回一个错误代码。留下一个无效的对象，
并依靠用户在使用前不断检查is\_valid函数是繁琐的，容易出错且效率低下。

**例外** 有一些领域，比如一些硬件实时系统（比如飞行控制），如果没有额外工具支持的话，从时间角度看，异常处理是不能充分预测的。
那么就需要使用is\_valid技术，在这个情况下，持续并立即检查is\_valid以模拟RAII。

**替代方案** 如果你很想使用“构造后初始化”或者“两段初始化”的idiom，尽量不要，真要这样做，可以看下factory函数。

**注意** 只有一个情况人们使用init()函数而不是在构造函数中初始化，是为了避免代码重复。代理构造函数和默认成员初始化可以做的更好。
另一个原因推迟初始化，是为了需要一个对象时才进行初始化，解决这个问题的办法是，在正确初始化之前不要声明一个变量。


**Enforcement**
- 如果一个类可拷贝，且有=操作符，但是没有默认构造函数应该标记出。
- 如果一个类有比较函数==，但没有默认构造函数应该标记出。

<br/>
### **C.43:确保一个可复制类有一个默认构造函数**

**原因** 许多语言和库依赖构造函数来初始化它们的元素，比如T a\[10\]和std::vector\<T\> v(10)。默认构造函数通常简化了可拷贝类的合适的move-from state。

**Example**
```cpp
class Date { // BAD: no default constructor
public:
    Date(int dd, int mm, int yyyy);
    // ...
};

vector<Date> vd1(1000);   // default Date needed here
vector<Date> vd2(1000, Date{Month::October, 7, 1885});   // alternative
```
默认构造函数只有在没有用户定义构造函数的情况下才自动生成，因此上面的例子中无法初始化vector d1。没有默认值通常会给用户带来惊讶，并使其使用复杂化。
如果可以合理定义一个默认值，就应该这样做。

选择用Date是来鼓励思考的。没有一个“自然”的默认date，大爆炸的时间太过遥远，对于大多数人都是无用的，因此这个例子是非琐碎的。
\{0, 0, 0\}不是一个有效日期，所以选择它有点像浮点数的NaN。然而大多数实际的Date都有一个“起始日期”（1970年1月1日），通常把它当作默认值是容易的。


```cpp
class Date {
public:
    Date(int dd, int mm, int yyyy);
    Date() = default; // [See also](#Rc-default)
    // ...
private:
    int dd {1};
    int mm {1};
    int yyyy {1970};
    // ...
};

vector<Date> vd1(1000);
```
```cpp
struct X {
    string s;
    vector<int> v;
};

X x; // means X{ {}, {} }; that is the empty string and the empty vector
```

**注意** 一个类的所有成员都有默认构造函数的话，隐式的表示会有一个默认的构造函数。

注意内置类型没有合适的默认构造函数。
```cpp
struct X {
    string s;
    int i;
};

void f()
{
    X x;    // x.s is initialized to the empty string; x.i is uninitialized

    cout << x.s << ' ' << x.i << '\n';
    ++x.i;
}
```
内置类型的静态分配对象默认初始化为0，但局部内置变量则不会。注意有的编译器会默认初始化局部内置变量，但一些优化后的编译器不会。
因此上面的例子看起来是有效的，但是未定义的。如果真想要初始化，一个显式的默认构造函数可以解决。
```cpp
struct X {
    string s;
    int i {};   // default initialize (to 0)
};
```

**注意** 没有默认构造函数的类通常也是不可复制的，这种情况下，本条规则不适用。
举个例子，基类不可复制的，那就不需要一个默认构造函数。
```cpp
// Shape is an abstract base class, not a copyable type.
// It might or might not need a default constructor.
struct Shape {
    virtual void draw() = 0;
    virtual void rotate(int) = 0;
    // =delete copy/move functions
    // ...
};
```

一个类在构造函数中必须获得调用者提供的资源，通常也不能有一个默认构造函数，但它也不属于本条规则，因为通常这样的类是不可复制的。
```cpp
// std::lock_guard is not a copyable type.
// It does not have a default constructor.
lock_guard g {mx};  // guard the mutex mx
lock_guard g2;      // error: guarding nothing
```

一个拥有“特殊状态”的类，必须由成员函数或用户将其与其他状态分开处理，这会增加额外的工作，而且很可能是更多错误。这样的类型自然
可以用特殊状态来作为默认的构造值，不管它是否可复制。
```cpp
// std::ofstream is not a copyable type.
// It does happen to have a default constructor
// that goes along with a special "not open" state.
ofstream out {"Foobar"};
// ...
out << log(time, transaction);
```

类似的可复制的特殊状态类型，例如具有特殊状态“==nullptr”的可复制智能指针，应该用特殊状态来作为它的默认构造值。

然而，最好还是默认构造函数使对象成为一个默认有意义的一个状态，比如std::string的""和std::vector的{}。

**Enforcement**
- 如果一个类可拷贝，且有=操作符，但是没有默认构造函数应该标记出。
- 如果一个类有比较函数==，但没有默认构造函数应该标记出。

<br/>
### **C.44:倾向于默认构造函数是简单的且不会抛异常**

**原因** 不会失败的默认构造函数，可以简化错误处理和移动操作。

**Example, problematic**
```cpp
template<typename T>
// elem points to space-elem element allocated using new
class Vector0 {
public:
    Vector0() :Vector0{0} {}
    Vector0(int n) :elem{new T[n]}, space{elem + n}, last{elem} {}
    // ...
private:
    own<T*> elem;
    T* space;
    T* last;
};
```
这很好很普遍，但是在出错后将Vector0设置为空涉及到一个分配，这可能会失败。另外让一个默认的Vector0表示为{new T[0], 0, 0}似乎很浪费。
例如，Vector0\<int\> v[100]需要100次分配。

**Example**
```cpp
template<typename T>
// elem is nullptr or elem points to space-elem element allocated using new
class Vector1 {
public:
    // sets the representation to {nullptr, nullptr, nullptr}; doesn't throw
    Vector1() noexcept {}
    Vector1(int n) :elem{new T[n]}, space{elem + n}, last{elem} {}
    // ...
private:
    own<T*> elem {};
    T* space {};
    T* last {};
};
```
{nullptr, nullptr, nullptr}让Vector1{}很便宜，但却是一个特例，且意味着运行时检查。在检测到错误后将一个Vector1设置为空是很简单的。

**Enforcement**
- 默认构造函数会抛异常的都应该标记出。

<br/>
### **C.45:不要定义一个只初始化数据成员的默认构造函数；使用初始化成员函数**

**原因** 使用类内成员初始化可以让编译器为你生成构造函数，编译器生成的函数可以更有效。

**Example, bad**
```cpp
class X1 { // BAD: doesn't use member initializers
    string s;
    int i;
public:
    X1() :s{"default"}, i{1} { }
    // ...
};
```

**Example**
```cpp
class X2 {
    string s {"default"};
    int i {1};
public:
    // use compiler-generated default constructor
    // ...
};
```

**Enforcement**
- 默认构造函数应该做更多的事情，而不仅仅是初始化成员变量。

<br/>
### **C.46:默认情况下，单参数构造函数应该explicit声明**

**原因** 避免无意识的转换。

**Example, bad**
```cpp
class String {
public:
    String(int);   // BAD
    // ...
};

String s = 10;   // surprise: string of size 10
```

**例外** 如果你想从一个函数参数类型隐式地转换为类类型，不要使用explicit。
```cpp
class Complex {
public:
    Complex(double d);   // OK: we want a conversion from d to {d, 0}
    // ...
};

Complex z = 10.7;   // unsurprising conversion
```

**注意** 拷贝和移动构造函数不应该explicit，因为它们不执行转换。显式地拷贝和移动构造函数让值传递和返回变得困难。

**Enforcement** 单参数构造函数应该explicit声明。在大多数代码中，但参数而非explicit是罕见的。

<br/>
### **C.47:按照成员声明的顺序定义和初始化成员变量**

**原因** 最小化混淆和错误。

**Example, bad**
```cpp
class Foo {
    int m1;
    int m2;
public:
    Foo(int x) :m2{x}, m1{++x} { }   // BAD: misleading initializer order
    // ...
};

Foo x(1); // surprise: x.m1 == x.m2 == 2
```

**Enforcement** 成员的初始化列表应该按照它们被声明的顺序。

<br/>
### **C.48:对于常量初始化构造函数，优先选择类内初始化而不是成员初始化**

**原因** 显式指出所有构造函数中都使用相同值，避免重复，避免维护问题。这个写法是最短的且高效的代码。

**Example, bad**
```cpp
class X {   // BAD
    int i;
    string s;
    int j;
public:
    X() :i{666}, s{"qqq"} { }   // j is uninitialized
    X(int ii) :i{ii} {}         // s is "" and j is uninitialized
    // ...
};
```
维护者如何知道j是否故意不初始化（反正大概率是坏主意），是否故意在一种情况下给s默认值""，在另外一种情况下赋值qqq（几乎肯定是个错误）
j的问题（未初始化）经常发生在一个新的成员添加到已有的类中。

**Example**
```cpp
class X2 {
    int i {666};
    string s {"qqq"};
    int j {0};
public:
    X2() = default;        // all members are initialized to their defaults
    X2(int ii) :i{ii} {}   // s and j initialized to their defaults
    // ...
};
```

**替代方案** 我们可以从构造函数的默认参数获得部分好处，这在老代码中并不少见。然而这样做不大明确，且导致更多的参数被传递，而且在有多个构造函数情况下是重复的。
```cpp
class X3 {   // BAD: inexplicit, argument passing overhead
    int i;
    string s;
    int j;
public:
    X3(int ii = 666, const string& ss = "qqq", int jj = 0)
        :i{ii}, s{ss}, j{jj} { }   // all members are initialized to their defaults
    // ...
};
```

**Enforcement**
- 每个构造函数应该初始化每个成员变量，要么显式，要么通过代理构造，或者通过默认构造。
- 构造函数的默认参数表明类内的初始化可能更合适。

<br/>
### **C.49:构造函数中优先考虑初始化而不是赋值**

**原因** 初始化应该明确进行初始化动作而不是赋值，且更优雅和高效。可以防止先使用后设置的错误。

**Example, good**
```cpp
class A {   // Good
    string s1;
public:
    A(czstring p) : s1{p} { }    // GOOD: directly construct (and the C-string is explicitly named)
    // ...
};
```

**Example,bad**
```cpp
class B {   // BAD
    string s1;
public:
    B(const char* p) { s1 = p; }   // BAD: default constructor followed by assignment
    // ...
};

class C {   // UGLY, aka very bad
    int* p;
public:
    C() { cout << *p; p = new int{10}; }   // accidental use before initialized
    // ...
};
```

**Example, better still** C++17之后应该使用std::string\_view来替换const char\*，或者用gsl::span\<char\>。
```cpp
class D {   // Good
    string s1;
public:
    D(string_view v) : s1{v} { }    // GOOD: directly construct
    // ...
};
```

<br/>
### **C.50:如果你在初始化过程中需要virtual行为，请使用factory函数**

**原因** 如果一个基类对象的状态依赖派生类的部分状态，我们需要一个虚函数，同时应该尽量减少滥用一个不完善对象的机会窗口。

**注意** factory函数的返回类型通常默认为unique\_ptr。如果某些使用是共享的，调用者可以将unique\_ptr转到shared\_ptr中，然而，
如果factor函数作者知道返回的对象用于共享，那么直接返回shared\_ptr，并在代码中使用make\_shared来节省分配操作。

**Example, bad**
```cpp
class B {
public:
    B()
    {
        /* ... */
        f(); // BAD: C.82: Don't call virtual functions in constructors and destructors
        /* ... */
    }

    virtual void f() = 0;
};
```

**Example**
```cpp
class B {
protected:
    class Token {};

public:
    explicit B(Token) { /* ... */ }  // create an imperfectly initialized object
    virtual void f() = 0;

    template<class T>
    static shared_ptr<T> create()    // interface for creating shared objects
    {
        auto p = make_shared<T>(typename T::Token{});
        p->post_initialize();
        return p;
    }

protected:
    virtual void post_initialize()   // called right after construction
        { /* ... */ f(); /* ... */ } // GOOD: virtual dispatch is safe
};

class D : public B {                 // some derived class
protected:
    class Token {};

public:
    explicit D(Token) : B{ B::Token{} } {}
    void f() override { /* ...  */ };

protected:
    template<class T>
    friend shared_ptr<T> B::create();
};

shared_ptr<D> p = D::create<D>();  // creating a D object
```
make\_shared要求构造函数是public的，通过要求一个protected的Token，那么构造函数就不能被publicly调用，这样就可以避免不完整的构造。
只有通过factory函数create，我们可以让构造变得方便（从堆分配）。

**注意** 传统的factory函数会在堆上分配，而不是栈上或封闭的对象中分配。

<br/>
### **C.51:使用代理函数来表示一个类所有构造函数所共有部分**

**原因** 避免重复和意外的差异。

**Example, bad**
```cpp
class Date {   // BAD: repetitive
    int d;
    Month m;
    int y;
public:
    Date(int dd, Month mm, year yy)
        :d{dd}, m{mm}, y{yy}
        { if (!valid(d, m, y)) throw Bad_date{}; }

    Date(int dd, Month mm)
        :d{dd}, m{mm} y{current_year()}
        { if (!valid(d, m, y)) throw Bad_date{}; }
    // ...
};
```
**Example**
```cpp
class Date2 {
    int d;
    Month m;
    int y;
public:
    Date2(int dd, Month mm, year yy)
        :d{dd}, m{mm}, y{yy}
        { if (!valid(d, m, y)) throw Bad_date{}; }

    Date2(int dd, Month mm)
        :Date2{dd, mm, current_year()} {}
    // ...
};
```

<br/>
### **C.52:使用继承构造函数将构造函数导入到派生类中，而不需要进一步显式初始化**

**原因** 如果你在派生类中需要这些构造函数，重新实现它们是乏味且易错的。

**Example** std::vector有很多棘手的构造函数，如果我想要实现自己的vector，我不想重新各个实现它们。
```cpp
class Rec {
    // ... data and lots of nice constructors ...
};

class Oper : public Rec {
    using Rec::Rec;
    // ... no data members ...
    // ... lots of nice utility functions ...
};
```

**Example, bad**
```cpp
struct Rec2 : public Rec {
    int x;
    using Rec::Rec;
};

Rec2 r {"foo", 7};
int val = r.x;   // uninitialized
```

**Enforcement** 确保派生类的每个成员都被初始化。

<br/>
## **C.copy:拷贝和移动**
实体类型通常是可拷贝的，但类层次结构中的接口不应该是可复制的。资源的句柄可能不可复制的，也可能可复制的，出于逻辑和性能原因，可以定义类型来move。

<br/>
### **C.60:拷贝赋值函数应该为non-virtual，用const&传参数，且返回non-const&**

**原因** 简单而高效。如果你想对rvalue进行优化，请提供一个带&&参数的重载函数。

**Example**
```cpp
class Foo {
public:
    Foo& operator=(const Foo& x)
    {
        // GOOD: no need to check for self-assignment (other than performance)
        auto tmp = x;
        swap(tmp); // see C.83
        return *this;
    }
    // ...
};

Foo a;
Foo b;
Foo f();

a = b;    // assign lvalue: copy
a = f();  // assign rvalue: potentially move
```

**Example** 但是，如果不做临时拷贝能获得明显的性能提升么？考虑一个简单的vector，它专用于一个领域，这个领域大的以及同等大小的vector分配是很常见的。
在这种情况下，swap所隐式的技术实现的拷贝可能会让成本增加一个数量级。
```cpp
template<typename T>
class Vector {
public:
    Vector& operator=(const Vector&);
    // ...
private:
    T* elem;
    int sz;
};

Vector& Vector::operator=(const Vector& a)
{
    if (a.sz > sz) {
        // ... use the swap technique, it can't be bettered ...
        return *this;
    }
    // ... copy sz elements from *a.elem to elem ...
    if (a.sz < sz) {
        // ... destroy the surplus elements in *this and adjust size ...
    }
    return *this;
}
```
通过直接写入目标元素，我们将只得到基本的保证，而不是swap所提供的强保证。谨防自我赋值。

**替代方案** 如果你认为你需要一个virtual的赋值运算符，且理解这样做有很大的问题，那就不要operator=，把它命名为virtual void assign(const Foo&)。

**Enforcement**
- 赋值运算符不应该是virtual的。
- 赋值运算符应该返回T&以实现链式操作，而不是像const T&，影响了可组合性以及可将对象放入容器内。
- 赋值运算符应该（隐式或显式）调用所有基类和成员赋值运算符，看下析构函数可以确定类型是否指针语义或值语义。

<br/>
### **C.61:一个拷贝操作就应该有拷贝**

**原因** 这就是一般假设的语义。x=y之后，x应该==y，拷贝之后x和y应该都是独立的对象（值语义，非指针内置类型和标准库类型的工作方式），也可以指向一个共享对象
（指针语义，指针的工作方式）。

**Example**
```cpp
class X {   // OK: value semantics
public:
    X();
    X(const X&);     // copy X
    void modify();   // change the value of X
    // ...
    ~X() { delete[] p; }
private:
    T* p;
    int sz;
};

bool operator==(const X& a, const X& b)
{
    return a.sz == b.sz && equal(a.p, a.p + a.sz, b.p, b.p + b.sz);
}

X::X(const X& a)
    :p{new T[a.sz]}, sz{a.sz}
{
    copy(a.p, a.p + sz, p);
}

X x;
X y = x;
if (x != y) throw Bad{};
x.modify();
if (x == y) throw Bad{};   // assume value semantics
```

**Example**
```cpp
class X2 {  // OK: pointer semantics
public:
    X2();
    X2(const X2&) = default; // shallow copy
    ~X2() = default;
    void modify();          // change the pointed-to value
    // ...
private:
    T* p;
    int sz;
};

bool operator==(const X2& a, const X2& b)
{
    return a.sz == b.sz && a.p == b.p;
}

X2 x;
X2 y = x;
if (x != y) throw Bad{};
x.modify();
if (x != y) throw Bad{};  // assume pointer semantics
```

**注意** 除非你正在实现类似“智能指针”，优先使用值语义，值语义是最简单的，也是标准库设施所期望的。

<br/>
### **C.62:拷贝赋值操作时应该保证自我赋值安全**

**原因** 如果x=x改变了x的值，人们会感到惊讶，也会出现不好的错误比如泄漏。

**Example** 标准库容器能够优雅和高效地处理self-assignment。
```cpp
std::vector<int> v = {3, 1, 4, 1, 5, 9};
v = v;
// the value of v is still {3, 1, 4, 1, 5, 9}
```

**注意** 如果所有成员能够处理自我赋值问题，那么类也可以处理。
```cpp
struct Bar {
    vector<pair<int, int>> v;
    map<string, int> m;
    string s;
};

Bar b;
// ...
b = b;   // correct and efficient
```

**注意** 你可以显式地测试自我赋值来处理自我赋值，但通常情况下，没有这样的测试也能更快、更优雅地应对（比如使用swap）。
```cpp
class Foo {
    string s;
    int i;
public:
    Foo& operator=(const Foo& a);
    // ...
};

Foo& Foo::operator=(const Foo& a)   // OK, but there is a cost
{
    if (this == &a) return *this;
    s = a.s;
    i = a.i;
    return *this;
}
```
这显然是安全和高效的。然而，如果我们每一百万次赋值才做一次自我赋值呢，那就大约是一百万次的冗余测试（但由于答案基本上总是相同的，
所以计算机的分支预测基本上每次都会猜对）。
```cpp
Foo& Foo::operator=(const Foo& a)   // simpler, and probably much better
{
    s = a.s;
    i = a.i;
    return *this;
}
```
std::string和int自我赋值是安全的，所有的开销都由自我赋值承担（很少见）。

**Enforcement** 赋值操作中不应该包括if (this == &a) return \*this; ???

<br/>
### **C.63:移动赋值函数应该为non-virtual，用&&传参，且返回non-const&**

**原因** 简单而高效。
跟拷贝赋值规则一致。

**Enforcement** 跟拷贝赋值规则一致。

<br/>
### **C.64:一个移动操作应该发生移动并使其来源处于有效状态**

**原因** 这就是一般假设的语义，在y = std::move(x)之后，y的值应该是x的值，而且x应该处于有效状态。

**Example**
```cpp
class X {   // OK: value semantics
public:
    X();
    X(X&& a) noexcept;  // move X
    X& operator=(X&& a) noexcept; // move-assign X
    void modify();     // change the value of X
    // ...
    ~X() { delete[] p; }
private:
    T* p;
    int sz;
};

X::X(X&& a) noexcept
    :p{a.p}, sz{a.sz}  // steal representation
{
    a.p = nullptr;     // set to "empty"
    a.sz = 0;
}

void use()
{
    X x{};
    // ...
    X y = std::move(x);
    x = X{};   // OK
} // OK: x can be destroyed
```

**注意** 理想情况下，moved-from对象应该有类型默认值，除非有特别好的理由我们应该确保这样。然而不是所有的类型都有一个默认值，
对于某些类型来说，设置默认值可能很昂贵。标准只要求被移出的对象可以被销毁，通常情况下，我们可以容易且cheap地实现。
标准库假定对被移出的对象进行赋值，总是让被移出的对象出于某种有效状态。

**注意** 除非有特别强的理由，否则让x = std::move(y); y = z; 更传统语义一致。

**Enforcement** 在move操作中，查看对成员的赋值，如果有一个默认的构造函数，将这些赋值与默认构造函数中的初始化进行比较。

<br/>
### **C.65:移动赋值操作时应该保证自我赋值安全**

**原因** 通常不会直接写一个变成移动的自我赋值，但它也可能发生，然而，std::swap使用移动操作实现的，如果a和b指向同个对象，
然后不小心swap(a, b)，未能处理自我移动可能是个严重且微妙的错误。

**Example**
```cpp
class Foo {
    string s;
    int i;
public:
    Foo& operator=(Foo&& a);
    // ...
};

Foo& Foo::operator=(Foo&& a) noexcept  // OK, but there is a cost
{
    if (this == &a) return *this;  // this line is redundant
    s = std::move(a.s);
    i = a.i;
    return *this;
}
```

**注意** 目前还没有已知的一般方法来避免if (this == &a) return \*this;来识别移动赋值且获得一个正确的答案（即x = x后值也没有变化）。

**注意** 标准只保证了标准库容器的“有效但未指定”的状态。显然在过去10年中的实践和生产环境使用中，这并未成为一个问题。如果你找到一个反面例子，
请联系编辑。这里的规则是更谨慎，更坚持完全的安全性。

**Example** 这里有一个不用测试就能移动指针的方法。
```cpp
// move from other.ptr to this->ptr
T* temp = other.ptr;
other.ptr = nullptr;
delete ptr;
ptr = temp;
```

**Enforcement**
- 自我赋值情况下，移动赋值操作符不应该让对象持有已被删除或设置为nullptr的指针。
- 看看标准库容器类型（包括字符串）的使用，认定它们对普通使用是安全的。

<br/>
### **C.66:移动操作应该设为noexcept**

**原因** 抛出异常违法大多数人的合理假定，标准库和语言设施会更有效地使用不抛异常的move操作。

**Example**
```cpp
template<typename T>
class Vector {
public:
    Vector(Vector&& a) noexcept :elem{a.elem}, sz{a.sz} { a.sz = 0; a.elem = nullptr; }
    Vector& operator=(Vector&& a) noexcept { elem = a.elem; sz = a.sz; a.sz = 0; a.elem = nullptr; }
    // ...
private:
    T* elem;
    int sz;
};
```
这些操作不会抛出异常。

**Example,bad**
```cpp
template<typename T>
class Vector2 {
public:
    Vector2(Vector2&& a) { *this = a; }             // just use the copy
    Vector2& operator=(Vector2&& a) { *this = a; }  // just use the copy
    // ...
private:
    T* elem;
    int sz;
};
```
Vector2的问题不仅是不够高效，拷贝过程中涉及到分配，它可能抛出异常。

**Enforcement** move操作应该标记为noexcept。

<br/>
### **C.67:多态类应该抑制public的拷贝或移动**

**原因** 多态类是一个定义或继承了至少一个虚函数的类，它很可能被用于其他具有多态行为派生类的基类，如果它被意外地以值传递，通过隐式
生成的拷贝构造和拷贝赋值函数，我们就有被切片的风险，因为只有基类的部分会被复制，那么多态的行为就会被破坏。
如果一个类没有数据，=delete掉拷贝/移动函数，否则把它们设为protected。

**Example,bad**
```cpp
class B { // BAD: polymorphic base class doesn't suppress copying
public:
    virtual char m() { return 'B'; }
    // ... nothing about copy operations, so uses default ...
};

class D : public B {
public:
    char m() override { return 'D'; }
    // ...
};

void f(B& b)
{
    auto b2 = b; // oops, slices the object; b2.m() will return 'B'
}

D d;
f(d);
```

**Example**
```cpp
class B { // GOOD: polymorphic class suppresses copying
public:
    B() = default;
    B(const B&) = delete;
    B& operator=(const B&) = delete;
    virtual char m() { return 'B'; }
    // ...
};

class D : public B {
public:
    char m() override { return 'D'; }
    // ...
};

void f(B& b)
{
    auto b2 = b; // ok, compiler will detect inadvertent copying, and protest
}

D d;
f(d);
```

**注意** 多态对象如果需要深拷贝的话，使用clone()。

**例外** 表示异常对象的类既要有多态性，又要有可复制的结构。

**Enforcement**
- 多态类的public拷贝操作应该标记出。
- 多态类对象的赋值也应该标记出。

<br/>
## **C.other:其他默认操作规则**
除了语言提供默认实现的操作外，还有一些操作是非常基础的，需要为其定义指定具体的规则：比较、swap和hash。

<br/>
### **C.80:如果你显式使用默认语义的话使用=default**

**原因** 编译器更有可能得到正确的默认语义，你不可能比编译器更好地实现这些函数。

**Example**
```cpp
class Tracer {
    string message;
public:
    Tracer(const string& m) : message{m} { cerr << "entering " << message << '\n'; }
    ~Tracer() { cerr << "exiting " << message << '\n'; }

    Tracer(const Tracer&) = default;
    Tracer& operator=(const Tracer&) = default;
    Tracer(Tracer&&) = default;
    Tracer& operator=(Tracer&&) = default;
};
```
因为定义了析构函数，所以拷贝和移动函数也需要定义。=default是最好且最简单的方法。

**Example,bad**
```cpp
class Tracer2 {
    string message;
public:
    Tracer2(const string& m) : message{m} { cerr << "entering " << message << '\n'; }
    ~Tracer2() { cerr << "exiting " << message << '\n'; }

    Tracer2(const Tracer2& a) : message{a.message} {}
    Tracer2& operator=(const Tracer2& a) { message = a.message; return *this; }
    Tracer2(Tracer2&& a) :message{a.message} {}
    Tracer2& operator=(Tracer2&& a) { message = a.message; return *this; }
};
```
这里拷贝和移动操作的代码是冗长的，乏味的且易错的。编译器可以做的更好。

**Enforcement** 特殊函数的主体不应该与编译器生成的代码有相同的访问性和语义，因为这是多余的。

<br/>
### **C.81:如果你想禁用默认语义且不希望有其他替代的话=delete**

**原因** 少数情况下，默认操作是不可取的。

**Example**
```cpp
class Immortal {
public:
    ~Immortal() = delete;   // do not allow destruction
    // ...
};

void use()
{
    Immortal ugh;   // error: ugh cannot be destroyed
    Immortal* p = new Immortal{};
    delete p;       // error: cannot destroy *p
}
```

**Example** unique\_ptr只能移动不能复制，实现方法就是禁用这些操作。避免拷贝就把所有从左值拷贝操作=delete。
```cpp
template<class T, class D = default_delete<T>> class unique_ptr {
public:
    // ...
    constexpr unique_ptr() noexcept;
    explicit unique_ptr(pointer p) noexcept;
    // ...
    unique_ptr(unique_ptr&& u) noexcept;   // move constructor
    // ...
    unique_ptr(const unique_ptr&) = delete; // disable copy from lvalue
    // ...
};

unique_ptr<int> make();   // make "something" and return it by moving

void f()
{
    unique_ptr<int> pi {};
    auto pi2 {pi};      // error: no move constructor from lvalue
    auto pi3 {make()};  // OK, move: the result of make() is an rvalue
}
```
注意=delete的函数应该是public的。

<br/>
### **C.82:不要在构造函数和析构函数中调用virtual函数**

**原因** 调用的函数是目前已经构造完的对象的函数，而不是派生类中可能的重写函数。这令人困惑，更糟糕的是，
从构造函数或析构函数直接或间接调用一个未实现的纯虚函数，结果是未定义的。

**Example,bad**
```cpp
class Base {
public:
    virtual void f() = 0;   // not implemented
    virtual void g();       // implemented with Base version
    virtual void h();       // implemented with Base version
    virtual ~Base();        // implemented with Base version
};

class Derived : public Base {
public:
    void g() override;   // provide Derived implementation
    void h() final;      // provide Derived implementation

    Derived()
    {
        // BAD: attempt to call an unimplemented virtual function
        f();

        // BAD: will call Derived::g, not dispatch further virtually
        g();

        // GOOD: explicitly state intent to call only the visible version
        Derived::g();

        // ok, no qualification needed, h is final
        h();
    }
};
```
注意，调用一个特定的显式限定的函数并不是虚拟调用，即使该函数是virtual的。

注意factory函数如何实现对派生类函数调用而不会产生未定义。

**注意** 从构造函数和析构函数中调用虚函数，本质上没有问题。这种调用是类型安全的，然而经验表明，这种调用很少碰到，且容易让维护者感到困惑。
新手容易误用的来源。

**Enforcement** 构造函数和析构函数中调用虚函数标记出来。

<br/>
### **C.83:对于值类型，考虑提供noexcept的swap函数**

**原因** swap可以方便地实现一些idioms，从平滑地移动对象到轻松地实现赋值，再到提供一个有保证的提交函数，从而实现强防错的调用代码。
考虑使用swap来实现拷贝构造函数的拷贝赋值。另外也参见析构函数、释放分配、swap必须永不失败。

**Example,good**
```cpp
class Foo {
public:
    void swap(Foo& rhs) noexcept
    {
        m1.swap(rhs.m1);
        std::swap(m2, rhs.m2);
    }
private:
    Bar m1;
    int m2;
};
```
在类的相同命名空间中提供一个非成员的swap函数，方便调用。
```cpp
void swap(Foo& a, Foo& b)
{
    a.swap(b);
}
```

**Enforcement**
- 非琐碎的可复制类应该提供一个swap成员函数或swap重载。
- 类有一个swap成员函数时，应该声明为noexcept。

<br/>
### **C.84:swap函数不能失败**

**原因** swap被广泛使用，且被认为永不失败，一旦swap出现失败程序很难正常工作。

**Example,bad**
```cpp
void swap(My_vector& x, My_vector& y)
{
    auto tmp = x;   // copy elements
    x = y;
    y = tmp;
}
```
这不仅慢，而且如果为tmp里的元素发生内存分配，swap可能抛出异常，如果与stl算法一起使用会使stl算法失败。

**Enforcement** 当一个类有swap成员函数时，应该声明为noexcept。

<br/>
### **C.85:swap函数应该设为noexcept**

**原因** 如上一节所讲，swap函数不应该失败。如果swap退出抛出异常，那就是一个糟糕的设计错误，程序最好终止。

**Enforcement** 当一个类有swap成员函数时，应该声明为noexcept。

<br/>
### **C.86:在操作类型和noexcept应该对==有对称性**

**原因** 对操作数的不对称处理是令人惊讶的，也可能是发生转换的错误来源。==是一个基础操作，程序员应该不用担心失败地使用它。

**Example**
```cpp
struct X {
    string name;
    int number;
};

bool operator==(const X& a, const X& b) noexcept {
    return a.name == b.name && a.number == b.number;
}
```

**Example, bad**
```cpp
class B {
    string name;
    int number;
    bool operator==(const B& a) const {
        return name == a.name && number == a.number;
    }
    // ...
};
```
B的比较符接受第二个操作符数的转换，但不接受第一个操作数的转换。

**注意** 如果一个类有一个失败的状态，比如double的NaN，那就会有对失败的状态进行比较，抛出异常。替代方案是，让两个失败的状态比较相等，
任何有效的状态和失败的状态比较为false。

**注意** 本条规则适用于所有常见的比较运算符：!=, <, <=, >, and >=。

**Enforcement** 
- 对于比较操作符函数，如果比较参数类型不一致应该标记出，!=, <, <=, >, and >=操作符也是一样操作。
- 成员函数operator==()应该标记出。!=, <, <=, >, and >=操作符也是一样操作。

<br/>
### **C.87:谨防在基类上==**

**原因** 要为派生关系的层次结构写一个万无一失且有用的==，真的很困难。

**Example,bad**
```cpp
class B {
    string name;
    int number;
public:
    virtual bool operator==(const B& a) const
    {
         return name == a.name && number == a.number;
    }
    // ...
};
```
B接受第二个操作数的转换，但不接受第一个操作数的转换。
```cpp
class D : public B {
    char character;
public:
    virtual bool operator==(const D& a) const
    {
        return B::operator==(a) && character == a.character;
    }
    // ...
};

B b = ...
D d = ...
b == d;    // compares name and number, ignores d's character
d == b;    // compares name and number, ignores d's character
D d2;
d == d2;   // compares name, number, and character
B& b2 = d2;
b2 == d;   // compares name and number, ignores d2's and d's character
```
当然，有很多方法可以让==在层次结构中使用，但无法扩展。

**注意** 此规则适用于所有常用的比较运算符：!=、<、<=、>、>= 和 <=>。

**Enforcement**
- virtual的比较符函数都应该标记出，!=、<、<=、>、>= 和 <=>也是同样。

<br/>
### **C.89:hash函数应该设为noexcept**

**原因** hash容器的用户间接使用了hash，不希望简单的访问会抛出异常，这是标准库的要求。

**Example, bad**
```cpp
template<>
struct hash<My_type> {  // thoroughly bad hash specialization
    using result_type = size_t;
    using argument_type = My_type;

    size_t operator()(const My_type & x) const
    {
        size_t xs = x.s.size();
        if (xs < 4) throw Bad_My_type{};    // "Nobody expects the Spanish inquisition!"
        return hash<size_t>()(x.s.size()) ^ trim(x.s);
    }
};

int main()
{
    unordered_map<My_type, int> m;
    My_type mt{ "asdfg" };
    m[mt] = 7;
    cout << m[My_type{ "asdfg" }] << '\n';
}
```
如果你必须定义一个hash特化，简单地让它和标准库的hash特化和^(xor)结合起来，对于非专业人员来说，这往往更高效。

**Enforcement**
- hash函数如果会抛异常则标记出。

<br/>
### **C.90:依赖构造函数和赋值操作符，而不是memset和memcpy**

**原因** 标准C++机制使用构造函数来构造一个类型实体，正如准则C.4所规定的：一个构造函数应该创建一个完全初始化的对象，而不应该
要求额外的初始化，比如通过memcpy。一个类型提供一个拷贝构造函数和/或拷贝赋值运算符函数，以适当地拷贝类，保留类的不可变性。
使用memcpy来拷贝一个非琐碎的可复制的类未定义，这通常导致切片或数据损坏。

**Example, good**
```cpp
struct base {
    virtual void update() = 0;
    std::shared_ptr<int> sp;
};

struct derived : public base {
    void update() override {}
};
```

**Example,bad**
```cpp
void init(derived& a)
{
    memset(&a, 0, sizeof(derived));
}
```
这样做类型不安全，且会覆盖vtable。

**Example,bad**
```cpp
void copy(derived& a, derived& b)
{
    memcpy(&a, &b, sizeof(derived));
}
```
这样做类型不安全，且会覆盖vtable。

**Enforcement**
- 所有非琐碎的类型拷贝进行memset或memcpy都要标记出。

<br/>
## **C.con:容器及其它资源句柄**
容器是一个容纳某种类型的对象序列的对象，std::vector是典型的容器。资源句柄是一个拥有资源的类，std::vector是典型的资源句柄。
它的资源是它的元素序列。

容器规则摘要：
- [C.100:定义容器时遵循stl惯例](#c100定义容器时遵循stl惯例)
- [C.101:容器应该值语义](#c101容器应该值语义)
- [C.102:容器应该有移动操作](#c102容器应该有移动操作)
- [C.103:容器应该有一个初始化列表构造函数](#c103容器应该有一个初始化列表构造函数)
- [C.104:容器应该有一个默认构造函数将其设置为空](#c104容器应该有一个默认构造函数将其设置为空)
- [C.109:如果一个资源句柄有指针语义，那应该提供\*和->](#c109如果一个资源句柄有指针语义那应该提供和-)

<br/>
### **C.100:定义容器时遵循stl惯例**

**原因** 大多数C++程序员熟悉STL容器，这是个基本的合理设计。

**注意** 特别是，std::vector和std::map提供了有用的相对简单的模型。

**Example**
```cpp
// simplified (e.g., no allocators):

template<typename T>
class Sorted_vector {
    using value_type = T;
    // ... iterator types ...

    Sorted_vector() = default;
    Sorted_vector(initializer_list<T>);    // initializer-list constructor: sort and store
    Sorted_vector(const Sorted_vector&) = default;
    Sorted_vector(Sorted_vector&&) = default;
    Sorted_vector& operator=(const Sorted_vector&) = default;   // copy assignment
    Sorted_vector& operator=(Sorted_vector&&) = default;        // move assignment
    ~Sorted_vector() = default;

    Sorted_vector(const std::vector<T>& v);   // store and sort
    Sorted_vector(std::vector<T>&& v);        // sort and "steal representation"

    const T& operator[](int i) const { return rep[i]; }
    // no non-const direct access to preserve order

    void push_back(const T&);   // insert in the right place (not necessarily at back)
    void push_back(T&&);        // insert in the right place (not necessarily at back)

    // ... cbegin(), cend() ...
private:
    std::vector<T> rep;  // use a std::vector to hold elements
};

template<typename T> bool operator==(const Sorted_vector<T>&, const Sorted_vector<T>&);
template<typename T> bool operator!=(const Sorted_vector<T>&, const Sorted_vector<T>&);
// ...
```
在这里，STL风格被遵循，但不完整。这并不罕见。只提供对特定容器有意义的功能。关键是定义常规的构造函数、赋值、析构函数和迭代器
（对于特定的容器来说是有意义的）及其常规语义。在这个基础上，可以根据需要对容器进行扩展。这里，我们添加了参数为std::vector的特定构造函数。

<br/>
### **C.101:容器应该值语义**

**原因** 常规对象比其他对象更容易思考和推理，因为比较熟悉。

**注意** 如果有意义的话，让一个容器像常规类型（概念上），特别是，确保一个对象和它的拷贝是相等的。

**Example**
```cpp
void f(const Sorted_vector<string>& v)
{
    Sorted_vector<string> v2 {v};
    if (v != v2)
        cout << "Behavior against reason and logic.\n";
    // ...
}
```

<br/>
### **C.102:容器应该有移动操作**

**原因** 容器往往变得很大，如果没有移动构造函数和拷贝构造函数，一个对象的移动就会很昂贵，从而让人们经常用指针传递，这会陷入资源管理问题。

**Example**
```cpp
Sorted_vector<int> read_sorted(istream& is)
{
    vector<int> v;
    cin >> v;   // assume we have a read operation for vectors
    Sorted_vector<int> sv = v;  // sorts
    return sv;
}
```
用户可以合理地认为返回一个类似标准的容器是便宜的。

<br/>
### **C.103:容器应该有一个初始化列表构造函数**

**原因** 人们期望能够用一组值来初始化一个容器，因为熟悉。

**Example**
```cpp
Sorted_vector<int> sv {1, 3, -1, 7, 0, 0}; // Sorted_vector sorts elements as needed
```

<br/>
### **C.104:容器应该有一个默认构造函数将其设置为空**

**原因** 常规类型都这样。

**Example**
```cpp
vector<Sorted_sequence<string>> vs(100);    // 100 Sorted_sequences each with the value ""
```
<br/>
### **C.109:如果一个资源句柄有指针语义，那应该提供*和->**

**原因** 跟人们使用指针一样情况，因为熟悉。

<br/>
## **C.lambdas:函数对象和lambdas**
一个函数对象是一个提供重载()的对象，以便你可以调用它。一个lambda表达式是生成一个函数对象的notation。函数对象的复制成本应该很低（因此通过值传递）。
摘要：
- [F.10:如果一个操作可以被重复使用，就该给它一个名字](#f10如果一个操作可以被重复使用就该给它一个名字)
- [F.11:如果你只在一个地方需要一个简单的函数对象，那应该使用未命名的lambda](#f11如果你只在一个地方需要一个简单的函数对象那应该使用未命名的lambda)
- [F.50:当函数不能做的时候（捕捉局部变量，编写局部函数），使用lambda](#f50当函数不能做的时候捕捉局部变量编写局部函数使用lambda)
- [F.52:倾向于在局部使用的lambdas中通过引用来捕获，包括传递给算法](#f52倾向于在局部使用的lambdas中通过引用来捕获包括传递给算法)
- [F.53:避免在将被非本地使用的lambdas中通过引用来捕获，包括返回、存储在堆上或传递给另一个线程。](#f53避免在将被非本地使用的lambdas中通过引用来捕获包括返回存储在堆上或传递给另一个线程)
- [ES.28:用lambda来做复杂的初始化，特别是const变量](#es用lambda来做复杂的初始化特别是const变量)

<br/>
## **C.hier:类的分层（OOP）**
构建类的层次结构是为了代表一组分层组织的概念（仅仅）。通常情况下，基类充当接口。层次结构有两个主要用途，通常被成为实现继承和接口继承。

类层次结构规则摘要：
- [C.120:使用类的层次结构来表示内在层次结构的概念](#c120使用类的层次结构来表示内在层次结构的概念)
- [C.121:如果基类用作接口，则将其设为纯抽象类](#c121如果基类用作接口则将其设为纯抽象类)
- [C.122:当需要接口和实现完全分离时，使用抽象类作为接口](#c122当需要接口和实现完全分离时使用抽象类作为接口)

为层次结构中的类设计规则摘要：
- [C.126:抽象类通常不需要用户编写的构造函数](#c126抽象类通常不需要用户编写的构造函数)
- [C.127:一个有虚函数的类应该有一个虚析构函数或protected析构函数](#c127一个有虚函数的类应该有一个虚析构函数或protected析构函数)
- [C.128:虚函数应该指定virtual、override和final其中之一](#c128虚函数应该指定virtualoverride和final其中之一)
- [C.129:设计类的层次结构时，要区分实现继承和接口继承](#c129设计类的层次结构时要区分实现继承和接口继承)
- [C.130:对于多态类的深度拷贝，优先使用virtual clone函数，而不是public拷贝构造和赋值](#c130对于多态类的深度拷贝优先使用virtual-clone函数而不是public拷贝构造和赋值)
- [C.131:避免简单琐碎的getter和setter](#c131避免简单琐碎的getter和setter)
- [C.132:不要无故把函数设为virtual](#c132不要无故把函数设为virtual)
- [C.133:避免protected数据](#c133避免protected数据)
- [C.134:确保所有非const数据成员具有相同的访问级别](#c134确保所有非const数据成员具有相同的访问级别)
- [C.135:使用多重继承来代表多个不同的接口](#c135使用多重继承来代表多个不同的接口)
- [C.136:使用多重继承来代表实现属性的union](#c136使用多重继承来代表实现属性的union)
- [C.137:使用virtual基类来避免过于笼统的基类](#c137使用virtual基类来避免过于笼统的基类)
- [C.138:用using为派生类和它的基类创建一个重载集](#c138用using为派生类和它的基类创建一个重载集)
- [C.139:尽量少在类上使用final](#c139尽量少在类上使用final)
- [C.140:不要在虚函数和重写函数上提供不同的参数](#c140不要在虚函数和重写函数上提供不同的参数)

访问层次结构中的对象规则摘要：
- [C.145:通过指针和引用来访问多态对象](#c145通过指针和引用来访问多态对象)
- [C.146:类的层次结构导航不可避免情况下，使用dynamic\_cast](#c146类的层次结构导航不可避免情况下使用dynamic_cast)
- [C.147:dynamic\_cast一个引用类型而找不到所需的类时，应该认定为错误](#c147dynamic_cast一个引用类型而找不到所需的类时应该认定为错误)
- [C.148:dynamic\_cast一个指针类型而找不到所需的类时，应该认定为一个有效的选择](#c148dynamic_cast一个指针类型而找不到所需的类时应该认定为一个有效的选择)
- [C.149:使用unique\_ptr或shared\_ptr来避免忘记删除使用new创建的对象](#c149使用unique_ptr或shared_ptr来避免忘记删除使用new创建的对象)
- [C.150:使用make\_unique()来构建unique\_ptr所拥有的对象](#c150使用make_unique来构建unique_ptr所拥有的对象)
- [C.151:使用make\_shared()来构建shared\_ptr所拥有的对象](#c151使用make_shared来构建shared_ptr所拥有的对)
- [C.152:永远不要把派生类对象数组的指针分配给其基类的指针](#c152永远不要把派生类对象数组的指针分配给其基类的指针)
- [C.153:优先使用virtual函数而不是cast](#c153优先使用virtual函数而不是cast)

<br/>
### **C.120:使用类的层次结构来表示内在层次结构的概念**

**原因** 在代码中直接表达所想便于理解和维护。确保基类表示的想法和派生类完全匹配，没有比使用继承的紧耦合更好的表达方式了。
当只拥有一个数据成员就可以时，不要使用继承。通常这意味着派生类需要覆盖基类虚函数或访问protected成员。

**Example**
```cpp
class DrawableUIElement {
public:
    virtual void render() const = 0;
    // ...
};

class AbstractButton : public DrawableUIElement {
public:
    virtual void onClick() = 0;
    // ...
};

class PushButton : public AbstractButton {
    void render() const override;
    void onClick() override;
    // ...
};

class Checkbox : public AbstractButton {
// ...
};
```

**Example, bad** 不要将非层次结构领域的概念表示为类层次结构。
```cpp
template<typename T>
class Container {
public:
    // list operations:
    virtual T& get() = 0;
    virtual void put(T&) = 0;
    virtual void insert(Position) = 0;
    // ...
    // vector operations:
    virtual T& operator[](int) = 0;
    virtual void sort() = 0;
    // ...
    // tree operations:
    virtual void balance() = 0;
    // ...
};
```
在这里，大多数重写类不能很好地实现接口中所要求的大部分功能。因此，基类变成了一个实现的负担。此外，Container的用户不能依靠成员函数实际有效地执行有意义的操作。
它可能抛出异常，用户不得不求助于运行时检查和/或不使用这个（过度）通用的接口，而选择通过运行时类型查询（例如，dynamic\_cast）找到的特定接口。

**Enforcement**
- 找出有很多成员的类，什么都不做，但是会抛出异常。
- 对于每个非共有继承B的派生类D，其中D没有重写一个虚函数或者访问B中的protected成员，而且B不属于以下情况：空、D的模版参数或参数包和D的特化类模版，都应该标记出。

<br/>
### **C.121:如果基类用作接口，则将其设为纯抽象类**

**原因** 类如果不含数据，就会比较稳定（不那么脆），接口通常应该完全由public的纯虚函数和一个默认/空的虚析构函数构成。

**Example**
```cpp
class My_interface {
public:
    // ...only pure virtual functions here ...
    virtual ~My_interface() {}   // or =default
};
```

**Example, bad**
```cpp
class Goof {
public:
    // ...only pure virtual functions here ...
    // no virtual destructor
};

class Derived : public Goof {
    string s;
    // ...
};

void use()
{
    unique_ptr<Goof> p {new Derived{"here we go"}};
    f(p.get()); // use Derived through the Goof interface
    g(p.get()); // use Derived through the Goof interface
} // leak
```
Derived对象是通过Goof接口来delete的，所以它的string泄漏了，应该把Goof析构函数改为virtual的。

**Enforcement**
- 类包含数据成员且有一个没有从基类继承的可重写的（非final）虚函数，应该发出警告。

<br/>
### **C.122:当需要接口和实现完全分离时，使用抽象类作为接口**

**原因** 如在ABI（链接）边界上。

**Example**
```cpp
struct Device {
    virtual ~Device() = default;
    virtual void write(span<const char> outbuf) = 0;
    virtual void read(span<char> inbuf) = 0;
};

class D1 : public Device {
    // ... data ...

    void write(span<const char> outbuf) override;
    void read(span<char> inbuf) override;
};

class D2 : public Device {
    // ... different data ...

    void write(span<const char> outbuf) override;
    void read(span<char> inbuf) override;
};
```
用户可以通过Device提供的接口互换地使用D1和D2，此外我们可以用跟旧版本二进制不兼容的方式更新D1和D2，只有所有的访问都通过Device接口进行。

<br/>
## **C.hierclass:在一个层次结构中设计类**

<br/>
### **C.126:抽象类通常不需要用户编写的构造函数**

**原因** 一个抽象类通常没有任何数据可以让构造函数初始化。

**Example**
```cpp
class Shape {
public:
    // no user-written constructor needed in abstract base class
    virtual Point center() const = 0;    // pure virtual
    virtual void move(Point to) = 0;
    // ... more pure virtual functions...
    virtual ~Shape() {}                 // destructor
};

class Circle : public Shape {
public:
    Circle(Point p, int rad);           // constructor in derived class
    Point center() const override { return x; }
};
```

**例外**
- 一个有具体工作的基类构造函数，比如在某个地方注册一个对象，那可能需要一个构造函数。
- 在极其罕见情况下，你可能会发现一个抽象类拥有所有派生类共享的一点数据是合理的（例如，使用统计数据、调试信息等）。
这种类往往有构造函数，但要注意的是，这样的类也往往容易要求虚继承。

**Enforcement** 抽象类有构造函数要标记出来。

<br/>
### **C.127:一个有虚函数的类应该有一个虚析构函数或protected析构函数**

**原因** 拥有虚函数的类通常是通过基类指针使用，最后一个使用者在基类指针上delete（一般通过智能指针），那么析构函数必须是
virtual且public的。不太常见的是，如果不打算通过基类指针进行delete，那么析构函数应该是protected的且非virtual的，参见C.53。

**Example, bad**
```cpp
struct B {
    virtual int f() = 0;
    // ... no user-written destructor, defaults to public non-virtual ...
};

// bad: derived from a class without a virtual destructor
struct D : B {
    string s {"default"};
    // ...
};

void use()
{
    unique_ptr<B> p = make_unique<D>();
    // ...
} // undefined behavior, might call B::~B only and leak the string
```

**注意** 有人不遵循这个规则，因为它们打算只通过shared\_ptr来使用一个类，std::share\_ptr\<B\> p = std::make\_shared\<D\>(args);这里，
共享指针会处理好delete，所以不会因为基类不适当delete而发生泄漏。但一直这样做的人会得到false positive，如果有人改为make\_unique分配呢？
除非确保B的使用者不会滥用，比如让构造函数私有，并提供一个工厂函数来强制使用make\_shared分配。

**Enforcement**
- 一个有任何虚函数的类的析构函数，要么是public和virtual的，要么是protected和non-virtual的。
- 如果delete一个含虚函数的类，但是类没有虚析构函数，应该标记出。

<br/>
### **C.128:虚函数应该指定virtual、override和final其中之一**

**原因** 可读性。错误检测。显式写出virtual、override和final是self-documenting，编译器可以捕获基类和派生来的类型或名字不匹配。
然而如果三个中指定了两个以上是多余的，也是潜在错误来源。
- virtual明确表示这是一个新的虚函数。
- override明确表示这个是一个non-final重写。
- final明确表示这是最后一个重写。

**Example, bad**
```cpp
struct B {
    void f1(int);
    virtual void f2(int) const;
    virtual void f3(int);
    // ...
};

struct D : B {
    void f1(int);        // bad (hope for a warning): D::f1() hides B::f1()
    void f2(int) const;  // bad (but conventional and valid): no explicit override
    void f3(double);     // bad (hope for a warning): D::f3() hides B::f3()
    // ...
};
```

**Example,good**
```cpp
struct Better : B {
    void f1(int) override;        // error (caught): Better::f1() hides B::f1()
    void f2(int) const override;
    void f3(double) override;     // error (caught): Better::f3() hides B::f3()
    // ...
};
```

**讨论**
我们想消除两类特殊的错误：
- **implicit virtual:** 程序员打算让该函数隐含虚拟，而它确实是（但代码的读者无法分辨）；或者程序员打算让该函数隐含虚拟，但它不是（例如，由于参数列表的微妙不匹配）；或者程序员没有打算让该函数隐含虚拟，但它确实是（因为它刚好与基类中的虚拟有相同的签名） 
- **implicit override:** 程序员打算让这个函数隐含地成为一个覆盖者，而它确实是（但代码的读者无法分辨）；或者程序员打算让这个函数隐含地成为一个覆盖者，但它不是（例如。因为一个微妙的参数列表不匹配）；或者程序员并不打算让该函数成为一个覆盖者，但它确实是（因为它恰好与基类中的虚函数有相同的签名--注意这个问题的出现，无论该函数是否明确声明为虚，因为程序员可能打算创建一个新的虚函数或一个新的非虚函数）

注意：在一个定义为final的类上，你在单个虚函数上override或final并不重要。

注意：尽量少在函数上使用final，它不一定会得到优化，但它排除了进一步重写的可能性。

**Enforcement**
- 比较基类和派生类中虚函数名字，没有override的同名要标记出来。
- 重写函数没有override或final的要标记出。
- 函数声明超过2个（virtual,override,final)的要标记出。

<br/>
### **C.129:设计类的层次结构时，要区分实现继承和接口继承**

**原因** 接口中的实现细节会使接口变得很脆弱，也就是说，如果实现要变化，那就不得不重新编译代码。基类中有数据增加了基类实现复杂性，并可能导致代码复制。

**注意** 定义：
- 接口继承利用继承将接口和实现分离，特别是允许派生类添加接口而不影响基类用户使用。
- 实现继承利用继承来简化新设施的实现，方法是让新设施相关的实现者可以使用有用的操作。

一个纯接口类是一组纯虚函数集合。见I.25。

在早期的OOP中（如20世纪80年代和90年代），实现继承和接口继承往往是混合的，坏习惯是很难改的。即使是现在，在旧的代码库和旧式的教材中，混合的情况也并不少见。

**Example,bad**
```cpp
class Shape {   // BAD, mixed interface and implementation
public:
    Shape();
    Shape(Point ce = {0, 0}, Color co = none): cent{ce}, col {co} { /* ... */}

    Point center() const { return cent; }
    Color color() const { return col; }

    virtual void rotate(int) = 0;
    virtual void move(Point p) { cent = p; redraw(); }

    virtual void redraw();

    // ...
private:
    Point cent;
    Color col;
};

class Circle : public Shape {
public:
    Circle(Point c, int r) : Shape{c}, rad{r} { /* ... */ }

    // ...
private:
    int rad;
};

class Triangle : public Shape {
public:
    Triangle(Point p1, Point p2, Point p3); // calculate center
    // ...
};
```
问题有：
- 随着类分层结构增加，更多的数据将会添加到Shape，构造函数变得难写且难维护。
- 为什么要计算矩形的center，我们可能根本不会用到它。
- Shape增加一个成员（比如绘画样式或画布），所有派生类都要被review，可能会被修改，也可能要被重新编译。
Shape::move()是实现继承的例子，我们为所有派生类一次性定义了move()，这样基类成员函数实现的代码越多，
将数据放在基类中共享的越多，我们获得好处就越多--这样类层级结构稳定性就越差。

**Example** 用接口继承重写:
```cpp
class Shape {  // pure interface
public:
    virtual Point center() const = 0;
    virtual Color color() const = 0;

    virtual void rotate(int) = 0;
    virtual void move(Point p) = 0;

    virtual void redraw() = 0;

    // ...
};
```
纯虚函数没有构造函数，因为它没有啥数据需要构造。
```cpp
class Circle : public Shape {
public:
    Circle(Point c, int r, Color c) : cent{c}, rad{r}, col{c} { /* ... */ }

    Point center() const override { return cent; }
    Color color() const override { return col; }

    // ...
private:
    Point cent;
    int rad;
    Color col;
};
```
这样的接口就不那么脆弱了，但在成员函数的实现上有更多的工作。例如每一个派生类都必须实现center。

**Example, dual hierarchy** 我们怎样才能从实现层次结构中获得稳定层次结构的好处，并从实现继承中获得实现重用的好处？
一种流行的技术是双层次结构。有很多方法可以实现双层次结构的想法；在这里，我们使用一个多继承的变体。
```cpp
class Shape {   // pure interface
public:
    virtual Point center() const = 0;
    virtual Color color() const = 0;

    virtual void rotate(int) = 0;
    virtual void move(Point p) = 0;

    virtual void redraw() = 0;

    // ...
};

class Circle : public virtual Shape {   // pure interface
public:
    virtual int radius() = 0;
    // ...
};
```
为了使这个接口有用，我们必须提供它的实现类（这里以等价方式命名，但在Impl命名空间）。
```cpp
class Impl::Shape : public virtual ::Shape { // implementation
public:
    // constructors, destructor
    // ...
    Point center() const override { /* ... */ }
    Color color() const override { /* ... */ }

    void rotate(int) override { /* ... */ }
    void move(Point p) override { /* ... */ }

    void redraw() override { /* ... */ }

    // ...
};
```
```cpp
class Impl::Circle : public virtual ::Circle, public Impl::Shape {   // implementation
public:
    // constructors, destructor

    int radius() override { /* ... */ }
    // ...
};
```
```cpp
class Smiley : public virtual Circle { // pure interface
public:
    // ...
};

class Impl::Smiley : public virtual ::Smiley, public Impl::Circle {   // implementation
public:
    // constructors, destructor
    // ...
}
```
现在有两个继承：
- 接口：Smiley -> Circle -> Shape
- 实现：Impl::Smiley -> Impl::Circle -> Impl::Shape
由于每个实现都是由其接口以及实现基类派生出来的，所以我们得到了一个网格（DAG）。

```cpp
Smiley     ->         Circle     ->  Shape
  ^                     ^               ^
  |                     |               |
Impl::Smiley -> Impl::Circle -> Impl::Shape
```
如前所述，这只是构建二元层次结构的一种方式。
实现层次结构可以直接使用，而不是通过抽象接口。
```cpp
void work_with_shape(Shape&);

int user()
{
    Impl::Smiley my_smiley{ /* args */ };   // create concrete shape
    // ...
    my_smiley.some_member();        // use implementation class directly
    // ...
    work_with_shape(my_smiley);     // use implementation through abstract interface
    // ...
}
```

**注意** 另外一种接口实现分离的技术是Pimpl。

**Enforcement**
- 一个基类同时有数据和虚函数，从这样的基类继承的派生类到基类转换要标记出来（除了从派生类成员调用到基类成员调用）。

<br/>
### **C.130:对于多态类的深度拷贝，优先使用virtual clone函数，而不是public拷贝构造和赋值**

**原因** 因为切片问题，不鼓励多态类的拷贝，参见C67。如果你真的需要拷贝语义，那么要深拷贝。提供一个virtual clone函数，
它拷贝实际的最外层的派生类，并返回一个指向新对象的所有权指针，然后在派生类中返回派生类型。

**Example**
```cpp
class B {
public:
    B() = default;
    virtual ~B() = default;
    virtual gsl::owner<B*> clone() const = 0;
protected:
     B(const B&) = default;
     B& operator=(const B&) = default;
     B(B&&) = default;
     B& operator=(B&&) = default;
    // ...
};

class D : public B {
public:
    gsl::owner<D*> clone() const override
    {
        return new D{*this};
    };
};
```
一般来说，建议使用智能指针来表示所有权（见R.20）。然而由于语言规则，共同的返回类型不能是一个智能指针：
D::clone不能返回unique\_ptr\<D\>，而B::clone返回unique\_ptr\<B\>。因此你需要在所有重写中一致地返回
unique\_ptr\<B\>，或者使用GSL中的owner工具。

<br/>
### **C.131:避免简单琐碎的getter和setter**

**原因** 琐碎的setter和getter不增加任何语义价值，数据可以只设置public就可以。

**Example**
```cpp
class Point {   // Bad: verbose
    int x;
    int y;
public:
    Point(int xx, int yy) : x{xx}, y{yy} { }
    int get_x() const { return x; }
    void set_x(int xx) { x = xx; }
    int get_y() const { return y; }
    void set_y(int yy) { y = yy; }
    // no behavioral member functions
};
```
考虑把这个类改为struct，也就是一堆无行为的变量，全部是public变量且没有成员变量。
```cpp
struct Point {
    int x {0};
    int y {0};
};
```
注意我们可以把默认的初始化放在成员变量上。参见C.49。

**注意** 这个规则的关键是getter/setter的语义是否微不足道。虽然这不是 "微不足道 "的完整定义，但考虑如果getter/setter是一个公共数据成员，
除了语法之外是否会有任何区别。非微不足道的语义的例子是：保持类的不变性或在内部类型和接口类型之间进行转换。

<br/>
### **C.132:不要无故把函数设为virtual**

**原因** 冗余的virtual函数增加了运行时开销和对象代码大小。虚函数可以被重写，因此容易在派生类中用错。
一个虚函数确保了在一个模版化的层次结构中的代码复制。

**Example, bad**
```cpp
template<class T>
class Vector {
public:
    // ...
    virtual int size() const { return sz; }   // bad: what good could a derived class do?
private:
    T* elem;   // the elements
    int sz;    // number of elements
};
```
这个类型的Vector根本就不是为了作为基类使用。

**Enforcement**
- 有虚函数但没有派生类的都要标记出。
- 类所有成员是virtual并有实现的类都要标记出。

<br/>
### **C.133:避免protected数据**

**原因** protected数据是复杂性和错误的来源，protected数据是的不变性声明变得复杂，protected数据本质上违反了禁止将数据放在基类中的指导原则，
这通常导致不得不处理虚拟继承。

**Example, bad**
```cpp
class Shape {
public:
    // ... interface functions ...
protected:
    // data for use in derived classes:
    Color fill_color;
    Color edge_color;
    Style st;
};
```
现在要靠每个派生的Shape来正确操作受保护的数据。这很受欢迎，但也是维护问题的主要来源。在一个大的类层次结构中，
受保护数据的一致使用是很难维护的，因为可能有很多代码，分布在很多类中。可以接触这些数据的类的集合是开放的：
任何人都可以派生出一个新的类并开始操作受保护的数据。通常情况下，我们不可能检查完整的类集，所以对类的表示的任何改变都是不可行的。
受保护数据没有强制的不变性；它很像一组全局变量。受保护的数据事实上已经成为一大批代码的全局。

**注意** protected数据通常看起来很容易通过派生类来实现任意改进。但经常得到的是无原则的改变和错误。优先考虑具有明确指定和强制执行的不变量的私有数据。
另一种方法，通常也是更好的方法是，将数据排除在用作接口的任何类之外。

**注意** protected成员函数就OK。

**Enforcement** 类有protected成员应该标记出。

<br/>
### **C.134:确保所有非const数据成员具有相同的访问级别**

**原因** 防止逻辑混乱导致错误。如果非const数据成员没有相同的访问等级，那么这个类型就会对他所要做的事情感到困惑。
它是一个维护不变量的类型，还是仅仅是一个值的集合？

**讨论** 核心问题是：什么代码负责该变量有意义且正确？
确切的说有两种类型的数据成员：
- A:不参与对象的不变性的那些成员。这些成员的值的任何组合都是有效的。
- B:那些确实参与了对象的不变性的值。不是每一个值的组合都是有意义的（否则就没有不变性了）。
因此，所有对这些变量有写入权限的代码都必须知道不变性，知道语义，并知道（并积极实现和执行）保持数值正确的规则。

A类数据成员应该是public的（或者如果你只想在派生类中看到它们的话，设为protected）。它们不需要封装。系统中的所有代码都可以看到并操作它们。

B类中的数据成员应该是私有的或const的。这因为封装很重要。
B类中的数据成员应该是私有的或常量的。这是因为封装很重要。让它们成为非私有和非常量意味着对象不能控制自己的状态。
类之外的大量代码需要知道这个不变性并准确地参与维护它--如果这些数据成员是公开的，那就是所有使用该对象的调用代码；
如果它们是保护的，那就是当前和未来派生类中的所有代码。这导致了脆性和紧耦合的代码，很快就会成为维护的噩梦。
任何不经意间将数据成员设置为无效或意外的数值组合的代码都会破坏该对象和该对象的所有后续使用。

大多数的类要么全是A要么全是B：
- 全是public的。如果你在写变量聚合包，而这些变量之间没有不变量，那么所有的变量都应该是public的，按照管理，
应该声明为struct而不是class。
- 全是private的。如果你在写个保持不变的类型。所有的非const变量应该都是private的，它应该被封装起来。

**Enforcement** 任何具有不同访问级别的非const数据成员的类都要标记出来。

<br/>
### **C.135:使用多重继承来代表多个不同的接口**

**原因** 并非所有类都必然支持所有接口，并非所有调用者都必然要处理所有操作。特别是将整体接口分解为给定派生类支持的行为“方面”。

**Example**
```cpp
class iostream : public istream, public ostream {   // very simplified
    // ...
};
```
istream 提供了输入操作的接口； ostream 提供输出操作的接口。 iostream 提供了 istream 和 ostream 接口的联合，以及在一个流中允许两者的同步。

**注意** 这是继承的一种非常普遍的用法，因为一个实现需要多个不同的接口，而这种接口往往不容易或不自然地组织成一个单一的层次结构。

**注意** 这样的接口通常是抽象类。

<br/>
### **C.136:使用多重继承来代表实现属性的union**

**原因** 某些形式的mixins具有状态并且通常在该状态上进行操作。如果操作是虚拟的，则使用继承是必要的，如果不使用继承可以避免模板和转发。

**Example**
```cpp
class iostream : public istream, public ostream {   // very simplified
    // ...
};
```
istream 提供了输入操作的接口； ostream 提供输出操作的接口。 iostream 提供了 istream 和 ostream 接口的联合，以及在一个流中允许两者的同步。

**注意** 这是一个相对少见的用法，因为实现通常可以组织成单根层次结构。

**Example** 有时，“实现属性”更像是一个“mixin”，它确定实现的行为并注入成员以实现它所需的策略。例如，参见std::enable\_shared\_from\_this
或来自boost.intrusive的各种基础（例如list\_base\_hook或intrusive\_ref\_counter）。

<br/>
### **C.137:使用virtual基类来避免过于笼统的基类**

**原因** 允许共享数据和接口分离。避免将所有共享数据放入终极基类。

**Example**
```cpp
struct Interface {
    virtual void f();
    virtual int g();
    // ... no data here ...
};

class Utility {  // with data
    void utility1();
    virtual void utility2();    // customization point
public:
    int x;
    int y;
};

class Derive1 : public Interface, virtual protected Utility {
    // override Interface functions
    // Maybe override Utility virtual functions
    // ...
};

class Derive2 : public Interface, virtual protected Utility {
    // override Interface functions
    // Maybe override Utility virtual functions
    // ...
};
```
如果许多派生类共享重要的“实现细节”，那么分解Utility是有意义的。

**注意** 层次结构的线性化是一个更好的解决方案。

**Enforcement** 混合接口和实现层次结构应该被标记出来。

<br/>
### **C.138:用using为派生类和它的基类创建一个重载集**

**原因** 如果不用using声明，派生类的成员函数会隐藏整个继承的重载集。

**Example, bad**
```cpp
#include <iostream>
class B {
public:
    virtual int f(int i) { std::cout << "f(int): "; return i; }
    virtual double f(double d) { std::cout << "f(double): "; return d; }
    virtual ~B() = default;
};
class D: public B {
public:
    int f(int i) override { std::cout << "f(int): "; return i + 1; }
};
int main()
{
    D d;
    std::cout << d.f(2) << '\n';   // prints "f(int): 3"
    std::cout << d.f(2.3) << '\n'; // prints "f(int): 3"
}
```

**Example,good**
```cpp
class D: public B {
public:
    int f(int i) override { std::cout << "f(int): "; return i + 1; }
    using B::f; // exposes f(double)
};
```

**注意** 这个问题对虚拟和非虚拟成员函数都有影响，对于可变基数，C++17引入了可变形式的using声明。
```cpp
template<class... Ts>
struct Overloader : Ts... {
    using Ts::operator()...; // exposes operator() from every base
};
```

**Enforcement** 名称隐藏警告。

<br/>
### **C.139:尽量少在类上使用final**

**原因** 出于逻辑原因，很少需要使用最终类来限制层次结构，并且可能会破坏层次结构的可扩展性。

**Example, bad**
```cpp
class Widget { /* ... */ };

// nobody will ever want to improve My_widget (or so you thought)
class My_widget final : public Widget { /* ... */ };

class My_improved_widget : public My_widget { /* ... */ };  // error: can't do that
```

**注意** 并非每个类都是为了成为基类。大多数标准库类都是这样的例子（例如，std::vector 和 std::string 不是设计来派生的）。
这条规则是关于在具有虚函数的类上使用final，这些虚函数意在成为一个类层次结构的接口。

**注意** 用final封顶单个虚函数很容易出错，因为在定义/覆盖一组函数时很容易忽略final。幸运的是，编译器捕获了这样的错误：您不能在派生类中重新声明/重新打开final成员。

**注意** 关于final的性能改进的说法应该得到证实。很多时候，这种说法都是基于猜想或其他语言的经验。
有一些例子表明，final对于逻辑和性能的原因都很重要。一个例子是编译器或语言分析工具中对性能至关重要的AST层次。
新的派生类不是每年都添加的，而且只由库实现者添加。然而，滥用是（或至少已经是）更常见的。

**Enforcement** 在类上使用final应该标记出。

<br/>
### **C.140:不要在虚函数和重写函数上提供不同的参数**

**原因** 这可能会引起混淆。一个overrider不继承默认参数。

**Example,bad**
```cpp
class Base {
public:
    virtual int multiply(int value, int factor = 2) = 0;
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    int multiply(int value, int factor = 10) override;
};

Derived d;
Base& b = d;

b.multiply(10);  // these two calls will call the same function but
d.multiply(10);  // with different arguments and so different results
```

**Enforcement** 虚拟函数的默认参数在基类声明和派生类声明不同，则标记出。

<br/>
## **C.hier-access:访问分层结构中的对象**

<br/>
### **C.145:通过指针和引用来访问多态对象**

**原因** 如果你有一个带虚函数的类，一般情况下你不知道哪个类提供的函数要被使用。

**Example**
```cpp
struct B { int a; virtual int f(); virtual ~B() = default };
struct D : B { int b; int f() override; };

void use(B b)
{
    D d;
    B b2 = d;   // slice
    B b3 = b;
}

void use2()
{
    D d;
    use(d);   // slice
}
```
两个d都被切片。

**例外** 你可以在一个命名的多态对象定义范围内访问，只是不是切片使用。
```cpp
void use3()
{
    D d;
    d.f();   // OK
}
```

另外见[C.67:多态类应该抑制public的拷贝或移动](#c7多态类应该抑制public的拷贝或移动)

**Enforcement** 所有切片都应该标记出。

<br/>
### **C.146:类的层次结构导航不可避免情况下，使用dynamic_cast**

**原因** dynamic\_cast运行时才检测。

**Example**
```cpp
struct B {   // an interface
    virtual void f();
    virtual void g();
    virtual ~B();
};

struct D : B {   // a wider interface
    void f() override;
    virtual void h();
};

void user(B* pb)
{
    if (D* pd = dynamic_cast<D*>(pb)) {
        // ... use D's interface ...
    }
    else {
        // ... make do with B's interface ...
    }
}
```
使用其他类型转换可能会违反类型安全，导致程序访问一个实际属于X类型的变量时，会被当作不相关的Z类型来访问。
```cpp
void user2(B* pb)   // bad
{
    D* pd = static_cast<D*>(pb);    // I know that pb really points to a D; trust me
    // ... use D's interface ...
}

void user3(B* pb)    // unsafe
{
    if (some_condition) {
        D* pd = static_cast<D*>(pb);   // I know that pb really points to a D; trust me
        // ... use D's interface ...
    }
    else {
        // ... make do with B's interface ...
    }
}

void f()
{
    B b;
    user(&b);   // OK
    user2(&b);  // bad error
    user3(&b);  // OK *if* the programmer got the some_condition check right
}
```

**注意** 像其他cast一样，dynamic\_cast被过度使用了。优先使用虚函数而不是转换。尽可能使用静态多态性，而不是层次结构导航，
因为不需要运行时解析。

**注意** 有些人在使用typeid更合适的地方使用dynamic\_cast；dynamic\_cast是一个通用的 "是一种 "操作，用于发现一个对象的最佳接口，
而typeid是一个 "给我这个对象的确切类型 "的操作，用于发现一个对象的实际类型。后者是一个本质上更简单的操作，应该更快。
后者（typeid）在必要时很容易手写（例如，如果在一个由于某种原因禁止RTTI的系统上工作），前者（dynamic\_cast）在一般情况下很难正确实现。
```cpp
struct B {
    const char* name {"B"};
    // if pb1->id() == pb2->id() *pb1 is the same type as *pb2
    virtual const char* id() const { return name; }
    // ...
};

struct D : B {
    const char* name {"D"};
    const char* id() const override { return name; }
    // ...
};

void use()
{
    B* pb1 = new B;
    B* pb2 = new D;

    cout << pb1->id(); // "B"
    cout << pb2->id(); // "D"


    if (pb1->id() == "D") {         // looks innocent
        D* pd = static_cast<D*>(pb1);
        // ...
    }
    // ...
}
```
pb2-\>id() == "D"的结果实际上是实现定义的，我们添加它是为了警告自己实现的RTTI的危险操作，这段代码可能会像预期的那样工作多年，
但在新的机器、新的编译器或新的链接器上就会失效，因为新的编译器没有统一的字符不变量。
如果你实现你自己的RTTI，要小心。

**Enforcement**
- 所有向下转换的static\_cast都要标记出，包括执行static\_cast的C风格转换。
- 这个规则是类型安全配置的一部分

<br/>
### **C.147:dynamic_cast一个引用类型而找不到所需的类时，应该认定为错误**

**原因** 引用的cast意味着你最终要一个有效的转换对象，所以必须cast成功，如果失败，dynamic\_cast会抛出异常。

**Example**
```cpp
std::string f(Base& b)
{
    return dynamic_cast<Derived&>(b).to_string();
}
```

<br/>
### **C.148:dynamic_cast一个指针类型而找不到所需的类时，应该认定为一个有效的选择**

**原因** dynamic\_cast转换允许测试指针是否指向在其层次结构中具有给定类的多态对象。
由于找不到类仅返回空值，因此可以在运行时对其进行测试。这允许编写可以根据结果选择替代路径的代码。 
跟[C.147](c147dynamiccast一个引用类型而找不到所需的类时应该认定为错误)相反，失败是个错误，不能用于条件执行。

**Example** 下面的例子描述了一个Shape\_owner的添加功能，该功能接收构建的Shape对象的所有权。
这些对象也根据其几何属性被分类到视图中。在这个例子中，Shape并没有继承Geometric\_attributes。只有它的子类才继承。
```cpp
void add(Shape* const item)
{
  // Ownership is always taken
  owned_shapes.emplace_back(item);

  // Check the Geometric_attributes and add the shape to none/one/some/all of the views

  if (auto even = dynamic_cast<Even_sided*>(item))
  {
    view_of_evens.emplace_back(even);
  }

  if (auto trisym = dynamic_cast<Trilaterally_symmetrical*>(item))
  {
    view_of_trisyms.emplace_back(trisym);
  }
}
```

**注意** 如果不能找到所需的类，将导致dynamic\_cast返回一个null，而null解引用的结果是未定义的，
因此，dynamic\_cast的结果应该始终被视为可能包含一个空值，并进行测试。

**Enforcement**
- 除非对一个指针类型的dynamic\_cast的结果有一个判空测试，否则在解指针引用时发出警告。

<br/>
### **C.149:使用unique_ptr或shared_ptr来避免忘记删除使用new创建的对象**

**原因** 避免资源泄漏。

**Example**
```cpp
void use(int i)
{
    auto p = new int {7};           // bad: initialize local pointers with new
    auto q = make_unique<int>(9);   // ok: guarantee the release of the memory-allocated for 9
    if (0 < i) return;              // maybe return and leak
    delete p;                       // too late
}
```

**Enforcement**
- 用裸指针初始化new的结果都要标记出。
- delete局部变量标记出。

<br/>
### **C.150:使用make_unique()来构建unique_ptr所拥有的对象**

参见[R.23](#r23使用make_unique来获取unique_ptr)

<br/>
### **C.151:使用make_shared()来构建shared_ptr所拥有的对**

参见[R.22](#r22用make_shared来获取shared_ptr)

<br/>
### **C.152:永远不要把派生类对象数组的指针分配给其基类的指针**

**原因** 对产生的基指针进行下标会导致无效的对象访问，并可能导致内存损坏。

**Example**
```cpp
struct B { int x; };
struct D : B { int y; };

void use(B*);

D a[] = { {1, 2}, {3, 4}, {5, 6} };
B* p = a;     // bad: a decays to &a[0] which is converted to a B*
p[1].x = 7;   // overwrite a[0].y

use(a);       // bad: a decays to &a[0] which is converted to a B*
```

**Enforcement**
- 所有数组退化操作以及基类到派生类的转换组合都要标记出。
- 将数组作为一个span而不是指针来传递，在进入span前，不要让数组名有派生类到基类的转换。

<br/>
### **C.153:优先使用virtual函数而不是cast**

**原因** 虚函数调用是安全的，而转换是容易出错的。虚函数调用可以达到最派生的函数，而转换可能会达到一个中间类，
因此会得到一个错误的结果（特别是在维护过程中对层次结构的修改）。

<br/>
## **C.over:重载运算符**
你可以重载普通函数、函数模板和运算符。你不能重载函数对象。

重载规则摘要：
- [C.160:定义运算符主要是为了模仿常规用法](#c160定义运算符主要是为了模仿常规用法)
- [C.161:对称运算符使用非成员函数](#c161对称运算符使用非成员函数)
- [C.162:重载操作大致等效](#c162重载操作大致等效)
- [C.163:仅对大致等效的操作进行重载](#c163仅对大致等效的操作进行重载)
- [C.164:避免隐式转换运算符](#c164避免隐式转换运算符)
- [C.165:在定制的点上使用using](#c165在定制的点上使用using)
- [C.166:只有作为智能指针和引用系统的一部分，才可以重载单操作符&](#c166只有作为智能指针和引用系统的一部分才可以重载单操作符)
- [C.167:使用运算符进行常规意义上的操作](#c167用运算符进行常规意义上的操作)
- [C.168:在其操作符的命名空间中定义重载操作符](#c168其操作符的命名空间中定义重载操作符)
- [C.170:如果你觉得要重载一个lambda，可以使用一个通用lambda](#c170如果你觉得要重载一个lambda可以使用一个通用lambda)

<br/>
### **C.160:定义运算符主要是为了模仿常规用法**

**原因** 减少意外情况。

**Example**
```cpp
class X {
public:
    // ...
    X& operator=(const X&); // member function defining assignment
    friend bool operator==(const X&, const X&); // == needs access to representation
                                                // after a = b we have a == b
    // ...
};
```

**Example,bad**
```cpp
X operator+(X a, X b) { return a.v - b.v; }   // bad: makes + subtract
```

**注意** 非成员运算符应该是friend的，或者与它们操作符定义在同个命名空间内。二元运算符应该等价地对待它们的操作符。

<br/>
### **C.161:对称运算符使用非成员函数**

**原因** 如果你使用一个成员函数，那么你需要两个。除非你使用一个非成员函数，例如==，否则a==b和b==a会有细微不同。

**Example**
```cpp
bool operator==(Point a, Point b) { return a.x == b.x && a.y == b.y; }
```

**Enforcement**
- 运算符成员函数都应该标记出。

<br/>
### **C.162:重载操作大致等效**

**原因** 对不同参数类型的逻辑等价操作使用不同的名称是混乱的，会导致在函数名称中编码类型信息，并抑制了通用编程。

**Example**
```cpp
void print(int a);
void print(int a, int base);
void print(const string&);
```
这三个函数都会打印它们的参数（适当地）。
```cpp
void print_int(int a);
void print_based(int a, int base);
void print_string(const string&);
```
相反，这三个函数也打印它们的参数（适当地）。在名称上添加只是引入了冗长，抑制了通用代码。

<br/>
### **C.163:仅对大致等效的操作进行重载**

**原因** 对逻辑上不同的函数使用相同的名称是混乱的，在使用通用编程时，会导致错误。

**Example**
```cpp
void open_gate(Gate& g);   // remove obstacle from garage exit lane
void fopen(const char* name, const char* mode);   // open file
```
这两种操作在本质上是不同的（而且不相关），所以它们的名称不同是好事。
相反：
```cpp
void open(Gate& g);   // remove obstacle from garage exit lane
void open(const char* name, const char* mode ="r");   // open file
```
这两种操作在本质上仍然是不同的（而且是不相关的），但名字已经一样，增加了混淆的概率。

**注意** open,move,+,==,常见和流行的名称要特别注意。

<br/>
### **C.164:避免隐式转换运算符**

**原因** 隐式转换可能是必不可少的（例如从double到int），但往往会造成意外（比如String到C风格的字符串）。

**Example**
```cpp
struct S1 {
    string s;
    // ...
    operator char*() { return s.data(); }  // BAD, likely to cause surprises
};

struct S2 {
    string s;
    // ...
    explicit operator char*() { return s.data(); }
};

void f(S1 s1, S2 s2)
{
    char* x1 = s1;     // OK, but can cause surprises in many contexts
    char* x2 = s2;     // error (and that's usually a good thing)
    char* x3 = static_cast<char*>(s2); // we can be explicit (on your head be it)
}
```
令人惊讶和具有潜在破坏性的隐性转换可能发生在任意难以发现的情况下，例如：
```cpp
S1 ff();

char* g()
{
    return ff();
}
```
ff()返回的字符串在返回的指针被使用之前被销毁。

**Enforcement** 非explicit的转换操作符都应该被标记出。

<br/>
### **C.165:在定制的点上使用using**

**原因** 要找到定义在独立命名空间的函数对象和函数，以 "定制 "一个普通的函数。

**Example** 比如swap。它是一个通用的（标准库）函数，其定义几乎对任何类型都有效。然而，为特定类型定义特定的swap()是可取的。
例如，一般的swap()会复制被交换的两个vector的元素，而一个好的实现则根本不会复制元素。
```cpp
namespace N {
    My_type X { /* ... */ };
    void swap(X&, X&);   // optimized swap for N::X
    // ...
}

void f1(N::X& a, N::X& b)
{
    std::swap(a, b);   // probably not what we wanted: calls std::swap()
}
```
f1()函数里的std::swap()正是我们要求它做的：它调用命名空间std中的swap()。不幸的是，这可能不是我们想要的结果。怎么用N::X中的swap？
```cpp
void f2(N::X& a, N::X& b)
{
    swap(a, b);   // calls N::swap
}
```
但这可能不是我们对generic代码的要求。在那里，我们通常想要特定的函数（如果它存在的话）和一般的函数（如果不存在的话）。这可以通过在函数的查找中包括一般函数来实现。
```cpp
void f3(N::X& a, N::X& b)
{
    using std::swap;   // make std::swap available
    swap(a, b);        // calls N::swap if it exists, otherwise std::swap
}
```

<br/>
### **C.166:只有作为智能指针和引用系统的一部分，才可以重载单操作符&**

**原因** 在C++中，&运算符是最基本的。C++语义的许多部分都假定它的默认含义。

**Example**
```cpp
class Ptr { // a somewhat smart pointer
    Ptr(X* pp) : p(pp) { /* check */ }
    X* operator->() { /* check */ return p; }
    X operator[](int i);
    X operator*();
private:
    T* p;
};

class X {
    Ptr operator&() { return Ptr{this}; }
    // ...
};
```

**注意** 如果确认要“乱用”运算符&，请确保其定义在结果类型上对->、[]、\*和.有匹配的含义。请注意，操作符.目前不能被重载，所以一个完美的系统是不可能的。
我们希望能够弥补这一点。[Operator Dot(R2)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4477.pdf)。
注意std::addressof()总是产生一个内置指针。

**Enforcement** 棘手的问题。如果&是用户定义的，而没有为结果类型定义->，则发出警告。

<br/>
### **C.167:用运算符进行常规意义上的操作**

**原因** 可读性。公约。可重用性。对通用代码的支持。

**Example**
```cpp
void cout_my_class(const My_class& c) // confusing, not conventional,not generic
{
    std::cout << /* class members here */;
}

std::ostream& operator<<(std::ostream& os, const my_class& c) // OK
{
    return os << /* class members here */;
}
```
就其本身而言，cout\_my\_class是可以的，但它不能与那些依赖<<惯例的输出的代码一起使用/兼容。
```cpp
My_class var { /* ... */ };
// ...
cout << "var = " << var << '\n';
```

**注意** 对于大多数运算符的含义，都有强大而有力的约定，例如:
- 比较（==, !=, <, <=, >, >=, and <=>）
- 数学运算（+, -, \*, /, and %）
- 访问运算符（., ->, unary \*, and []）
- 赋值（=）
不要对这些进行非常规的定义，不要为它们发明自己的名字。

<br/>
### **C.168:其操作符的命名空间中定义重载操作符**

**原因** 可读性。使用ADL查找运算符的能力。避免在不同的命名空间中出现不一致的定义。

**Example**
```cpp
struct S { };
S operator+(S, S);   // OK: in the same namespace as S, and even next to S
S s;

S r = s + s;
```

```cpp
namespace N {
    struct S { };
    S operator+(S, S);   // OK: in the same namespace as S, and even next to S
}

N::S s;

S r = s + s;  // finds N::operator+() by ADL
```

**Example,bad**
```cpp
struct S { };
S s;

namespace N {
    bool operator!(S a) { return true; }
    bool not_s = !s;
}

namespace M {
    bool operator!(S a) { return false; }
    bool not_s = !s;
}
```
在这里，!s的含义在N和M中是不同的，这可能是最令人困惑的。

**注意** 如果一个二元运算符被定义为两个不同命名空间的类型，你就不能遵循这个规则。比如说：
```cpp
Vec::Vector operator*(const Vec::Vector&, const Mat::Matrix&);
```
这可能是最好避免的事情。

这是另一条规则的特例[将辅助函数放在与它们支持的类相同的命名空间中](c5将辅助函数放在与它们支持的类相同的命名空间中)

**Enforcement**
- 不在运算符命名空间中的运算符定义要标记出。

<br/>
### **C.170:如果你觉得要重载一个lambda，可以使用一个通用lambda**

**原因** 你不能通过定义两个同名的不同lambdas来进行重载。

**Example**
```cpp
void f(int);
void f(double);
auto f = [](char);   // error: cannot overload variable and function

auto g = [](int) { /* ... */ };
auto g = [](double) { /* ... */ };   // error: cannot overload variables

auto h = [](auto) { /* ... */ };   // OK
```

**Enforcement** 编译器会捕捉到试图重载lambda的行为。

<br/>
## **C.union:Unions**

union是一个struct，其中所有成员都从同一个地址开始，所以它一次只能容纳一个成员。union并不跟踪哪个成员被存储，所以程序员必须把它弄清楚；
这本身就容易出错，但有一些方法可以弥补。

union再加上当前持有哪个成员的指示器的类型被称为tagged union, a discriminated union, or a variant。

union规则摘要：
- [C.180:用union来节省内存](#c180用union来节省内存)
- [C.181:避免使用原生union](#c181避免使用原生union)
- [C.182:使用匿名union来实现tagged union](#c182使用匿名union来实现tagged-union)
- [C.183:不要用union来做类型双关](#c183不要用union来做类型双关)

<br/>
### **C.180:用union来节省内存**

**原因** union允许一块内存在不同时间被用于不同类型的对象。因此，当我们有几个从不同时使用的对象时，可以用它来节省内存。

**Example**
```cpp
union Value {
    int x;
    double d;
};

Value v = { 123 };  // now v holds an int
cout << v.x << '\n';    // write 123
v.d = 987.654;  // now v holds a double
cout << v.d << '\n';    // write 987.654
```
但请注意警告。[C.181:避免使用原生union](#c181避免使用原生union)

**Example**
```cpp
// Short-string optimization

constexpr size_t buffer_size = 16; // Slightly larger than the size of a pointer

class Immutable_string {
public:
    Immutable_string(const char* str) :
        size(strlen(str))
    {
        if (size < buffer_size)
            strcpy_s(string_buffer, buffer_size, str);
        else {
            string_ptr = new char[size + 1];
            strcpy_s(string_ptr, size + 1, str);
        }
    }

    ~Immutable_string()
    {
        if (size >= buffer_size)
            delete[] string_ptr;
    }

    const char* get_str() const
    {
        return (size < buffer_size) ? string_buffer : string_ptr;
    }

private:
    // If the string is short enough, we store the string itself
    // instead of a pointer to the string.
    union {
        char* string_ptr;
        char string_buffer[buffer_size];
    };

    const size_t size;
};
```

<br/>
### **C.181:避免使用原生union**

**原因** 原生裸union是一个没有相关指示持有哪个成员的union，因此程序员必须保持追踪，但这也是造成错误来源。

**Example, bad**
```cpp
union Value {
    int x;
    double d;
};

Value v;
v.d = 987.654;  // v holds a double
```
到目前为止，情况还不错，但我们很容易滥用union：
```cpp
cout << v.x << '\n';    // BAD, undefined behavior: v holds a double, but we read it as an int
```
请注意，类型错误是在没有任何显式转换的情况下发生的。当我们测试该程序时，打印的最后一个值是1683627180，这是987.654的比特模式的整数值。
我们在这里遇到的是一个 "看不见的 "类型错误，它恰好给出了一个很容易看起来很无辜的结果。
而且，说到 "不可见"，这段代码没有产生任何输出。
```cpp
v.x = 123;
cout << v.d << '\n';    // BAD: undefined behavior
```

**替代方案** 将union与一个类型字段打包一起放在一个类中。

C++17的variant类型就是专用用于此：
```cpp
variant<int, double> v;
v = 123;        // v holds an int
int x = get<int>(v);
v = 123.456;    // v holds a double
w = get<double>(v);
```

<br/>
### **C.182:使用匿名union来实现tagged union**

**原因** 一个精心设计的tagged union是类型安全的。一个匿名的union简化了一个具有（tag,union）对的类的定义。

**Example** 这个例子主要是从TC++PL4第216-218页借用的。你可以去那里找解释。
这段代码有点复杂。处理一个具有用户定义的赋值和析构函数的类型是很棘手的。让程序员不必编写这样的代码是将variant纳入标准的一个原因。

```cpp
class Value { // two alternative representations represented as a union
private:
    enum class Tag { number, text };
    Tag type; // discriminant

    union { // representation (note: anonymous union)
        int i;
        string s; // string has default constructor, copy operations, and destructor
    };
public:
    struct Bad_entry { }; // used for exceptions

    ~Value();
    Value& operator=(const Value&);   // necessary because of the string variant
    Value(const Value&);
    // ...
    int number() const;
    string text() const;

    void set_number(int n);
    void set_text(const string&);
    // ...
};

int Value::number() const
{
    if (type != Tag::number) throw Bad_entry{};
    return i;
}

string Value::text() const
{
    if (type != Tag::text) throw Bad_entry{};
    return s;
}

void Value::set_number(int n)
{
    if (type == Tag::text) {
        s.~string();      // explicitly destroy string
        type = Tag::number;
    }
    i = n;
}

void Value::set_text(const string& ss)
{
    if (type == Tag::text)
        s = ss;
    else {
        new(&s) string{ss};   // placement new: explicitly construct string
        type = Tag::text;
    }
}

Value& Value::operator=(const Value& e)   // necessary because of the string variant
{
    if (type == Tag::text && e.type == Tag::text) {
        s = e.s;    // usual string assignment
        return *this;
    }

    if (type == Tag::text) s.~string(); // explicit destroy

    switch (e.type) {
    case Tag::number:
        i = e.i;
        break;
    case Tag::text:
        new(&s) string(e.s);   // placement new: explicit construct
    }

    type = e.type;
    return *this;
}

Value::~Value()
{
    if (type == Tag::text) s.~string(); // explicit destroy
}
```

<br/>
### **C.183:不要用union来做类型双关**

**原因** 读取一个union的类型与它被写入的类型不同，是未定义的行为。这样的双关是不可见的，或者至少比使用命名转换更难发现。
使用union的类型双关是一个错误的来源。

**Example, bad**
```cpp
union Pun {
    int x;
    unsigned char c[sizeof(int)];
};
```
Pun的想法是能够看清一个int的字符表示。
```cpp
void bad(Pun& u)
{
    u.x = 'x';
    cout << u.c[0] << '\n';     // undefined behavior
}
```
如果你想看一个int的字节，就用一个（命名的）cast。
```cpp
void if_you_must_pun(int& x)
{
    auto p = reinterpret_cast<std::byte*>(&x);
    cout << p[0] << '\n';     // OK; better
    // ...
}
```
访问从对象的声明类型到char\*、无符号char\*或std::byte\*的reinterpret\_cast的结果是定义的行为。
(不鼓励使用reinterpret\_cast，但至少我们可以看到一些棘手的事情正在发生)。

**注意** C++17引入了一个独特的类型std::byte，以方便对原始对象表示的操作。在这些操作中使用std::byte而不是unsigned char或char。

<br/>
# **E:枚举**

枚举被用来定义整数值的集合，以及为这些值的集合定义类型。有两种枚举，普通枚举和枚举class。

枚举规则摘要：
- [E.1:优先选择枚举而不是宏](#e1优先选择枚举而不是宏)
- [E.2:使用枚举来表示相关的常量的集合](#e2使用枚举来表示相关的常量的集合)
- [E.3:优先选择枚举class而不是普通枚举](#e3优先选择枚举class而不是普通枚举)
- [E.4:定义对枚举的操作，以便安全和简单地使用](#e4定义对枚举的操作以便安全和简单地使用)
- [E.5:枚举不要使用ALL\_CAPS](#e5枚举不要使用all_caps)
- [E.6:避免未命名的枚举](#e6避免未命名的枚举)
- [E.7:只有在必要时才指定一个枚举的基本类型](#e7只有在必要时才指定一个枚举的基本类型)
- [E.8:仅在必要时指定枚举的值](#e8仅在必要时指定枚举的值)

<br/>
### **E.1:优先选择枚举而不是宏**

**原因** 宏不遵守范围和类型规则。另外，宏的名字在预处理过程中被删除，所以通常不会出现在调试器等工具中。

**Example, bad**
```cpp
// webcolors.h (third party header)
#define RED   0xFF0000
#define GREEN 0x00FF00
#define BLUE  0x0000FF

// productinfo.h
// The following define product subtypes based on color
#define RED    0
#define PURPLE 1
#define BLUE   2

int webby = BLUE;   // webby == 2; probably not what was desired
```
使用枚举替换
```cpp
enum class Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
enum class Product_info { red = 0, purple = 1, blue = 2 };

int webby = blue;   // error: be specific
Web_color webby = Web_color::blue;
```
我们使用了一个enum class来避免名称冲突。

**Enforcement** 定义整数值的宏要标记出。

<br/>
### **E.2:使用枚举来表示相关的常量的集合**

**原因** 枚举显示了枚举者的关系，可以是一个命名的类型。

**Example**
```cpp
enum class Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
```

**注意** 在枚举上进行切换是很常见的，编译器可以对case标签的异常模式提出警告。比如：
```cpp
enum class Product_info { red = 0, purple = 1, blue = 2 };

void print(Product_info inf)
{
    switch (inf) {
    case Product_info::red: cout << "red"; break;
    case Product_info::purple: cout << "purple"; break;
    }
}
```
这种差一个switch语句通常是添加枚举器和不充分测试的结果。

**Enforcement**
- switch语句，其中的case涵盖了一个枚举的大多数但不是所有的枚举者要标记出。
- switch语句，其中case涵盖了一个枚举的几个枚举者，但没有default要标记出。

<br/>
### **E.3:优先选择枚举class而不是普通枚举**

**原因** 为了减少意外：传统的枚举转换为int太容易了。

**Example**
```cpp
void Print_color(int color);

enum Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
enum Product_info { red = 0, purple = 1, blue = 2 };

Web_color webby = Web_color::blue;

// Clearly at least one of these calls is buggy.
Print_color(webby);
Print_color(Product_info::blue);
```
使用enum class替换：
```cpp
void Print_color(int color);

enum class Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
enum class Product_info { red = 0, purple = 1, blue = 2 };

Web_color webby = Web_color::blue;
Print_color(webby);  // Error: cannot convert Web_color to int.
Print_color(Product_info::red);  // Error: cannot convert Product_info to int.
```

**Enforcement** 非枚举class的枚举标记出。
 
<br/>
### **E.4:定义对枚举的操作，以便安全和简单地使用**

**原因** 使用方便，避免错误。

**Example**
```cpp
enum Day { mon, tue, wed, thu, fri, sat, sun };

Day& operator++(Day& d)
{
    return d = (d == Day::sun) ? Day::mon : static_cast<Day>(static_cast<int>(d)+1);
}

Day today = Day::sat;
Day tomorrow = ++today;
```
使用static\_cast并不漂亮，但
```cpp
Day& operator++(Day& d)
{
    return d = (d == Day::sun) ? Day::mon : Day{++d};    // error
}
```
是一个无限递归的过程，在写它的时候，如果没有cast，在所有的情况下都使用switch，那就太啰嗦了。

<br/>
### **E.5:枚举不要使用ALL_CAPS**

**原因** 避免与宏发生冲突。

**Example, bad**
```cpp
 // webcolors.h (third party header)
#define RED   0xFF0000
#define GREEN 0x00FF00
#define BLUE  0x0000FF

// productinfo.h
// The following define product subtypes based on color

enum class Product_info { RED, PURPLE, BLUE };   // syntax error
```

<br/>
### **E.6:避免未命名的枚举**

**原因** 如果你不能命名一个枚举，那么这些值就没有关系。

**Example, bad**
```cpp
enum { red = 0xFF0000, scale = 4, is_signed = 1 };
```
这样的代码在有指定整数常量的便利替代方法之前编写的代码中并不少见。

**替换**
```cpp
constexpr int red = 0xFF0000;
constexpr short scale = 4;
constexpr bool is_signed = true;
```

**Enforcement** 标记未命名的枚举。

<br/>
### **E.7:只有在必要时才指定一个枚举的基本类型**

**原因** 默认的是最容易读和写的，int是默认的整数类型，int与C的枚举兼容。

**Example**
```cpp
enum class Direction : char { n, s, e, w,
                              ne, nw, se, sw };  // underlying type saves space

enum class Web_color : int32_t { red   = 0xFF0000,
                                 green = 0x00FF00,
                                 blue  = 0x0000FF };  // underlying type is redundant
```

**注意** 在枚举的前向声明中，指定基础类型是必要的。
```cpp
enum Flags : char;

void f(Flags);

// ....

enum Flags : char { /* ... */ };
```

<br/>
### **E.8:仅在必要时指定枚举的值**

**原因** 这是最简单的。它避免了重复的枚举器值。默认给出了一组连续的值，对switch语句的实现有好处。

**Example**
```cpp
enum class Col1 { red, yellow, blue };
enum class Col2 { red = 1, yellow = 2, blue = 2 }; // typo
enum class Month { jan = 1, feb, mar, apr, may, jun,
                   jul, august, sep, oct, nov, dec }; // starting with 1 is conventional
enum class Base_flag { dec = 1, oct = dec << 1, hex = dec << 2 }; // set of bits
```
指定数值是必要的，以便与传统的数值相匹配（例如，月），以及在不希望有连续数值的情况下（例如，像Base\_flag那样获得独立的bits）。

**Enforcement**
- 重复的枚举值要标记出
- 明确指定所有连续的枚举器值要标记出

<br/>
# **R:资源管理**

资源一般需要获取和释放，比如内存，文件句柄，套接字，锁等。为什么需要释放的原因是，资源是有限的。如果不释放甚至延迟释放都可能造成问题。  
本节目的：如何不造成内存泄漏，以及避免持有资源时间过长。
<br/>
- 资源管理规则摘要
	- [R.1:使用资源句柄和RAII来管理资源](#r1使用资源句柄和raii来管理资源)
	- [R.2:接口使用裸指针来代表单个对象](#r2接口使用裸指针来代表单个对象)
	- [R.3:T\*不拥有所有权](#r3t不拥有所有权)
	- [R.4:T&不拥有所有权](#r4t不拥有所有权)
	- [R.5:优先使用scoped对象，非必要不使用堆分配](#r5优先使用scoped对象非必要不使用堆分配)
	- [R.6:避免使用非const全局变量](#r6避免使用非const全局变量)
- 分配释放规则摘要
	- [R.10:避免使用malloc()和free()](#r10避免使用malloc和free)
	- [R.11:避免显式调用new和delete](#r11避免显式调用new和delete)
	- [R.12:资源的分配应立即丢给一个可管理对象](#r12资源的分配应立即丢给一个可管理对象)
	- [R.13:在一个表达式中最多执行一次显式资源分配](#r13在一个表达式中最多执行一次显式资源分配)
	- [R.14:避免使用[]，用span](#r14避免使用用span)
	- [R.15:重载allocation/deallocation要配套](#r15重载allocationdeallocation要配套)
- 智能指针规则摘要
	- [R.20:用unique\_ptr和shared\_ptr来表达所有权](#r20用unique_ptr和shared_ptr来表达所有权)
	- [R.21:优先使用unique\_ptr，除非要共享所有权才用shared\_ptr](#r21优先使用unique_ptr除非要共享所有权才用shared_ptr)
	- [R.22:使用make\_shared来获取shared\_ptr](#r22使用make_shared来获取shared_ptr)
	- [R.23:使用make\_unique来获取unique\_ptr](#r23使用make_unique来获取unique_ptr)
	- [R.24:使用weak\_ptr来避免shared\_ptr循环引用](#r24使用weak_ptr来避免shared_ptr循环引用)
	- [R.30:原则上用T\*或者T&，而不是用smart pointer来传递参数](#r30只有在需要表达生命期语意前提下才用智能指针传递参数)
	- [R.32:使用unique\_ptr\<Widget\>来表达一个函数对Widget的所有权](#r32使用unique_ptrwidget来表达一个函数对widget的所有权)
	- [R.33:使用unique\_ptr\<Widget\>&参数来表达函数reseats Widget](#r33使用unique_ptrwidget参数来表达函数reseats-widget)
	- [R.34:使用shared\_ptr\<T\>参数来表达共享所有权](#r34使用shared_ptrt参数来表达共享所有权)
	- [R.35:shared\_ptr\<T\>&参数用来表达可能reseat这个shared\_ptr](#r35shared_ptrt参数用来表达可能reseat这个shared_ptr)
	- [R.36:传递const shared\_ptr\<T\>&来表示有可能保留对象引用计数](#r36传递const-shared_ptrt来表示有可能保留对象引用计数)
	- [R.37:不要传参一个来自aliased的smart pointer的指针或引用](#r37不要传参来自aliased的smart-pointer的指针或引用)

<br/>
### **R.1:使用资源句柄和RAII来管理资源**

**原因** 为了避免泄漏以及简化管理资源，C++语言层面的构造、析构函数刚好对应资源获取、释放，比如fopen/fclose，lock/unlock，new/delete。\
每当你要处理资源，比如获取、释放资源的函数调用时，应该把它封装到一个对象中，构造函数获取资源，析构函数释放资源。

**Example, bad**
```cpp
void send(X* x, string_view destination)
{
    auto port = open_port(destination);
    my_mutex.lock();
    // ...
    send(port, x);
    // ...
    my_mutex.unlock();
    close_port(port);
    delete x;
}
```
在以上代码中，你必须记得要unlock，close\_port，delete，不管任何一条路径都要处理到，而且，一旦在 ... 中抛出异常，x就会泄漏，my\_mutex就还会锁住。

**Example**
```cpp
void send(unique_ptr<X> x, string_view destination)  // x owns the X
{
    Port port{destination};            // port owns the PortHandle
    lock_guard<mutex> guard{my_mutex}; // guard owns the lock
    // ...
    send(port, x);
    // ...
} // automatically unlocks my_mutex and deletes the pointer in x
```
现在所有的资源都自动清理，所有的路径上都执行一次，无法是否有异常，不仅如此，函数签名也表示它接管了指针所有权。
那Port是如何封装的？
```cpp
class Port {
    PortHandle port;
public:
    Port(string_view destination) : port{open_port(destination)} { }
    ~Port() { close_port(port); }
    operator PortHandle() { return port; }

    // port handles can't usually be cloned, so disable copying and assignment if necessary
    Port(const Port&) = delete;
    Port& operator=(const Port&) = delete;
};
```

<br/>
### **R.2:接口使用裸指针来代表单个对象**

**原因** 数组最好由一个容器来表示，vector表示拥有所有权，span表示非所有权，它们拥有足够的信息来做范围检查。

**Example, bad**
```cpp
void f(int* p, int n)   // n is the number of elements in p[]
{
    // ...
    p[2] = 7;   // bad: subscript raw pointer
    // ...
}
```
编译器不会读注释，如果没有其他代码，那就不知道p到底拥有几个元素。建议用span替换。

**Example**
```cpp
void g(int* p, int fmt)   // print *p using format #fmt
{
    // ... uses *p and p[0] only ...
}
```

例外：C-style一般传递以0结尾的字符串指针，使用zstring而不是char\*，来表示依赖这个约定。
注意：当前很多用指向单元素的指针都可以用引用，但nullptr可以表示一个合法的指针，引用不行。

**Enforcement** 

* 所有非container,view,iterator的指针计算（包括++），都应标记指出，但是这对老代码会产生巨多false positives。
* 所有数组名被当作单一指针传递都应该被标记。

<br/>
### **R.3:T\*不拥有所有权**

**原因** 如果是拥有所有权的指针都会被标识出来，这样就可以可靠且有效的删除所指向对象。

**Example**
```cpp
void f()
{
    int* p1 = new int{7};           // bad: raw owning pointer
    auto p2 = make_unique<int>(7);  // OK: the int is owned by a unique pointer
    // ...
}
```
unique\_ptr保证资源会析构，不会泄漏，即使抛出异常。T\*不会。

**Example**
```cpp
template<typename T>
class X {
public:
    T* p;   // bad: it is unclear whether p is owning or not
    T* q;   // bad: it is unclear whether q is owning or not
    // ...
};
```
优化
```cpp
template<typename T>
class X2 {
public:
    owner<T*> p;  // OK: p is owning
    T* q;         // OK: q is not owning
    // ...
};
```

**Example, bad**
```cpp
Gadget* make_gadget(int n)
{
    auto p = new Gadget{n};
    // ...
    return p;
}

void caller(int n)
{
    auto p = make_gadget(n);   // remember to delete p
    // ...
    delete p;
}
```
返回一个指针意味着调用者不确定这个指针的生命期，谁来负责删除所指向对象呢？
如果Gadget move函数开销不大，比如该对象很小，或者移动该对象很高效，那么直接返回value
就可以。
```cpp
Gadget make_gadget(int n)
{
    Gadget g{n};
    // ...
    return g;
}
```
注意：本规则适用于factory模式。如果确实需要返回指针，比如指向子类的基类指针，那么应该返回smart pointer。

**Enforcement**

- delete一个非所有权指针要给出警告
- 在每条代码路径上都没能reset或者显式delete都应该给出警告
- new出来的对象被一个裸指针指向给出警告
- 函数内构造的对象被当作函数返回值，并且该对象有移动构造，那么应该建议直接返回value。

<br/>
### **R.4:T&不拥有所有权**

**Example**
```cpp
void f()
{
    int& r = *new int{7};  // bad: raw owning reference
    // ...
    delete &r;             // bad: violated the rule against deleting raw pointers
}
```

<br/>
### **R.5:优先使用scoped对象，非必要不使用堆分配**

**原因** 一个scoped对象指的是一个局部、全局对象或者成员，这意味着不用再有额外的allocation和deallocation，scoped对象的成员也是个scoped对象，scoped对象的生命期管理由构造、析构函数管理。

**Example**
以下代码不够高效，因为由非必要的分配和释放内存操作，而且一旦在...部分抛出异常，会造成内存泄漏，而且代码也不简洁。
```cpp
void f(int n)
{
    auto p = new Gadget{n};
    // ...
    delete p;
}
```
Instead
```cpp
void f(int n)
{
    Gadget g{n};
    // ...
}
```

**Enforcement**

- 在函数内，如果对象被分配，然后在各个代码路径上被释放，那么首选stack object。
- 如果unique_ptr对象没有被移动，shared_ptr没有被拷贝，reassigned or reset，那么应当发出警告。

<br/>
### **R.6:避免使用非const全局变量**

**Example**
```cpp
struct Data {
    // ... lots of stuff ...
} data;            // non-const data

void compute()     // don't
{
    // ... use data ...
}

void output()     // don't
{
    // ... use data ...
}
```
还有谁可以修改data？

一个线程读取全局数据，另一个线程修改这个全局数据，就会data race。


<br/>
### **R.10:避免使用malloc()和free()**

**原因** malloc()和free()不支持构造和析构，跟new和delete也不能混着用。

**Example**
```cpp
class Record {
    int id;
    string name;
    // ...
};

void use()
{
    // p1 might be nullptr
    // *p1 is not initialized; in particular,
    // that string isn't a string, but a string-sized bag of bits
    Record* p1 = static_cast<Record*>(malloc(sizeof(Record)));

    auto p2 = new Record;

    // unless an exception is thrown, *p2 is default initialized
    auto p3 = new(nothrow) Record;
    // p3 might be nullptr; if not, *p3 is default initialized

    // ...

    delete p1;    // error: cannot delete object allocated by malloc()
    free(p2);    // error: cannot free() object allocated by new
}
```

<br/>
### **R.11:避免显式调用new和delete**

<br/>
### **R.12:资源的分配应立即丢给一个可管理对象**

**原因** 不这样做，一个异常或返回都会导致泄漏。

**Example,bad**
```cpp
void func(const string& name)
{
    FILE* f = fopen(name, "r");            // open the file
    vector<char> buf(1024);
    auto _ = finally([f] { fclose(f); });  // remember to close the file
    // ...
}
```
**Example**
```cpp
void func(const string& name)
{
    ifstream f{name};   // open the file
    vector<char> buf(1024);
    // ...
}
```

<br/>
### **R.13:在一个表达式中最多执行一次显式资源分配**

**原因** 如果在一个表达式中有两次以上资源分配，因为子表达式的计算顺序比如函数参数，会导致内存泄漏。

**Example**
```cpp
void fun(shared_ptr<Widget> sp1, shared_ptr<Widget> sp2);
```
```cpp
// BAD: potential leak
fun(shared_ptr<Widget>(new Widget(a, b)), shared_ptr<Widget>(new Widget(c, d)));
```
编译器会重组这两个表达式，它不是异常安全的，有可能编译器先分配两段内存，然后再执行Widget构造函数，
如果其中一个构造函数抛出异常，那么就会内存泄漏。
一个简化的解决方案就是不要在一个表达式中有超过一次显式的资源分配。
```cpp
shared_ptr<Widget> sp1(new Widget(a, b)); // Better, but messy
fun(sp1, new Widget(c, d));
```
```cpp
fun(make_shared<Widget>(a, b), make_shared<Widget>(c, d)); // Best
```

<br/>
### **R.14:避免使用[]，用span**

**原因** 数组会退化成指针，因此会丢失其size，造成range错误，使用span，它包含size信息。

**Example**
```cpp
void f(int[]);          // not recommended

void f(int*);           // not recommended for multiple objects
                        // (a pointer should point to a single object, do not subscript)

void f(gsl::span<int>); // good, recommended
```

<br/>
### **R.15:重载allocation/deallocation要配套**

**Example**
```cpp
class X {
    // ...
    void* operator new(size_t s);
    void operator delete(void*);
    // ...
};
```
注意：如果希望内存不被释放，释放操作应该=delete。

<br/>
### **R.20:用unique\_ptr和shared\_ptr来表达所有权**

**原因** 避免资源泄漏

**Example**
```cpp
void f()
{
    X x;
    X* p1 { new X };              // see also ???
    unique_ptr<X> p2 { new X };   // unique ownership; see also ???
    shared_ptr<X> p3 { new X };   // shared ownership; see also ???
    auto p4 = make_unique<X>();   // unique_ownership, preferable to the explicit use "new"
    auto p5 = make_shared<X>();   // shared ownership, preferable to the explicit use "new"
}
```

<br/>
### **R.21:优先使用unique\_ptr，除非要共享所有权才用shared\_ptr**

**Example,bad**
```cpp
void f()
{
    shared_ptr<Base> base = make_shared<Derived>();
    // use base locally, without copying it -- refcount never exceeds 1
} // destroy base
```
**Example**
```cpp
void f()
{
    unique_ptr<Base> base = make_unique<Derived>();
    // use base locally
} // destroy base
```

**Enforcement**

- 如果函数内使用shared\_ptr分配对象，但函数没有返回shared\_ptr，而且没有把shared\_ptr传递给其他函数，那么建议用unique\_ptr替换。


<br/>
### **R.22:使用make\_shared来获取shared\_ptr**

**Example**
```cpp
shared_ptr<X> p1 { new X{2} }; // bad
auto p = make_shared<X>(2);    // good
```

make\_shared更简洁，同时避免了对引用计数的单独allocation，它直接把引用计数放在其对象旁边。


<br/>
### **R.23:使用make\_unique来获取unique\_ptr**

**Example**
```cpp
unique_ptr<Foo> p {new Foo{7}};    // OK: but repetitive

auto q = make_unique<Foo>(7);      // Better: no repetition of Foo
```

简洁，而且在复杂的表达式中更能保证异常安全。


<br/>
### **R.24:使用weak\_ptr来避免shared\_ptr循环引用**

**原因** shared\_ptr依赖引用计数，循环依赖的引用计数永远不会清零。

**Example**
```cpp
#include <memory>

class bar;

class foo {
public:
  explicit foo(const std::shared_ptr<bar>& forward_reference)
    : forward_reference_(forward_reference)
  { }
private:
  std::shared_ptr<bar> forward_reference_;
};

class bar {
public:
  explicit bar(const std::weak_ptr<foo>& back_reference)
    : back_reference_(back_reference)
  { }
  void do_something()
  {
    if (auto shared_back_reference = back_reference_.lock()) {
      // Use *shared_back_reference
    }
  }
private:
  std::weak_ptr<foo> back_reference_;
};
```

<br/>
### **R.30:只有在需要表达生命期语意前提下，才用智能指针传递参数**

见[F.7](#f7一般情况下传参用裸指针或引用而不是智能指针)

**原因** 如果有表达所有权的语意，比如传递或者共享所有权，才使用smart pointer来传递参数。不涉及到生命期修改，应当用裸指针或引用传参。

传递smart pointer限制了函数调用方，一个函数如果需要一个widgt，就应当接受任何widget对象，而不是生命期要智能管理的对象。

传递一个shared\_ptr意味着一个runtime开销。

**Example**
```cpp
// accepts any int*
void f(int*);

// can only accept ints for which you want to transfer ownership
void g(unique_ptr<int>);

// can only accept ints for which you are willing to share ownership
void g(shared_ptr<int>);

// doesn't change ownership, but requires a particular ownership of the caller
void h(const unique_ptr<int>&);

// accepts any int
void h(int&);
```

**Example, bad**
```
// callee
void f(shared_ptr<widget>& w)
{
    // ...
    use(*w); // only use of w -- the lifetime is not used at all
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(my_widget);

widget stack_widget;
f(stack_widget); // error
```

**Example, good**
```cpp
// callee
void f(widget& w)
{
    // ...
    use(w);
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(*my_widget);

widget stack_widget;
f(stack_widget); // ok -- now this works
```

函数参数自然是在函数调用生命周期内，因此极少有生命周期问题。而且dangling指针可以在静态检查检出。

**Enforement**

- 如果一个函数需要一个智能指针(重载->或\*)参数，该智能指针是可拷贝的，但函数只调用(\*,->或get())其中之一，那建议只传递T*或者T&。
- 传递一个智能指针(重载->或\*)当参数，该智能指针是copyable/movable，但是在函数调用中没有用到拷贝/移动，而且也不修改智能指针所指向数据，也不会被传递给另一个可以这样做的函数。这意味着所有权语意没有用到，那么应当被标记改为传递T*或者T&。


<br/>
### **R.32:使用unique_ptr\<Widget\>来表达一个函数对Widget的所有权**

**原因** 以这种方式使用unique\_ptr，既记录又执行了函数调用的所有权转移。 

**Example**
```cpp
void sink(unique_ptr<widget>); // takes ownership of the widget

void uses(widget*);            // just uses the widget
```

**Example, bad**
```cpp
void thinko(const unique_ptr<widget>&); // usually not what you want
```

**Enforcement**

- 如果一个函数传入lvalue左值unique_ptr，但是没有赋值或reset()，那么应该考虑换成T*或T&。
- 如果一个函数传入const unique_ptr<T>&，那应该考虑换成const T*或const T&。

<br/>
### **R.33:使用unique_ptr\<Widget\>&参数来表达函数reseats Widget**

**原因** 使用这种方式，即记录又执行了函数调用的reseat作用。

**Example**
```cpp
void reseat(unique_ptr<widget>&); // "will" or "might" reseat pointer
```

**Example, bad**
```cpp
void thinko(const unique_ptr<widget>&); // usually not what you want
```

**Enforcement**

- 如果传入左值引用unique_ptr<T>当函数参数，但是函数内没有调用赋值或reset，建议用T*或者T&替换。
- 如果一个函数传入const unique_ptr<T>&，建议用const T*或者const T&替换。

<br/>
### __R.34:使用shared_ptr\<T\>参数来表达共享所有权__

**原因** 显式表达共享所有权。

**Example, Good**
```cpp
class WidgetUser
{
public:
    // WidgetUser will share ownership of the widget
    explicit WidgetUser(std::shared_ptr<widget> w) noexcept:
        m_widget{std::move(w)} {}
    // ...
private:
    std::shared_ptr<widget> m_widget;
};
```

**Enforcement**

- 如果函数传递左值shared_ptr<T>参数，但是调用函数内没有对它赋值或reset()，建议用T*或T&替换。
- 如果函数传递const shared_ptr<T>或const shared_ptr<T>&，但是没有任何拷贝或者移动给另外一个shared_ptr，建议用T*或T&替换。
- 如果函数传递shared_ptr<T>右值，那么建议直接传值T。

<br/>
### **R.35:shared_ptr\<T\>&参数用来表达可能reseat这个shared_ptr**

**原因** 显式表达函数可能reseat，reseat表示一个引用或一个智能指针指向一个不同的对象。

**Example, good**
```cpp
void ChangeWidget(std::shared_ptr<widget>& w)
{
    // This will change the callers widget
    w = std::make_shared<widget>(widget{});
}
```

**Enforcement**

- 如果一个函数传入左值引用shared\_ptr<T>，但是函数内没有赋值或reset()，建议用T*或T&替换。
- 如果一个函数传入const shared_ptr<T>，或const shared_ptr<T>&，但是没有任何拷贝或者移动给另外一个shared_ptr<T>，建议用T*或T&替换。
- 如果传入一个右值引用shared_ptr<T>，那么建议直接传递T。

<br/>
### **R.36:传递const shared_ptr\<T\>&来表示有可能保留对象引用计数**


**Example, good**
```cpp
void share(shared_ptr<widget>);            // share -- "will" retain refcount

void reseat(shared_ptr<widget>&);          // "might" reseat ptr

void may_share(const shared_ptr<widget>&); // "might" retain refcount
```

<br/>
### **R.37:不要传参来自aliased的smart pointer的指针或引用**

**原因** 造成引用计数丢失或者悬空指针最大的一个原因就是违反本条规则，函数调用传入指针或者引用应该在同个调用链上，\
在调用链的顶端应该从一个智能指针获得裸指针或引用，并保持对象存在，保证智能指针在整个调用链上没有reset或者被重新赋值。

注意：有时候会先本地获取一个智能指针的拷贝，然后保持对象存活，直到整个调用链。

**Example**
```cpp
// global (static or heap), or aliased local ...
shared_ptr<widget> g_p = ...;

void f(widget& w)
{
    g();
    use(w);  // A
}

void g()
{
    g_p = ...; // oops, if this was the last shared_ptr to that widget, destroys the widget
}
```
以下调用过不了code review
```cpp
void my_code()
{
    // BAD: passing pointer or reference obtained from a non-local smart pointer
    //      that could be inadvertently reset somewhere inside f or its callees
    f(*g_p);

    // BAD: same reason, just passing it as a "this" pointer
    g_p->func();
}
```
修复以上bug，可以先在开头获得一个本地拷贝，这样可以保持引用计数。
```cpp
void my_code()
{
    // cheap: 1 increment covers this entire function and all the call trees below us
    auto pin = g_p;

    // GOOD: passing pointer or reference obtained from a local unaliased smart pointer
    f(*pin);

    // GOOD: same reason
    pin->func();
}
```

**Enforcement**
- 如果从一个不是本地的或者是有别名的智能指针变量，获取的指针或引用，在函数中被使用，那应该发出警告。
- 如果这个智能指针是shared\_ptr，那应该先获取该shared\_ptr的本地拷贝，然后再获取它的指针或引用。

<br/>
# **E:错误处理**

错误处理涉及：
- 检测一个错误
- 将有关错误的信息传递给一些处理程序代码
- 保存程序的有效状态
- 避免资源泄漏

不可能从所有的错误中恢复。如果不可能从错误中恢复，重要的是以一种明确的方式迅速"脱身"。错误处理的策略必须是简单的，
否则就会成为更糟糕的错误的来源。未经测试和很少执行的错误处理代码本身就是许多错误的来源。

这些规则旨在帮助避免几种类型的错误：

- 违反类型（e.g., misuse of unions and casts）
- 资源泄漏（包括内存泄漏）
- 界限错误
- 生命期错误（例如，在一个对象被删除后再去访问它）
- 复杂性错误（由于思想表达过于复杂而可能出现的逻辑错误）
- 接口错误（例如，通过一个接口传递了一个意外的值）

错误处理摘要：
- [E.1:在设计的早期制定错误处理策略](#e1在设计的早期制定错误处理策略)
- [E.2:抛出一个异常来表明一个函数不能执行其指定的任务](#e2抛出一个异常来表明一个函数不能执行其指定的任务)
- [E.3:仅将异常用于错误处理](#e3仅将异常用于错误处理)
- [E.4:围绕不变量设计你的错误处理策略](#e4围绕不变量设计你的错误处理策略)
- [E.5:让构造函数建立一个不变量，如果它不能建立，则抛出](#e5让构造函数建立一个不变量如果它不能建立则抛出)
- [E.6:使用RAII来防止泄漏](#e6使用raii来防止泄漏)
- [E.7:陈述你的先决条件](#e7陈述你的先决条件)
- [E.8:陈述你的后置条件](#e8陈述你的后置条件)
- [E.12:当因为throw而退出一个函数是不可能的或不可接受的时候，使用noexcept](#e12当因为throw而退出一个函数是不可能的或不可接受的时候使用noexcept)
- [E.13:正在作为对象的直接所有者永远不要抛出](#e13正在作为对象的直接所有者永远不要抛出)
- [E.14:使用专门设计的用户定义类型作为例外（非内置类型）](#e14使用专门设计的用户定义类型作为例外非内置类型)
- [E.15:通过值抛出异常，通过引用从一个层次结构中捕捉异常](#e15通过值抛出异常通过引用从一个层次结构中捕捉异常)
- [E.16:析构函数、释放、swap和异常类型的复制/移动构造绝不能失败](#e16析构函数释放swap和异常类型的复制移动构造绝不能失败)
- [E.17:不要试图在每个函数中捕捉每个异常](#e17不要试图在每个函数中捕捉每个异常)
- [E.18:尽量减少对显式try/catch的使用](#e18尽量减少对显式trycatch的使用)
- [E.19:如果没有合适的资源句柄，使用final\_action对象来表达清理工作](#e19如果没有合适的资源句柄使用final_action对象来表达清理工作)
- [E.25:如果你不能抛出异常，模拟RAII进行资源管理](#e25如果你不能抛出异常模拟raii进行资源管理)
- [E.26:如果你不能抛出异常，考虑快速失败](#e26如果你不能抛出异常考虑快速失败)
- [E.27:如果你不能抛出异常，请系统地使用错误代码](#e27如果你不能抛出异常请系统地使用错误代码)
- [E.28:避免基于全局状态的错误处理（如errno）](#e28避免基于全局状态的错误处理如errno)
- [E.30:不要使用异常规范](#e30不要使用异常规范)
- [E.31:正确排序你的catch子句](#e31正确排序你的catch子句)

<br/>
### **E.1:在设计的早期制定错误处理策略**

**原因** 处理错误和资源泄漏的一致且完整的策略很难改造到系统中。 

<br/>
### **E.2:抛出一个异常来表明一个函数不能执行其指定的任务**

**原因** 使错误处理系统化、稳健化和非重复性。

**Example**
```cpp
struct Foo {
    vector<Thing> v;
    File_handle f;
    string s;
};

void use()
{
    Foo bar { {Thing{1}, Thing{2}, Thing{monkey} }, {"my_file", "r"}, "Here we go!"};
    // ...
}
```
在这里，vector和string构造函数可能无法为其元素分配足够的内存，vector构造函数可能无法复制其初始化列表中的事物，并且File\_handle可能无法打开所需的文件。
在每种情况下，它们都会抛出一个异常供use()的调用者处理。如果use()可以处理构造bar的失败，它可以使用try/catch来控制。
在任何一种情况下，Foo的构造函数都会在将控制权传递给任何试图创建Foo的对象之前正确地销毁构造的成员。
请注意，没有可能包含错误代码的返回值。 

File\_handle构造函数可以这样定义：
```cpp
File_handle::File_handle(const string& name, const string& mode)
    : f{fopen(name.c_str(), mode.c_str())}
{
    if (!f)
        throw runtime_error{"File_handle: could not open " + name + " as " + mode};
}
```

**注意** 人们常说，异常是为了表示特殊事件和失败。然而，这有点绕圈子，因为"什么是例外？"。
- 无法满足的先决条件
- 无法构造对象的构造函数（未能建立其类的不变量）
- 超出范围的错误（例如，v[v.size()] = 7）
- 无法获取资源（例如，网络已关闭）
相比之下，普通循环的终止并不特殊。除非循环的目的是无限的，否则终止是正常的，是可以预期的。

**注意** 不要把抛出作为从函数中返回数值的简单替代方式。

**例外** 某些系统，例如硬实时系统，需要保证在执行开始之前已知的（通常很短的）恒定最长时间内采取操作。
仅当有工具支持准确预测从抛出中恢复的最长时间时，此类系统才能使用异常。

**注意** 在决定您负担不起或不喜欢基于异常的错误处理之前，先看看替代方案；他们有自己的复杂性和问题。此外，尽可能在提出效率要求之前进行测量。

<br/>
### **E.3:仅将异常用于错误处理**

**原因** 为了使错误处理与"普通代码"分开。C++的实现往往是基于异常很少的假设而进行优化的。

**Example, don’t**
```cpp
// don't: exception not used for error handling
int find_index(vector<string>& vec, const string& x)
{
    try {
        for (gsl::index i = 0; i < vec.size(); ++i)
            if (vec[i] == x) throw i;  // found x
    }
    catch (int i) {
        return i;
    }
    return -1;   // not found
}
```
这样做比较复杂，而且很可能比明显的替代方案运行得更慢。在一个vector中寻找一个值并没有什么特别之处。

<br/>
### **E.4:围绕不变量设计你的错误处理策略**

**原因** 要使用一个对象，它必须处于有效状态（由不变量正式或非正式定义），并且要从错误中恢复，每个未销毁的对象都必须处于有效状态。

**注意** 不变量是一个对象成员的逻辑条件，构造函数必须为公共成员函数建立这样的条件。

<br/>
### **E.5:让构造函数建立一个不变量，如果它不能建立，则抛出**

**原因** 在不建立不变性的情况下留下一个对象是自找麻烦。并不是所有的成员函数都可以被调用。

**Example**
```cpp
class Vector {  // very simplified vector of doubles
    // if elem != nullptr then elem points to sz doubles
public:
    Vector() : elem{nullptr}, sz{0}{}
    Vector(int s) : elem{new double[s]}, sz{s} { /* initialize elements */ }
    ~Vector() { delete [] elem; }
    double& operator[](int s) { return elem[s]; }
    // ...
private:
    owner<double*> elem;
    int sz;
};
```
类的不变性--在这里作为注释--是由构造函数建立的。如果new不能分配到所需的内存，它就会抛出。操作符，特别是下标操作符，依赖于该不变式。

**Enforcement** 标记没有构造函数的具有private状态的类（public, protected, or private）

<br/>
### **E.6:使用RAII来防止泄漏**

**原因** 泄漏通常是不可接受的。手动释放资源容易出错。RAII（“Resource Acquisition Is Initialization”）是最简单、最系统的防止泄漏的方法。

**Example**
```cpp
void f1(int i)   // Bad: possible leak
{
    int* p = new int[12];
    // ...
    if (i < 17) throw Bad{"in f()", i};
    // ...
}
```
我们可以在throw前谨慎地释放资源。
```cpp
void f2(int i)   // Clumsy and error-prone: explicit release
{
    int* p = new int[12];
    // ...
    if (i < 17) {
        delete[] p;
        throw Bad{"in f()", i};
    }
    // ...
}
```
这是冗长的。在具有多个可能throw的较大代码中，显式释放变得重复且容易出错。
```cpp
void f3(int i)   // OK: resource management done by a handle (but see below)
{
    auto p = make_unique<int[]>(12);
    // ...
    if (i < 17) throw Bad{"in f()", i};
    // ...
}
```
请注意，即使throw是隐式的，这也有效，因为它发生在被调用的函数中：
```cpp
void f4(int i)   // OK: resource management done by a handle (but see below)
{
    auto p = make_unique<int[]>(12);
    // ...
    helper(i);   // might throw
    // ...
}
```
除非你真的需要指针语义，否则使用局部资源对象。
```cpp
void f5(int i)   // OK: resource management done by local object
{
    vector<int> v(12);
    // ...
    helper(i);   // might throw
    // ...
}
```
这甚至更简单、更安全，而且往往更有效率。


**注意** 如果没有明显的资源句柄并且由于某种原因无法定义适当的RAII对象/句柄，作为最后的手段，清理操作可以由final\_action对象表示。

**注意** 但是，如果我们正在编写一个不能使用异常的程序，我们该怎么办？首先挑战这个假设；周围有许多反异常神话。我们只知道几个很好的理由：
- 我们在一个如此小的系统上，异常支持会耗尽我们的大部分2K内存。
- 我们处于硬实时系统中，我们没有工具可以保证在规定时间内处理异常。
- 我们所处的系统中有大量遗留代码，以难以理解的方式使用大量指针（尤其是没有可识别的所有权策略），因此异常可能会导致泄漏。
- 我们对 C++ 异常机制的实现非常糟糕（缓慢、消耗内存、无法为动态链接库正常工作等）。向您的实施供应商投诉；如果没有用户抱怨，就不会发生改进。
- 如果我们挑战经理的古老智慧，我们就会被解雇。

这些原因中只有第一个是根本原因，因此只要有可能，就使用异常来实现RAII，或者将RAII对象设计为永不失败。当不能使用异常时，模拟RAII。
即在构造后系统地检查对象是否有效，并在析构函数中仍然释放所有资源。一种策略是向每个资源句柄添加一个valid() 操作：
```cpp
void f()
{
    vector<string> vs(100);   // not std::vector: valid() added
    if (!vs.valid()) {
        // handle error or exit
    }

    ifstream fs("foo");   // not std::ifstream: valid() added
    if (!fs.valid()) {
        // handle error or exit
    }

    // ...
} // destructors clean up as usual
```
显然，这会增加代码的大小，不允许隐式传播“异常”（valid() 检查），并且可以忘记valid() 检查。优先使用异常。
<br/>
### **E.7:陈述你的先决条件**

**原因** 为了避免接口错误。

另见[I.5:说明先决条件（如果有的话）](#i5说明先决条件如果有的话)

<br/>
### **E.8:陈述你的后置条件**

**原因** 为了避免接口错误。

另见[I.7:说明后置条件](#i7说明后置条件)

<br/>
### **E.12:当因为throw而退出一个函数是不可能的或不可接受的时候，使用noexcept**

**原因** 使错误处理系统化、稳健、高效。

**Example**
```cpp
double compute(double d) noexcept
{
    return log(sqrt(d <= 0 ? 1 : d));
}
```
在这里，我们知道compute不会抛出，因为它是由不会抛出的操作组成的。通过声明compute为noexcept，我们给了编译器和人类读者一些信息，使他们更容易理解和操作compute。

**注意** 许多标准库函数是noexcept的，包括所有从C标准库 "继承 "下来的标准库函数。

**Example**
```cpp
vector<double> munge(const vector<double>& v) noexcept
{
    vector<double> v2(v.size());
    // ... do something ...
}
```
这里的noexcept表示我不愿意或无法处理无法构造局部vector的情况。也就是说，我认为内存耗尽是一个严重的设计错误（与硬件故障相当），所以如果它发生，我愿意让程序崩溃。
 
<br/>
### **E.13:正在作为对象的直接所有者永远不要抛出**

**原因** 这将是一个漏洞。

**Example**
```cpp
void leak(int x)   // don't: might leak
{
    auto p = new int{7};
    if (x < 0) throw Get_me_out_of_here{};  // might leak *p
    // ...
    delete p;   // we might never get here
}
```
避免这类问题的一个方法是一致地使用资源柄。
```cpp
void no_leak(int x)
{
    auto p = make_unique<int>(7);
    if (x < 0) throw Get_me_out_of_here{};  // will delete *p if necessary
    // ...
    // no need for delete p
}
```
另一个解决方案（通常更好）是使用局部变量来消除指针的明确使用。
```cpp
void no_leak_simplified(int x)
{
    vector<int> v(7);
    // ...
}
```

**注意** 如果您有一个需要清理的局部thing，但不是由具有析构函数的对象表示，则此类清理也必须在抛出之前完成。有时，finally() 可以使这种不系统的清理更易于管理。见E.19。
 
<br/>
### **E.14:使用专门设计的用户定义类型作为例外（非内置类型）**

**原因** 一个用户定义的类型可以更好地将错误的信息传递给处理程序。信息可以被编码到类型本身，而且类型不太可能与其他人的异常发生冲突。

**Example**
```cpp
throw 7; // bad

throw "something bad";  // bad

throw std::exception{}; // bad - no info
```
从std::exception派生出来后，可以灵活地捕捉特定的异常，或者通过std::exception来处理。
```cpp
class MyException : public std::runtime_error
{
public:
    MyException(const string& msg) : std::runtime_error{msg} {}
    // ...
};

// ...

throw MyException{"something bad"};  // good
```
异常不需要从std::exception派生：
```cpp
class MyCustomError final {};  // not derived from std::exception

// ...

throw MyCustomError{};  // good - handlers must catch this type (or ...)
```
从std::exception派生的库类型，如果在检测点上不能添加有用的信息，可以作为通用的异常。
```cpp
throw std::runtime_error("someting bad"); // good

// ...

throw std::invalid_argument("i is not even"); // good
```
枚举类也可以：
```cpp
enum class alert {RED, YELLOW, GREEN};

throw alert::RED; // good
```

**Enforcement** 捕捉内置类型和std::异常的抛出。

<br/>
### **E.15:通过值抛出异常，通过引用从一个层次结构中捕捉异常**

**原因** 按value抛出（而不是按指针）和按引用捕捉可以防止复制，特别是slicing基础子对象。

**Example, bad**
```cpp
void f()
{
    try {
        // ...
        throw new widget{}; // don't: throw by value not by raw pointer
        // ...
    }
    catch (base_class e) {  // don't: might slice
        // ...
    }
}
```
Instead, use a reference:
```cpp
catch (base_class& e) { /* ... */ }
```
或通常更好的是一个常量引用。
```cpp
catch (const base_class& e) { /* ... */ }
```
大多数处理程序不会修改它们的异常，一般来说，我们建议使用const。

**注意** 对于一个小的值类型，如枚举值，按值catch是合适的。

**注意** 要重新抛出一个被捕获的异常，请使用throw;而不是throw e;。使用throw e;会抛出一个新的e副本
（切分到静态类型std::exception，当异常被catch (const std::exception& e)捕获时），而不是重新抛出原始的std::runtime\_error类型的异常。
(但请记住，不要试图在每个函数中捕捉每个异常，并尽量减少显式try/catch的使用。)

**Enforcement**
- 捕获一个带虚函数的类型的value要标记出
- 抛出裸指针要标记出

<br/>
### **E.16:析构函数、释放、swap和异常类型的复制/移动构造绝不能失败**

**原因** 如果析构函数、交换、内存释放或尝试复制/移动构造异常对象失败，我们不知道如何编写可靠的程序；也就是说，如果它因异常退出或根本不执行其所需的操作。

**Example, don’t**
```cpp
class Connection {
    // ...
public:
    ~Connection()   // Don't: very bad destructor
    {
        if (cannot_disconnect()) throw I_give_up{information};
        // ...
    }
};
```

**注意** 例如，许多人试图编写违反此规则的可靠代码，例如“拒绝关闭”的网络连接。据我们所知，还没有人找到通用的方法来做到这一点。
有时，对于非常具体的示例，您可以为将来的清理设置一些状态。例如，我们可能会将不想关闭的套接字放在“bad套接字”列表中，
以通过定期扫描系统状态进行检查。我们看到的每个例子都是容易出错的、专门的，而且经常有错误。

**注意** 标准库假定析构函数、释放函数（例如，delete）和swap不throw。如果他们这样做，基本的标准库不变量就会被破坏。

**注意** 
- 释放函数（包括delete）必须是noexcept。
- swap函数必须是noexcept。
- 默认情况下，大多数析构函数都是隐式的noexcept 。
- 此外，使移动操作noexcept。
- 如果编写旨在用作异常的类型，请确保其复制构造函数不是noexcept。一般来说，我们不能机械地执行这一点，因为我们不知道一个类型是否打算作为一个异常类型使用。
- 尽量不要抛出其复制构造函数不是noexcept的类型。通常我们不能机械地强制执行此操作，因为即使抛出std::string(...)也可以throw，但在实践中不会。

**Enforcement**
- 捕获抛出的析构函数、释放操作和swap。
- 捕捉这种不属于noexcept的操作。

<br/>
### **E.17:不要试图在每个函数中捕捉每个异常**

**原因** 在一个不能采取有意义的恢复措施的函数中捕捉异常，会导致复杂性和浪费。让异常传播，直到它到达一个可以处理它的函数。
让unwind路径上的清理动作由RAII来处理。

**Example, don’t**
```cpp
void f()   // bad
{
    try {
        // ...
    }
    catch (...) {
        // no action
        throw;   // propagate exception
    }
}
```

**Enforcement**
- 嵌套的try要标记出
- 源代码文件中的try段与函数的比例过高要标记出

<br/>
### **E.18:尽量减少对显式try/catch的使用**

**原因** try/catch是冗长的，非琐碎的使用容易出错。try/catch可能是不系统的和/或低级别的资源管理或错误处理的标志。

**Example, Bad**
```cpp
void f(zstring s)
{
    Gadget* p;
    try {
        p = new Gadget(s);
        // ...
        delete p;
    }
    catch (Gadget_construction_failure) {
        delete p;
        throw;
    }
}
```
这段代码很乱。try段中的裸指针可能会有泄漏。并非所有的异常都会被处理。删除一个构造失败的对象，几乎可以肯定是一个错误。
```cpp
void f2(zstring s)
{
    Gadget g {s};
}
```

**替代**
- 适当的资源处理和RAII
- finally

<br/>
### **E.19:如果没有合适的资源句柄，使用final_action对象来表达清理工作**

**原因** GSL的finally比try/catch更简洁，也更难出错。

**Example**
```cpp
void f(int n)
{
    void* p = malloc(n);
    auto _ = gsl::finally([p] { free(p); });
    // ...
}
```

**注意** finally不像try/catch那样混乱，但它仍然是临时性的。更推荐的资源管理对象。把finally看作是最后的手段。

**注意** finally的使用是对旧的goto exit的一种系统且相当干净的替代方法；在资源管理不系统的情况下处理cleanup的技术。

<br/>
### **E.25:如果你不能抛出异常，模拟RAII进行资源管理**

**原因** 即使没有异常，RAII通常是处理资源的最佳和最系统的方式。

**注意** 使用异常的错误处理是C++中处理非局部错误的唯一完整和系统的方法。特别是，以非侵入方式发出构建对象失败的信号需要异常。
以无法忽略的方式发出错误信号需要异常。如果您不能使用异常，请尽可能地模拟它们的使用。

**Example**
```cpp
void func(zstring arg)
{
    Gadget g {arg};
    // ...
}
```
如果Gadget没有被正确构造，func就会以一个异常退出。如果我们不能抛出一个异常，我们可以通过给Gadget添加一个valid()成员函数来模拟这种RAII风格的资源处理。
```cpp
error_indicator func(zstring arg)
{
    Gadget g {arg};
    if (!g.valid()) return gadget_construction_error;
    // ...
    return 0;   // zero indicates "good"
}
```
当然，问题是调用者现在必须记住测试返回值。为了鼓励这样做，可以考虑添加一个[[nodiscard]]。

<br/>
### **E.26:如果你不能抛出异常，考虑快速失败**

**原因** 如果你不能做好恢复工作，至少你可以在太多的后果性损害发生之前脱身。

**注意** 如果您不能系统地处理错误，请将"crash"视为对无法在本地处理的任何错误的响应。也就是说，如果您无法从检测到它的函数的上下文中的错误中恢复，
请调用abort()、quick\_exit()或将触发某种系统重启的类似函数。

在你有很多进程和/或很多计算机的系统中，无论如何你都需要预料到并处理致命的崩溃，比如硬件故障。在这种情况下，“崩溃”只是将错误处理留给系统的下一级。

**Example**
```cpp
void f(int n)
{
    // ...
    p = static_cast<X*>(malloc(n * sizeof(X)));
    if (!p) abort();     // abort if memory is exhausted
    // ...
}
```
大多数程序无论如何都不能优雅地处理内存耗尽的问题。这大致上相当于
```cpp
void f(int n)
{
    // ...
    p = new X[n];    // throw if memory is exhausted (by default, terminate)
    // ...
}
```
通常情况下，在退出前记录 "崩溃 "的原因是一个好主意。

<br/>
### **E.27:如果你不能抛出异常，请系统地使用错误代码**

**原因** 系统地使用任何错误处理策略可以最大限度地减少忘记处理错误的机会。

**注意** 有几个问题需要解决：
- 您如何从函数中传输错误指示符？
- 在执行错误退出之前如何从函数中释放所有资源？
- 你用什么作为错误指示器？
一般来说，返回一个错误指示器意味着返回两个值。结果和一个错误指示器。错误指示器可以是对象的一部分，例如，一个对象可以有一个valid()指示器，或者可以返回一对值。

**Example**
```cpp
Gadget make_gadget(int n)
{
    // ...
}

void user()
{
    Gadget g = make_gadget(17);
    if (!g.valid()) {
            // error handling
    }
    // ...
}
```
这种方法适合模拟RAII资源管理。valid()函数可以返回一个error\_indicator（例如error\_indicator枚举的成员）。

**Example**
如果我们不能或不想修改Gadget的类型呢？在这种情况下，我们必须返回一对值。
```cpp
std::pair<Gadget, error_indicator> make_gadget(int n)
{
    // ...
}

void user()
{
    auto r = make_gadget(17);
    if (!r.second) {
            // error handling
    }
    Gadget& g = r.first;
    // ...
}
```
如图所示，std::pair是一种可能的返回类型。有些人喜欢一个特定的类型。
```cpp
Gval make_gadget(int n)
{
    // ...
}

void user()
{
    auto r = make_gadget(17);
    if (!r.err) {
            // error handling
    }
    Gadget& g = r.val;
    // ...
}
```
喜欢特定返回类型的一个原因是为其成员命名，而不是有点神秘的first和second，并避免与std::pair的其他用途混淆。

**Example** 一般来说，你必须在错误退出之前进行清理。这可能会很混乱。
```cpp
std::pair<int, error_indicator> user()
{
    Gadget g1 = make_gadget(17);
    if (!g1.valid()) {
        return {0, g1_error};
    }

    Gadget g2 = make_gadget(31);
    if (!g2.valid()) {
        cleanup(g1);
        return {0, g2_error};
    }

    // ...

    if (all_foobar(g1, g2)) {
        cleanup(g2);
        cleanup(g1);
        return {0, foobar_error};
    }

    // ...

    cleanup(g2);
    cleanup(g1);
    return {res, 0};
}
```
模拟RAII可能很重要，尤其是在具有多种资源和多种可能错误的函数中。一种并不少见的技术是在函数末尾收集清理以避免重复
（请注意，g2周围的额外范围是不可取的，但对于使goto版本编译是必需的）
```cpp
std::pair<int, error_indicator> user()
{
    error_indicator err = 0;
    int res = 0;

    Gadget g1 = make_gadget(17);
    if (!g1.valid()) {
        err = g1_error;
        goto g1_exit;
    }

    {
        Gadget g2 = make_gadget(31);
        if (!g2.valid()) {
            err = g2_error;
            goto g2_exit;
        }

        if (all_foobar(g1, g2)) {
            err = foobar_error;
            goto g2_exit;
        }

        // ...

    g2_exit:
        if (g2.valid()) cleanup(g2);
    }

g1_exit:
    if (g1.valid()) cleanup(g1);
    return {res, err};
}
```
函数越大，这种技术就越有诱惑力。最后可以减轻一点痛苦。另外，程序越大，就越难系统地应用基于错误指示器的错误处理策略。

我们更喜欢基于异常的错误处理，并建议保持函数简短。

<br/>
### **E.28:避免基于全局状态的错误处理（如errno）**

**原因** 全局状态很难管理，很容易忘记检查它。你上次测试printf()的返回值是什么时候？

**Example, bad**
```cpp
int last_err;

void f(int n)
{
    // ...
    p = static_cast<X*>(malloc(n * sizeof(X)));
    if (!p) last_err = -1;     // error if memory is exhausted
    // ...
}
```

**注意** C风格的错误处理是基于全局变量errno，所以完全避免这种风格本质上是不可能的。

<br/>
### **E.30:不要使用异常规范**

**原因** 异常规范使错误处理变得很脆弱，带来了运行时的成本，并已从C++标准中删除。

**Example**
```cpp
int use(int arg)
    throw(X, Y)
{
    // ...
    auto x = f(arg);
    // ...
}
```

**注意** 如果f()抛出一个与X和Y不同的异常，那么就会调用意外处理程序，默认情况下会终止。这没问题，但是如果我们已经检查过这种情况不会发生，
而f被改变为抛出一个新的异常Z，那么我们现在就有一个崩溃，除非我们改变use()（并重新测试一切）。问题在于，f()可能在一个我们无法控制的库中，
而新的异常并不是use()能够处理的，也不是use()所感兴趣的。我们可以改变use()来传递Z，但现在use()的调用者可能需要修改。
这很快就会变得无法管理。另外，我们可以给use()添加一个try-catch，将Z映射成一个可接受的异常。这也会很快变得无法管理。
请注意，对异常集的改变往往发生在系统的最底层（例如，由于对网络库或一些中间件的改变），所以改变通过长的调用链 "冒泡"。
在一个大的代码库中，这可能意味着在最后一个用户被修改之前，没有人可以更新到一个新版本的库。如果use()是一个库的一部分，可能无法更新它，因为一个变化可能会影响到未知的客户端。

让异常传播直到它们到达可能处理它的函数的策略多年来已经证明了自己。

**注意** 如果不能抛出异常，请使用noexcept。

<br/>
### **E.31:正确排序你的catch子句**

**原因** catch-clauses是按照它们出现的顺序进行评估的，一个子句可以隐藏另一个。

**Example, bad**
```cpp
void f()
{
    // ...
    try {
            // ...
    }
    catch (Base& b) { /* ... */ }
    catch (Derived& d) { /* ... */ }
    catch (...) { /* ... */ }
    catch (std::exception& e) { /* ... */ }
}
```
如果Derived是从Base派生出来的，Derived处理程序将永远不会被调用。"catch everything"处理程序确保了std::exception-handler永远不会被调用。

<br/>
# **ES:表达式和语句**

表达式和语句是表达动作和计算的最低和最直接的方式。本地作用域中的声明是语句。

关于命名、注释和缩进规则，见[NL.命名和布局](#nl命名和布局)

一般规则：
- [ES.1:比起其他库和 "手工制作的代码"，优先使用标准库](#es1比起其他库和-手工制作的代码优先使用标准库) 
- [ES.2:优先使用合适的抽象，而不是直接使用语言特征](#es2优先使用合适的抽象而不是直接使用语言特征) 
- [ES.3:不要重复，避免多余的代码](#es3不要重复避免多余的代码) 

声明规则：
- [ES.5:保持小范围](#es5保持小范围) 
- [ES.6:在for-statement初始化和条件中声明名称，以限制范围](#es6在for-statement初始化和条件中声明名称以限制范围) 
- [ES.7:常用和局部命名要简短，不常见和非局部的命名要长一些](#es7常用和局部命名要简短不常见和非局部的命名要长一些) 
- [ES.8:避免类似的名字](#es8避免类似的名字) 
- [ES.9:避免使用ALL\_CAPS名字](#es9避免使用all_caps名字) 
- [ES.10:每个声明只声明一个名字](#es10每个声明只声明一个名字) 
- [ES.11:使用auto来避免类型名称的冗余重复](#es11使用auto来避免类型名称的冗余重复) 
- [ES.12:不要在嵌套作用域中重复使用名字](#es12不要在嵌套作用域中重复使用名字) 
- [ES.20:对象总是要初始化](#es20对象总是要初始化) 
- [ES.21:在你需要使用一个变量（或常数）之前，不要引入它](#es21在你需要使用一个变量或常数之前不要引入它) 
- [ES.22:在你有一个值来初始化它之前，不要声明一个变量](#es22在你有一个值来初始化它之前不要声明一个变量) 
- [ES.23:优先使用{}初始化的语法](#es23优先使用初始化的语法) 
- [ES.24:使用unique\_ptr\<T\>来保存指针](#es24使用unique_ptrt来保存指针) 
- [ES.25:声明一个对象为const或constexpr，除非你想在以后修改它的值](#es25声明一个对象为const或constexpr除非你想在以后修改它的值) 
- [ES.26:不要将一个变量用于两个不相关的目的](#es26不要将一个变量用于两个不相关的目的) 
- [ES.27:使用std::array或stack\_array处理堆栈中的数组](#es27使用stdarray或stack_array处理堆栈中的数组) 
- [ES.28:使用lambda进行复杂的初始化，特别是const变量的初始化](#es28使用lambda进行复杂的初始化特别是const变量的初始化) 
- [ES.30:不要使用宏来进行程序文本操作](#es30不要使用宏来进行程序文本操作) 
- [ES.31:不要将宏用于常量或函数](#es31不要将宏用于常量或函数) 
- [ES.32:对所有的宏名称使用ALL\_CAPS](#es32对所有的宏名称使用all_caps) 
- [ES.33:如果你必须使用宏，请给它们一个唯一名字](#es33如果你必须使用宏请给它们一个唯一名字) 
- [ES.34:不要定义一个（C风格的）variadic函数](#es34要定义一个c风格的variadic函数) 

表达式规则：
- [ES.40:避免复杂的表达式](#es40避免复杂的表达式) 
- [ES.41:如果对运算符的优先级有疑问，可以用括号表示](#es41如果对运算符的优先级有疑问可以用括号表示) 
- [ES.42:保持对指针的使用简单明了](#es42保持对指针的使用简单明了) 
- [ES.43:避免使用未定义评估顺序的表达式](#es43避免使用未定义评估顺序的表达式) 
- [ES.44:不依赖函数参数的评估顺序](#es44不依赖函数参数的评估顺序) 
- [ES.45:避免magic constants，使用符号常量](#es45避免magic-constants使用符号常量) 
- [ES.46:避免缩小转换范围](#es46避免缩小转换范围) 
- [ES.47:使用nullptr而不是0或NULL](#es47使用nullptr而不是0或null) 
- [ES.48:避免cast](#es48避免cast) 
- [ES.49:如果必须使用cast，使用一个命名的cast](#es49如果必须使用cast使用一个命名的cast) 
- [ES.50:不要对const对象使用cast](#es50不要对const对象使用cast) 
- [ES.55:避免范围检查的需要](#es55避免范围检查的需要) 
- [ES.56:只有当你需要明确地将一个对象移动到另一个范围时，才写std::move()](#es56只有当你需要明确地将一个对象移动到另一个范围时才写stdmove) 
- [ES.60:资源管理函数外避免new和delete](#es60资源管理函数外避免new和delete) 
- [ES.61:数组用delete[]，其他的用delete](#es61数组用delete其他的用delete) 
- [ES.62:不要比较指向不同数组的指针](#es62不要比较指向不同数组的指针) 
- [ES.63:不要切片](#es63不要切片) 
- [ES.64:使用T{e}表示法进行构造](#es64使用te表示法进行构造) 
- [ES.65:不要对无效指针解引用](#es65不要对无效指针解引用) 

语句规则：
- [ES.70:在有选择的情况下，更倾向于使用switch语句而不是if语句](#es70在有选择的情况下更倾向于使用switch语句而不是if语句) 
- [ES.71:当有选择时，优先用range-for语句而不是for语句](#es71当有选择时优先用range-for语句而不是for语句) 
- [ES.72:当有明显的循环变量时，更倾向于使用for语句而不是while语句](#es72当有明显的循环变量时更倾向于使用for语句而不是while语句) 
- [ES.73:当没有明显的循环变量时，更倾向于使用while语句而不是for语句](#es73当没有明显的循环变量时更倾向于使用while语句而不是for语句) 
- [ES.74:倾向于在for语句的初始化部分声明一个循环变量](#es74倾向于在for语句的初始化部分声明一个循环变量) 
- [ES.75:避免使用do语句](#es75避免使用do语句) 
- [ES.76:避免使用goto](#es76避免使用goto) 
- [ES.77:在循环中最小化使用break和continue](#es77在循环中最小化使用break和continue) 
- [ES.78:不要在switch语句中依赖隐式的fallthrough](#es78不要在switch语句中依赖隐式的fallthrough) 
- [ES.79:使用default来处理通用的情况](#es79使用default来处理通用的情况) 
- [ES.84:不要试图声明一个没有名字的局部变量](#es84不要试图声明一个没有名字的局部变量) 
- [ES.85:让空语句可见](#es85让空语句可见) 
- [ES.86:避免在for-loop的主体内修改循环控制变量](#es86避免在for-loop的主体内修改循环控制变量) 
- [ES.87:不要在条件中添加多余的==或!=](#es87不要在条件中添加多余的或) 

运算规则：
- [ES.100:不要混合有符号和无符号的计算](#es100不要混合有符号和无符号的计算) 
- [ES.101:位操作要使用无符号类型](#es101位操作要使用无符号类型) 
- [ES.102:算数计算要使用有符号类型](#es102算数计算要使用有符号类型) 
- [ES.103:不要溢出](#es103不要溢出) 
- [ES.104:不要下溢](#es104不要下溢) 
- [ES.105:不要被整数零除掉](#es105不要被整数零除掉) 
- [ES.106:不要试图通过使用无符号来避免负值](#es106不要试图通过使用无符号来避免负值) 
- [ES.107:不要对下标使用无符号，最好使用gsl::index](#es107不要对下标使用无符号最好使用gslindex) 

<br/>
### **ES.1:比起其他库和 "手工制作的代码"，优先使用标准库**

**原因** 使用库的代码可能比直接使用语言特性的代码更容易编写，更短，往往具有更高的抽象水平，而且库的代码可能已经被测试过。
ISO C++标准库是最广为人知和经过最佳测试的库之一。它可以作为所有C++实现的一部分。

**Example**
```cpp
auto sum = accumulate(begin(a), end(a), 0.0);   // good
```
以下更好：
```cpp
auto sum = accumulate(v, 0.0); // better
```
最好不要手写：
```cpp
int max = v.size();   // bad: verbose, purpose unstated
double sum = 0.0;
for (int i = 0; i < max; ++i)
    sum = sum + v[i];
```

**例外** 标准库的很大一部分依赖于动态分配（自由存储）。这些部分，特别是容器，但不是算法，不适合一些硬实时和嵌入式应用。
在这种情况下，考虑提供/使用类似的设施，例如，使用pool allocator实现的标准库式容器。

**Enforcement** 不容易？？？找出混乱的循环、嵌套循环、长函数、没有函数调用、没有使用非内置类型的情况。

<br/>
### **ES.2:优先使用合适的抽象，而不是直接使用语言特征**

**原因** 一个 "合适的抽象"（例如，库或类）比raw语言更接近应用概念，导致更短和更清晰的代码，并可能得到更好的测试。

**Example**
```cpp
vector<string> read1(istream& is)   // good
{
    vector<string> res;
    for (string s; is >> s;)
        res.push_back(s);
    return res;
}
```
更传统、更低级的近乎等价物是更长、更混乱、更难弄好，而且很可能更慢。
```cpp
char** read2(istream& is, int maxelem, int maxstring, int* nread)   // bad: verbose and incomplete
{
    auto res = new char*[maxelem];
    int elemcount = 0;
    while (is && elemcount < maxelem) {
        auto s = new char[maxstring];
        is.read(s, maxstring);
        res[elemcount++] = s;
    }
    nread = &elemcount;
    return res;
}
```
一旦增加了溢出检查和错误处理，代码就会变得相当混乱，而且还有一个问题，就是要记住delete返回的指针和数组中包含的C语言字符串。

**Enforcement** 不容易？？？找出混乱的循环、嵌套循环、长函数、没有函数调用、没有使用非内置类型的情况。

<br/>
### **ES.3:不要重复，避免多余的代码**

重复或多余的代码掩盖了意图，使人更难理解逻辑，并使维护更加困难，以及其他一些问题。它通常是由剪切和粘贴的编程产生的。

适当时使用标准算法，而不是自己编写一些实现。

另外可见[SL.1](#sl1尽可能使用库)[ES.11](#es11使用auto来避免类型名称的冗余重复)

**Example**
```cpp
void func(bool flag)    // Bad, duplicated code.
{
    if (flag) {
        x();
        y();
    }
    else {
        x();
        z();
    }
}

void func(bool flag)    // Better, no duplicated code.
{
    x();

    if (flag)
        y();
    else
        z();
}
```

**Enforcement**
- 使用一个静态分析。它至少会抓住一些多余的结构。
- Code review。

<br/>
## **ES.dcl:声明**

声明是一个语句。声明将一个名字引入到一个作用域中，并可能引起一个命名对象的构造。

<br/>
### **ES.5:保持小范围**

**原因** 可读性。最大限度地减少资源保留。避免意外的价误用。

**替代方案** 不要在一个不必要的大范围内声明一个名字。

**Example**
```cpp
void use()
{
    int i;    // bad: i is needlessly accessible after loop
    for (i = 0; i < 20; ++i) { /* ... */ }
    // no intended use of i here
    for (int i = 0; i < 20; ++i) { /* ... */ }  // good: i is local to for-loop

    if (auto pc = dynamic_cast<Circle*>(ps)) {  // good: pc is local to if-statement
        // ... deal with Circle ...
    }
    else {
        // ... handle error ...
    }
}
```

**Example,bad**
```cpp
void use(const string& name)
{
    string fn = name + ".txt";
    ifstream is {fn};
    Record r;
    is >> r;
    // ... 200 lines of code without intended use of fn or is ...
}
```
这个函数在大多数情况下都太长了，但问题是fn使用的资源和is持有的文件句柄保留的时间比需要的要长得多，
而且在函数的后面可能会发生对is和fn的意料之外的使用。在这种情况下，分解读取可能是个好主意：
```cpp
Record load_record(const string& name)
{
    string fn = name + ".txt";
    ifstream is {fn};
    Record r;
    is >> r;
    return r;
}

void use(const string& name)
{
    Record r = load_record(name);
    // ... 200 lines of code ...
}
```

**Enforcement**
- 循环变量在循环外声明，在循环后不使用则标记出
- 当昂贵的资源，如文件句柄和锁不被用于N行时（对于某个合适的N）则标记出。

<br/>
### **ES.6:在for-statement初始化和条件中声明名称，以限制范围**

**原因** 可读性。将循环变量的可见性限制在循环的范围内。避免在循环之后将循环变量用于其他目的。最大限度地减少资源保留。

**Example**
```cpp
void use()
{
    for (string s; cin >> s;)
        v.push_back(s);

    for (int i = 0; i < 20; ++i) {   // good: i is local to for-loop
        // ...
    }

    if (auto pc = dynamic_cast<Circle*>(ps)) {   // good: pc is local to if-statement
        // ... deal with Circle ...
    }
    else {
        // ... handle error ...
    }
}
```

**Example,don't**
```cpp
int j;                            // BAD: j is visible outside the loop
for (j = 0; j < 100; ++j) {
    // ...
}
// j is still visible here and isn't needed
```

另外见：[ES.26:不要将一个变量用于两个不相关的目的](#es26不要将一个变量用于两个不相关的目的)

**Enforcement**
- 当在for语句中修改的变量在循环外声明，并且没有在循环外使用时发出警告
- （难）在循环之前声明的循环变量，在循环之后用于无关的目的要标记出

**讨论:** 将循环变量限定在循环体中也对代码优化器有很大帮助。认识到归纳变量只能在循环体中访问可以解锁优化，例如提升、强度降低、循环不变代码移动等。

**C++17 and C++20 example**
注意：C++17和C++20还增加了if、switch和range-for初始化器语句。
```cpp
map<int, string> mymap;

if (auto result = mymap.insert(value); result.second) {
    // insert succeeded, and result is valid for this block
    use(result.first);  // ok
    // ...
} // result is destroyed here
```

**C++17 and C++20 enforcement (if using a C++17 or C++20 compiler)**
- 选择/循环变量在body之前声明，在body之后不使用的变量要标记出
- 在body之前声明的选择/循环变量，在body之后用于不相关的目的要标记出

<br/>
### **ES.7:常用和局部命名要简短，不常见和非局部的命名要长一些**

**原因** 可读性。降低不相关的非局部名字冲突的机会。

**Example** 传统的简短的局部名字增加了可读性。
```cpp
template<typename T>    // good
void print(ostream& os, const vector<T>& v)
{
    for (gsl::index i = 0; i < v.size(); ++i)
        os << v[i] << '\n';
}
```
一个索引通常被称为i，在这个元函数中没有提示vector的含义，所以v是一个很好的名字。比较下：
```cpp
template<typename Element_type>   // bad: verbose, hard to read
void print(ostream& target_stream, const vector<Element_type>& current_vector)
{
    for (gsl::index current_element_index = 0;
         current_element_index < current_vector.size();
         ++current_element_index
    )
    target_stream << current_vector[current_element_index] << '\n';
}
```

**Example** 非常规和简短的非局部名称使代码模糊不清。
```cpp
void use1(const string& s)
{
    // ...
    tt(s);   // bad: what is tt()?
    // ...
}
```
给非局部实体一个可读的名字更好：
```cpp
void use1(const string& s)
{
    // ...
    trim_tail(s);   // better
    // ...
}
```
在这里，读者有可能知道trim\_tail的意思，而且读者在查找后能记住它。

**Example,bad** 大型函数的参数名事实上是非局部的，应该是有意义的。
```cpp
void complicated_algorithm(vector<Record>& vr, const vector<int>& vi, map<string, int>& out)
// read from events in vr (marking used Records) for the indices in
// vi placing (name, index) pairs into out
{
    // ... 500 lines of code using vr, vi, and out ...
}
```
我们建议保持函数简短，但这一规则并没有得到普遍遵守，命名应该反映这一点。

**Enforcement** 检查局部和非局部名称的长度。还要考虑到函数的长度。

<br/>
### **ES.8:避免类似的名字**

**原因** 代码的清晰度和可读性。太过相似的名称会减慢理解速度，增加出错的可能性。

**Example,bad**
```cpp
if (readable(i1 + l1 + ol + o1 + o0 + ol + o1 + I0 + l0)) surprise();
```

**Example,bad** 不要声明一个与同一作用域中的类型同名的非类型。这样就不需要用诸如struct或enum这样的关键字来消除歧义。
它还消除了一个错误源，因为如果查找失败，struct X可以隐含地声明X。
```cpp
struct foo { int n; };
struct foo foo();       // BAD, foo is a type already in scope
struct foo x = foo();   // requires disambiguation
```

**例外** 古老的头文件可能在同一范围内声明非类型和具有相同名称的类型。

**Enforcement**
- 将名字与已知的混乱的字母和数字组合的清单进行核对
- 一个变量、函数或枚举器的声明，隐藏了在同一范围内声明的类或枚举要标记出

<br/>
### **ES.9:避免使用ALL_CAPS名字**

**原因** 这种名称通常用于宏。因此，ALL\_CAPS名称很容易被非故意的宏替换。 

**Example**
```cpp
// somewhere in some header:
#define NE !=

// somewhere else in some other header:
enum Coord { N, NE, NW, S, SE, SW, E, W };

// somewhere third in some poor programmer's .cpp:
switch (direction) {
case N:
    // ...
case NE:
    // ...
// ...
}
```

**注意** 不要因为常数曾经是宏，就对常数使用ALL\_CAPS。

**Enforcement** 所有使用ALL CAPS的情况要标记出。所有使用ALL CAPS的情况，所有非ALL-CAPS的宏名称要标记出。

<br/>
### **ES.10:每个声明只声明一个名字**

**原因** 每行一个声明可提高可读性并避免与C/C++语法相关的错误。它还为更具描述性的行尾注释留出了空间。

**Example, bad**
```cpp
char *p, c, a[7], *pp[7], **aa[10];   // yuck!
```

**例外** 一个函数声明可以包含几个函数参数的声明。

**例外** 一个结构化的绑定（C++17）是专门用来引入几个变量的。
```cpp
auto [iter, inserted] = m.insert_or_assign(k, val);
if (inserted) { /* new entry was inserted */ }
```

**Example**
```cpp
template<class InputIterator, class Predicate>
bool any_of(InputIterator first, InputIterator last, Predicate pred);
```
或用concept更好：
```cpp
bool any_of(input_iterator auto first, input_iterator auto last, predicate auto pred);
```

**Example**
```cpp
double scalbn(double x, int n);   // OK: x * pow(FLT_RADIX, n); FLT_RADIX is usually 2
```
or:
```cpp
double scalbn(    // better: x * pow(FLT_RADIX, n); FLT_RADIX is usually 2
    double x,     // base value
    int n         // exponent
);
```
or:
```cpp
// better: base * pow(FLT_RADIX, exponent); FLT_RADIX is usually 2
double scalbn(double base, int exponent);
```

**Example**
```cpp
int a = 10, b = 11, c = 12, d, e = 14, f = 15;
```

**Enforcement** 带有多个声明的变量和常量声明要标记出

<br/>
### **ES.11:使用auto来避免类型名称的冗余重复**

**原因**
- 简单的重复是乏味和容易出错的。
- 当你使用auto时，被声明的实体的名字在声明中处于一个固定的位置，增加了可读性。
- 在一个函数模板声明中，返回类型可以是一个成员类型。

**Example**
```cpp
auto p = v.begin();      // vector<DataRecord>::iterator
auto z1 = v[3];          // makes copy of DataRecord
auto& z2 = v[3];         // avoids copy
const auto& z3 = v[3];   // const and avoids copy
auto h = t.future();
auto q = make_unique<int[]>(s);
auto f = [](int x) { return x + 10; };
```
在每一种情况下，我们都省去了写一个长长的、难以记住的类型，这些类型是编译器已经知道的，但程序员可能会弄错。

**Example**
```cpp
template<class T>
auto Container<T>::first() -> Iterator;   // Container<T>::Iterator
```

**例外**
在初始化列表中，以及在你确切地知道你想要的类型和初始化可能需要转换的情况下，避免使用auto。

**Example**
```cpp
auto lst = { 1, 2, 3 };   // lst is an initializer list
auto x{1};   // x is an int (in C++17; initializer_list in C++11)
```

**注意** 从C++20开始，我们可以（也应该）使用concept来更具体地说明我们正在推导的类型。
```cpp
// ...
forward_iterator auto p = algo(x, y, z);
```

**Example(C++17)**
```cpp
std::set<int> values;
// ...
auto [ position, newly_inserted ] = values.insert(5);   // break out the members of the std::pair
```

**Enforcement**
- 声明中类型名称的冗余重复要标记出。

<br/>
### **ES.12:不要在嵌套作用域中重复使用名字**

**原因** 很容易混淆哪个变量被使用。可能会引起维护问题。

**Example, bad**
```cpp
int d = 0;
// ...
if (cond) {
    // ...
    d = 9;
    // ...
}
else {
    // ...
    int d = 7;
    // ...
    d = value_to_be_returned;
    // ...
}

return d;
```
如果这是一个大的if语句，很容易忽略在内部范围中引入了一个新的d。这是一个已知的错误来源。有时，这种在内部作用域中重复使用一个名字的做法被称为"shadowing"。

**注意** shadowing主要是在功能太大、太复杂时出现的问题。
```cpp
void f(int x)
{
    int x = 4;  // error: reuse of function argument name

    if (x) {
        int x = 7;  // allowed, but bad
        // ...
    }
}
```

**Example, bad** 将一个成员的名字作为局部变量重复使用，也可能是一个问题。
```cpp
struct S {
    int m;
    void f(int x);
};

void S::f(int x)
{
    m = 7;    // assign to member
    if (x) {
        int m = 9;
        // ...
        m = 99; // assign to local variable
        // ...
    }
}
```

**例外** 我们经常在派生类中重复使用基类的函数名。
```cpp
struct B {
    void f(int);
};

struct D : B {
    void f(double);
    using B::f;
};
```
这是很容易出错的。例如，如果我们忘记了using声明，调用d.f(1)就不会找到f的int版本。

**Enforcement**
- 嵌套的局部作用域中重复使用一个名称要标记出。
- 在成员函数中把一个成员的名字作为局部变量重新使用要标记出。
- 将全局名称作为局部变量或成员名称重新使用要标记出。
- 在派生类中重复使用基类成员名（函数名除外）要标记出。

<br/>
### **ES.20:对象总是要初始化**

**原因** 避免used-before-set的错误及其相关的未定义行为。避免对复杂初始化的理解问题。简化了重构。

**Example**
```cpp
void use(int arg)
{
    int i;   // bad: uninitialized variable
    // ...
    i = 7;   // initialize i
}
```
i = 7并没有初始化i，而是对其进行赋值。另外，i可以在...部分被读取。
```cpp
void use(int arg)   // OK
{
    int i = 7;   // OK: initialized
    string s;    // OK: default initialized
    // ...
}
```

```cpp
constexpr int max = 8 * 1024;
int buf[max];         // OK, but suspicious: uninitialized
f.read(buf, max);
```
```cpp
constexpr int max = 8 * 1024;
int buf[max] = {};   // zero all elements; better in some situations
f.read(buf, max);
```

不要认为作为输入操作目标的简单变量会是这条规则的例外。
```cpp
int i;   // bad
// ...
cin >> i;
```
```cpp
int i2 = 0;   // better, assuming that zero is an acceptable value for i2
// ...
cin >> i2;
```

**注意** 有时，lambda可以作为初始化使用，以避免未初始化的变量。
```cpp
error_code ec;
Value v = [&] {
    auto p = get_value();   // get_value() returns a pair<error_code, Value>
    ec = p.first;
    return p.second;
}();
```
or maybe:
```cpp
Value v = [] {
    auto p = get_value();   // get_value() returns a pair<error_code, Value>
    if (p.first) throw Bad_value{p.first};
    return p.second;
}();
```
另外见[ES.28](#使用lambda进行复杂的初始化特别是const变量的初始化)


**Enforcement**
- 每个未初始化的变量都要标记出。有默认构造函数的用户定义的类型的变量不要标记出。
- 检查一个未初始化的缓冲区是否在声明后立即被写进。传递一个未初始化的变量作为非const参数的引用可以被认为是写进了该变量。

<br/>
### **ES.21:在你需要使用一个变量（或常数）之前，不要引入它**

**原因** 可读性。限制变量可使用的范围。

**Example**
```cpp
int x = 7;
// ... no use of x here ...
++x;
```

**Enforcement** 与首次使用相距甚远的声明要标记出。

<br/>
### **ES.22:在你有一个值来初始化它之前，不要声明一个变量**

**原因** 可读性。限制变量可使用的范围。不要冒险在设置前使用。初始化往往比赋值更有效。

**Example, bad**
```cpp
string s;
// ... no use of s here ...
s = "what a waste";
```

**Example, bad**
```cpp
SomeLargeType var;  // Hard-to-read CaMeLcAsEvArIaBlE

if (cond)   // some non-trivial condition
    Set(&var);
else if (cond2 || !cond3) {
    var = Set2(3.14);
}
else {
    var = 0;
    for (auto& e : something)
        var += e;
}

// use var; that this isn't done too early can be enforced statically with only control flow
```
如果SomeLargeType有一个默认的初始化，并且不是太昂贵的话，这就很好。否则，程序员很可能会怀疑是否已经覆盖了通过条件迷宫的每一条可能的路径。
如果没有，我们就有一个"先使用后设置"的错误。这是一个维护陷阱。

对于中等复杂度的初始化器，包括对于常量变量，可以考虑使用lambda来表达初始化器。见[ES.28](#使用lambda进行复杂的初始化特别是const变量的初始化)

**Enforcement**
- 具有默认初始化的声明，这些声明在第一次被读取之前已被赋值要标记出。
- 在一个未初始化的变量之后和使用之前，任何复杂的计算都要被标记出。

<br/>
### **ES.23:优先使用{}初始化的语法**

**原因** 优先考虑{}。与其他形式的初始化相比，{}的初始化规则更简单、更通用、更不含糊，也更安全。
只有当你确定不可能有缩小的转换时才使用=。对于内置的算术类型，只有在使用auto时才使用=。

避免()的初始化，因为它增加解析的模糊性。

**Example**
```cpp
int x {f(99)};
int y = x;
vector<int> v = {1, 2, 3, 4, 5, 6};
```

**例外** 对于容器，有一个传统，即用{...}表示元素列表，用（...）表示大小。
```cpp
vector<int> v1(10);    // vector of 10 elements with the default value 0
vector<int> v2{10};    // vector of 1 element with the value 10

vector<int> v3(1, 2);  // vector of 1 element with the value 2
vector<int> v4{1, 2};  // vector of 2 elements with the values 1 and 2
```

**注意** {}初始化器不允许缩小转换（这通常是件好事），并允许显式构造器（这很好，我们有意在初始化一个新变量）。

**Example**
```cpp
int x {7.9};   // error: narrowing
int y = 7.9;   // OK: y becomes 7. Hope for a compiler warning
int z = gsl::narrow_cast<int>(7.9);  // OK: you asked for it
```

**注意** {}的初始化几乎可以用于所有的初始化；其他形式的初始化则不能。
```cpp
auto p = new vector<int> {1, 2, 3, 4, 5};   // initialized vector
D::D(int a, int b) :m{a, b} {   // member initializer (e.g., m might be a pair)
    // ...
};
X var {};   // initialize var to be empty
struct S {
    int m {7};   // default initializer for a member
    // ...
};
```
由于这个原因，{}-初始化通常被称为 "统一初始化"（尽管不幸的是还有一些不规则的东西）。

**注意** 用auto声明的变量的初始化只有一个值，例如{v}，在C++17之前有令人惊讶的结果。C++17的规则在某种程度上不那么令人惊讶。
```cpp
auto x1 {7};        // x1 is an int with the value 7
auto x2 = {7};      // x2 is an initializer_list<int> with an element 7

auto x11 {7, 8};    // error: two initializers
auto x22 = {7, 8};  // x22 is an initializer_list<int> with elements 7 and 8
```
如果你真的想要一个初始化器\_列表\<T\>，请使用={...}。
```cpp
auto fib10 = {1, 1, 2, 3, 5, 8, 13, 21, 34, 55};   // fib10 is a list
```

**注意** ={}拷贝初始化，而{}直接初始化。就像拷贝初始化和直接初始化本身的区别一样，这可能会导致意外的发生。{}接受显式构造函数；={}不接受。比如说:
```cpp
struct Z { explicit Z() {} };

Z z1{};     // OK: direct initialization, so we use explicit constructor
Z z2 = {};  // error: copy initialization, so we cannot use the explicit constructor
```
使用普通的{}初始化，除非你特别想禁用显式构造函数。

**Example**
```cpp
template<typename T>
void f()
{
    T x1(1);    // T initialized with 1
    T x0();     // bad: function declaration (often a mistake)

    T y1 {1};   // T initialized with 1
    T y0 {};    // default initialized T
    // ...
}
```

**Enforcement**
- 使用=来初始化发生缩小的算术类型要标记出。
- 使用()的初始化语法，实际上是声明要标记出。(许多编译器应该已经对此提出警告）。

<br/>
### **ES.24:使用unique_ptr\<T\>来保存指针**

**原因** 使用std::unique\_ptr是避免泄露的最简单方法。它是可靠的，它使类型系统做了很多验证所有权安全的工作，它增加了可读性，而且它的运行时间成本为零或接近零。

**Example**
```cpp
void use(bool leak)
{
    auto p1 = make_unique<int>(7);   // OK
    int* p2 = new int{7};            // bad: might leak
    // ... no assignment to p2 ...
    if (leak) return;
    // ... no assignment to p2 ...
    vector<int> v(7);
    v.at(7) = 0;                    // exception thrown
    // ...
}
```
如果leak == true，p2所指向的对象被泄露，p1所指向的对象没有被泄露。当at()抛出时，情况也是如此。

**Enforcement** 寻找作为new、malloc()目标的原始指针，或可能返回这种指针的函数。

<br/>
### **ES.25:声明一个对象为const或constexpr，除非你想在以后修改它的值**

**原因** 这样你就不会错误地改变数值。这种方式可能会给编译器提供优化的机会。

**Example**
```cpp
void f(int n)
{
    const int bufmax = 2 * n + 2;  // good: we can't change bufmax by accident
    int xmax = n;                  // suspicious: is xmax intended to change?
    // ...
}
```

<br/>
### **ES.26:不要将一个变量用于两个不相关的目的**

**原因** 可读性和安全。

**Example, bad**
```cpp
void use()
{
    int i;
    for (i = 0; i < 20; ++i) { /* ... */ }
    for (i = 0; i < 200; ++i) { /* ... */ } // bad: i recycled
}
```

**注意** 作为一种优化，你可能想重用一个缓冲区作，但即使如此，也最好尽可能地限制变量的范围，
并注意不要因为留在循环缓冲区中的数据而导致bug，因为这是安全bug的一个常见来源。
```cpp
void write_to_file()
{
    std::string buffer;             // to avoid reallocations on every loop iteration
    for (auto& o : objects) {
        // First part of the work.
        generate_first_string(buffer, o);
        write_to_file(buffer);

        // Second part of the work.
        generate_second_string(buffer, o);
        write_to_file(buffer);

        // etc...
    }
}
```

**Enforcement** 回收的变量要标记出。

<br/>
### **ES.27:使用std::array或stack_array处理堆栈中的数组**

**原因** 它们是可读的，不会隐式地转换为指针。它们不会与内置数组的非标准扩展相混淆。

**Example, bad**
```cpp
const int n = 7;
int m = 9;

void f()
{
    int a1[n];
    int a2[m];   // error: not ISO C++
    // ...
}
```

**注意** a1的定义是合法的C++，而且一直都是。有很多这样的代码。不过它很容易出错，特别是当长度是非局部的时候。
另外，它也是一个常见的错误来源（缓冲区溢出，数组退化的指针，等等）。a2的定义是C语言，但不是C++，被认为有安全风险。

**Example**
```cpp
const int n = 7;
int m = 9;

void f()
{
    array<int, n> a1;
    stack_array<int> a2(m);
    // ...
}
```

**Enforcement**
- 具有非恒定边界的数组要标记出。
- 具有非局部常数界限的数组要标记出。

<br/>
### **ES.28:使用lambda进行复杂的初始化，特别是const变量的初始化**

**Example, bad**
```cpp
widget x;   // should be const, but:
for (auto i = 2; i <= N; ++i) {          // this could be some
    x += some_obj.do_something_with(i);  // arbitrarily long code
}                                        // needed to initialize x
// from here, x should be const, but we can't say so in code in this style
```

**Example, good**
```cpp
const widget x = [&] {
    widget val;                                // assume that widget has a default constructor
    for (auto i = 2; i <= N; ++i) {            // this could be some
        val += some_obj.do_something_with(i);  // arbitrarily long code
    }                                          // needed to initialize x
    return val;
}();
```

**Enforcement** 很难。充其量是一种启发式方法。寻找一个未初始化的变量，然后是一个向它赋值的循环。

<br/>
### **ES.30:不要使用宏来进行程序文本操作**

**原因** 宏是错误的一个主要来源。宏不遵守通常的范围和类型规则。宏使人类看到的东西与编译器看到的东西不同。
宏使工具的建立变得复杂。

**Example, bad**
```cpp
#define Case break; case   /* BAD */
```
这个看似无害的宏使一个小写的c而不是C变成了一个糟糕的流控错误。

**注意** 这条规则并不禁止在#ifdefs中使用宏进行配置控制。
未来，模块可能会消除配置控制中对宏的需求。

**注意** 这条规则也是为了阻止使用#进行字符串化和##进行连接。像通常的宏一样，有些使用是基本无害的，但即使是这样也会给工具带来问题，
例如自动完成器、静态分析器和调试器。通常情况下，想要使用花哨的宏是设计过于复杂的标志。另外，#和##鼓励定义和使用宏。
```cpp
#define CAT(a, b) a ## b
#define STRINGIFY(a) #a

void f(int x, int y)
{
    string CAT(x, y) = "asdf";   // BAD: hard for tools to handle (and ugly)
    string sx2 = STRINGIFY(x);
    // ...
}
```
对于使用宏进行low-level的字符串操作，有一些变通方法。比如说：
```cpp
string s = "asdf" "lkjh";   // ordinary string literal concatenation

enum E { a, b };

template<int x>
constexpr const char* stringify()
{
    switch (x) {
    case a: return "a";
    case b: return "b";
    }
}

void f(int x, int y)
{
    string sx = stringify<x>();
    // ...
}
```
这不像宏那样方便定义，但同样容易使用，开销为零，而且是类型化和范围化的。
未来，静态反射可能会消除对程序文本操作的预处理器的最后需求。

**Enforcement** 当你看到不只是用于源码控制的宏时要注意。

<br/>
### **ES.31:不要将宏用于常量或函数**

**Example, bad**
```cpp
#define PI 3.14
#define SQUARE(a, b) (a * b)
```
即使我们没有在SQUARE中出错，也有表现得更好的替代品；例如:
```cpp
constexpr double pi = 3.14;
template<typename T> T square(T a, T b) { return a * b; }
```

**Enforcement** 当你看到不只是用于源码控制的宏时要注意。

<br/>
### **ES.32:对所有的宏名称使用ALL_CAPS**

**原因** 传统，可读性，区分宏。

**Example**
```cpp
#define forever for (;;)   /* very BAD */

#define FOREVER for (;;)   /* Still evil, but at least visible to humans */
```

<br/>
### **ES.33:如果你必须使用宏，请给它们一个唯一名字**

**原因** 宏不遵守范围规则。

**Example**
```cpp
#define MYCHAR        /* BAD, will eventually clash with someone else's MYCHAR*/

#define ZCORP_CHAR    /* Still evil, but less likely to clash */
```

<br/>
### **ES.34:要定义一个（C风格的）variadic函数**

**原因** 不是类型安全的。需要乱七八糟的、带有宏的代码才能正确工作。

**Example**
```cpp
#include <cstdarg>

// "severity" followed by a zero-terminated list of char*s; write the C-style strings to cerr
void error(int severity ...)
{
    va_list ap;             // a magic type for holding arguments
    va_start(ap, severity); // arg startup: "severity" is the first argument of error()

    for (;;) {
        // treat the next var as a char*; no checking: a cast in disguise
        char* p = va_arg(ap, char*);
        if (!p) break;
        cerr << p << ' ';
    }

    va_end(ap);             // arg cleanup (don't forget this)

    cerr << '\n';
    if (severity) exit(severity);
}

void use()
{
    error(7, "this", "is", "an", "error", nullptr);
    error(7); // crash
    error(7, "this", "is", "an", "error");  // crash
    const char* is = "is";
    string an = "an";
    error(7, "this", "is", an, "error"); // crash
}
```

**替代方案** Overloading. Templates. Variadic templates.
```cpp
#include <iostream>

void error(int severity)
{
    std::cerr << '\n';
    std::exit(severity);
}

template<typename T, typename... Ts>
constexpr void error(int severity, T head, Ts... tail)
{
    std::cerr << head;
    error(severity, tail...);
}

void use()
{
    error(7); // No crash!
    error(5, "this", "is", "not", "an", "error"); // No crash!

    std::string an = "an";
    error(7, "this", "is", "not", an, "error"); // No crash!

    error(5, "oh", "no", nullptr); // Compile error! No need for nullptr.
}
```

**注意** 这基本上就是printf的实现方式。

**Enforcement**
- C风格的variadic functions要标记出。
- #include \<cstdarg\>和#include \<stdarg.h\>要标记出。

<br/>
## **ES.expr:表达式**
表达式操作数值。

<br/>
### **ES.40:避免复杂的表达式**

**原因** 复杂的表达方式很容易出错。

**Example**
```cpp
// bad: assignment hidden in subexpression
while ((c = getc()) != -1)

// bad: two non-local variables assigned in sub-expressions
while ((cin >> c1, cin >> c2), c1 == c2)

// better, but possibly still too complicated
for (char c1, c2; cin >> c1 >> c2 && c1 == c2;)

// OK: if i and j are not aliased
int x = ++i + ++j;

// OK: if i != j and i != k
v[i] = v[j] + v[k];

// bad: multiple assignments "hidden" in subexpressions
x = a + (b = f()) + (c = g()) * 7;

// bad: relies on commonly misunderstood precedence rules
x = a & b + c * d && e ^ f == 7;

// bad: undefined behavior
x = x++ + x++ + ++x;
```
其中一些表达式绝对是错误的（例如，它们依赖于未定义的行为）。其他的非常复杂和/或不寻常，即使是优秀的程序员也可能在匆忙时误解它们或忽略问题。

**注意** C++17 收紧了求值顺序的规则（从左到右，除了赋值中的从右到左，函数参数的求值顺序未指定；参见 ES.43），但事实并非如此改变复杂表达式可能令人困惑的事实。

**注意** 一个程序员应该知道并使用表达式的基本规则。

**Example**
```cpp
x = k * y + z;             // OK

auto t1 = k * y;           // bad: unnecessarily verbose
x = t1 + z;

if (0 <= x && x < max)   // OK

auto t1 = 0 <= x;        // bad: unnecessarily verbose
auto t2 = x < max;
if (t1 && t2)            // ...
```

<br/>
### **ES.41:如果对运算符的优先级有疑问，可以用括号表示**

**原因** 避免错误。可读性。不是每个人都能记住操作符表的。

**Example**
```cpp
const unsigned int flag = 2;
unsigned int a = flag;

if (a & flag != 0)  // bad: means a&(flag != 0)
```
注意：我们建议程序员了解算术运算、逻辑运算的优先级表，但考虑将按位逻辑运算与其他需要括号的运算符混合使用。
```cpp
if ((a & flag) != 0)  // OK: works as intended
```

**注意** 你应该知道，有些不需要括号的。
```cpp
if (a < 0 || a <= max) {
    // ...
}
```

**Enforcement** 
- 按位逻辑运算符和其他运算符的组合要标记出。
- 赋值运算符不是最左边的运算符要标记出。

<br/>
### **ES.42:保持对指针的使用简单明了**

**原因** 复杂的指针操作是错误的一个主要来源。

**注意** 请改用gsl::span。指针应该只引用单个对象。指针算法脆弱且容易出错，是许多错误和安全违规的根源。
span是一种边界检查的安全类型，用于访问数据数组。使用常数作为下标访问一个已知边界的数组，可以被编译器验证。

**Example, bad**
```cpp
void f(int* p, int count)
{
    if (count < 2) return;

    int* q = p + 1;    // BAD

    ptrdiff_t d;
    int n;
    d = (p - &n);      // OK
    d = (q - p);       // OK

    int n = *p++;      // BAD

    if (count < 6) return;

    p[4] = 1;          // BAD

    p[count - 1] = 2;  // BAD

    use(&p[0], 3);     // BAD
}
```

**Example, good**
```cpp
void f(span<int> a) // BETTER: use span in the function declaration
{
    if (a.size() < 2) return;

    int n = a[0];      // OK

    span<int> q = a.subspan(1); // OK

    if (a.size() < 6) return;

    a[4] = 1;          // OK

    a[a.size() - 1] = 2;  // OK

    use(a.data(), 3);  // OK
}
```

**注意** span是一个运行时边界检查的安全类型，用于访问数据的数组。如果需要迭代器来访问一个数组，可以使用在数组上构建的span的迭代器。

**Example, bad**
```cpp
void f(array<int, 10> a, int pos)
{
    a[pos / 2] = 1; // BAD
    a[pos - 1] = 2; // BAD
    a[-1] = 3;    // BAD (but easily caught by tools) -- no replacement, just don't do this
    a[10] = 4;    // BAD (but easily caught by tools) -- no replacement, just don't do this
}
```

**Example, good** Use a span:
```cpp
void f1(span<int, 10> a, int pos) // A1: Change parameter type to use span
{
    a[pos / 2] = 1; // OK
    a[pos - 1] = 2; // OK
}

void f2(array<int, 10> arr, int pos) // A2: Add local span and use that
{
    span<int> a = {arr.data(), pos};
    a[pos / 2] = 1; // OK
    a[pos - 1] = 2; // OK
}
```
Use at():
```cpp
void f3(array<int, 10> a, int pos) // ALTERNATIVE B: Use at() for access
{
    at(a, pos / 2) = 1; // OK
    at(a, pos - 1) = 2; // OK
}
```

**Example, bad**
```cpp
void f()
{
    int arr[COUNT];
    for (int i = 0; i < COUNT; ++i)
        arr[i] = i; // BAD, cannot use non-constant indexer
}
```

**Example, good** Use a span:
```cpp
void f1()
{
    int arr[COUNT];
    span<int> av = arr;
    for (int i = 0; i < COUNT; ++i)
        av[i] = i;
}
```
Use a span and range-for:
```cpp
void f1a()
{
     int arr[COUNT];
     span<int, COUNT> av = arr;
     int i = 0;
     for (auto& e : av)
         e = i++;
}
```
Use at() for access:
```cpp
void f2()
{
    int arr[COUNT];
    for (int i = 0; i < COUNT; ++i)
        at(arr, i) = i;
}
```
Use a range-for:
```cpp
void f3()
{
    int arr[COUNT];
    int i = 0;
    for (auto& e : arr)
         e = i++;
}
```

**注意** 工具可以提供对涉及动态索引表达式的数组访问的重写，可以改为使用at()：
```cpp
static int a[10];

void f(int i, int j)
{
    a[i + j] = 12;      // BAD, could be rewritten as ...
    at(a, i + j) = 12;  // OK -- bounds-checked
}
```

**Example** 把一个数组变成一个指针（因为语言基本上总是这样做），就失去了检查的机会，所以要避免这样做。
```cpp
void g(int* p);

void f()
{
    int a[5];
    g(a);        // BAD: are we trying to pass an array?
    g(&a[0]);    // OK: passing one object
}
```
如果你想传递一个数组：
```cpp
void g(int* p, size_t length);  // old (dangerous) code

void g1(span<int> av); // BETTER: get g() changed.

void f2()
{
    int a[5];
    span<int> av = a;

    g(av.data(), av.size());   // OK, if you have no choice
    g1(a);                     // OK -- no decay here, instead use implicit span ctor
}
```

**Enforcement**
- 指针类型的表达式上的任何算术运算，其结果是指针类型的值要标记出。
- 数组类型（静态数组或std::array）的表达式或变量的任何索引表达式，其中索引不是值介于0和数组上限之间的编译时常量表达式都要标记出。
- 任何依赖于数组类型向指针类型的隐式转换的表达式都要标记出。

<br/>
### **ES.43:避免使用未定义评估顺序的表达式**

**原因** 你根本不知道这样的代码是干什么的。可移植性。即使它为你做了一些合理的事情，
它在另一个编译器（例如，你的编译器的下一个版本）或不同的优化器设置下可能会做一些不同的事情。

**注意** C++17收紧了评估顺序的规则：从左到右，除了赋值中的从右到左，函数参数的评估顺序是未指定的。
但是，请记住，您的代码可能是使用C++17之前的编译器编译的（例如，通过剪切和粘贴），所以不要太聪明。

**Example**
```cpp
v[i] = ++i;   //  the result is undefined
```
一个好的经验法则是，在一个表达式中，你不应该在写给它的地方读两次值。

**Enforcement** 可以通过一个好的分析工具检测出来。

<br/>
### **ES.44:不依赖函数参数的评估顺序**

**原因** 因为这个顺序是不明确的。

**注意** C++17收紧了求值顺序的规则，但函数参数的求值顺序仍然没有规定。

**Example**
```cpp
int i = 0;
f(++i, ++i);
```
在C++17之前，这个行为是未定义的，所以这个行为可能是任何东西（例如，f(2, 2)）。自C++17以来，这段代码没有未定义的行为，
但仍然没有指定哪个参数先被评估。调用将是f(1, 2)或f(2, 1)，但你不知道哪个。

**Example** 重载过的运算符会导致评估顺序的问题。
```cpp
f1()->m(f2());          // m(f1(), f2())
cout << f1() << f2();   // operator<<(operator<<(cout, f1()), f2())
```
在C++17中，这些示例按预期工作（从左到右）并且赋值从右到左计算（就像=的绑定是从右到左）。
```cpp
f1() = f2();    // undefined behavior in C++14; in C++17, f2() is evaluated before f1()
```

**Enforcement** 可以通过一个好的分析工具检测出。

<br/>
### **ES.45:避免magic constants，使用符号常量**

**原因** 嵌入在表达式中的未命名常量很容易被忽视并且通常难以理解。

**Example**
```cpp
for (int m = 1; m <= 12; ++m)   // don't: magic constant 12
    cout << month[m] << '\n';
```
以下更好：
```cpp
// months are indexed 1..12
constexpr int first_month = 1;
constexpr int last_month = 12;

for (int m = first_month; m <= last_month; ++m)   // better
    cout << month[m] << '\n';
```
更好的是，不要暴露常数。
```cpp
for (auto m : month)
    cout << m << '\n';
```

**Enforcement** 代码中的字面量要标记出。0、1、nullptr、\n、""这些可以略过。

<br/>
### **ES.46:避免缩小转换范围**

**原因** 缩小的转换会破坏信息，而且往往是意想不到的破坏。

**Example, bad**
```cpp
double d = 7.9;
int i = d;    // bad: narrowing: i becomes 7
i = (int) d;  // bad: we're going to claim this is still not explicit enough

void f(int x, long y, double d)
{
    char c1 = x;   // bad: narrowing
    char c2 = y;   // bad: narrowing
    char c3 = d;   // bad: narrowing
}
```
准则支持库提供了一个narrow\_cast操作，用于指定缩小是可以接受的，还有一个narrow("narrow if")，如果缩小会丢掉合法的值，则抛出一个异常。 
```cpp
i = gsl::narrow_cast<int>(d);   // OK (you asked for it): narrowing: i becomes 7
i = gsl::narrow<int>(d);        // OK: throws narrowing_error
```
我们还包括有损算术转换，例如从负浮点类型转换为无符号整数类型：
```cpp
double d = -7.9;
unsigned u = 0;

u = d;                               // bad: narrowing
u = gsl::narrow_cast<unsigned>(d);   // OK (you asked for it): u becomes 4294967289
u = gsl::narrow<unsigned>(d);        // OK: throws narrowing_error
```

**注意** 这一规则不适用于上下文转换为bool。

**Enforcement**
- 所有浮点到整数的转换（也许只有float-\>char和double-\>int）要标记出。
- 所有long-\>char（我怀疑int-\>char）很常见要标记出。
- 考虑缩小函数参数的转换范围，尤其值得怀疑。

<br/>
### **ES.47:使用nullptr而不是0或NULL**

**原因** 可读性。尽量减少意外：nullptr不能与int混淆。nullptr也有一个明确的（非常限制性的）类型，因此在更多的情况下，类型推理可能在NULL或0上做错事。

**Example**
```cpp
void f(int);
void f(char*);
f(0);         // call f(int)
f(nullptr);   // call f(char*)
```

**Enforcement** 对指针使用0和NULL的要标志出。

<br/>
### **ES.48:避免cast**

**原因** cast是一个众所周知的错误来源，并使一些优化不可靠。

**Example, bad**
```cpp
double d = 2;
auto p = (long*)&d;
auto q = (long long*)&d;
cout << d << ' ' << *p << ' ' << *q << '\n';
```
你认为这个代码片段会打印出什么？其结果充其量是定义的实现。
```cpp
2 0 4611686018427387904
```
增加代码：
```cpp
*q = 666;
cout << d << ' ' << *p << ' ' << *q << '\n';
```
得到：
```cpp
3.29048e-321 666 666
```

**注意** 编写强制转换的程序员通常假设他们知道自己在做什么，或者编写强制转换会使程序“更易于阅读”。事实上，他们经常禁用使用值的一般规则。
如果有合适的函数可供选择，重载解析和模板实例化通常会选择合适的函数。如果没有，也许应该有，而不是应用本地修复（强制转换）。

**注意** 强制转换在系统编程语言中是必需的。例如，我们如何将设备寄存器的地址放入指针中？然而，转换被严重过度使用，并且是错误的主要来源。
如果您觉得需要大量转换，则可能存在基本设计问题。

[type profile](#typeprofile)禁止reinterpret\_cast和C风格转换。

永远不要转换为(void)来忽略[[nodiscard]]返回值。如果你故意想要丢弃这样的结果，首先要认真考虑这是否真的是一个好主意
（通常有一个很好的理由，函数或返回类型的作者首先使用[[nodiscard]]。如果您仍然认为它是合适的并且您的代码审查者也同意，
请使用std::ignore = 关闭警告，该警告简单、可移植且易于grep。

**替代方案** 强制转换被广泛（误）使用。现代C++的规则和构造消除了在许多上下文中对强制转换的需要，例如:
- Use templates
- Use std::variant
- Rely on the well-defined, safe, implicit conversions between pointer types
- Use std::ignore = to ignore [[nodiscard]] values.

**Enforcement**
- 所有C类型转换要标记出，包括转换为void。
- 使用Type(value)进行函数式转换要标记出。使用Type{value}来代替，这不是转换。
- 指针类型之间的转换，其中源类型和目标类型是相同的要标记出。
- 本应该是隐式转换的，却用显式指针转换要标记出。。

<br/>
### **ES.49:如果必须使用cast，使用一个命名的cast**

**原因** 可读性。避免错误。命名转换比C语言或函数式转换更具体，允许编译器捕捉一些错误。

- static\_cast
- const\_cast
- reinterpret\_cast
- dynamic\_cast
- std::move // move(x) is an rvalue reference to x
- std::forward // forward\<T\>(x) is an rvalue or an lvalue reference to x depending on T
- gsl::narrow\_cast // narrow\_cast\<T\>(x) is static\_cast\<T\>(x)
- gsl::narrow // narrow\<T\>(x) is static\_cast\<T\>(x) if static\_cast\<T\>(x) == x or it throws narrowing\_error

**Example**
```cpp
class B { /* ... */ };
class D { /* ... */ };

template<typename D> D* upcast(B* pb)
{
    D* pd0 = pb;                        // error: no implicit conversion from B* to D*
    D* pd1 = (D*)pb;                    // legal, but what is done?
    D* pd2 = static_cast<D*>(pb);       // error: D is not derived from B
    D* pd3 = reinterpret_cast<D*>(pb);  // OK: on your head be it!
    D* pd4 = dynamic_cast<D*>(pb);      // OK: return nullptr
    // ...
}
```
该示例是从现实世界的错误中合成的，其中D曾经从B派生，但有人重构了层次结构。C风格的转换是危险的，因为它可以进行任何类型的转换，使我们无法避免错误（现在或将来）。

**注意** 在没有信息丢失的类型之间进行转换时（例如，从float到double或从int32到int64），可以改用大括号初始化。
```cpp
double d {some_float};
int64_t i {some_int32};
```
这使得类型转换的目的很明确，同时也防止了可能导致精度损失的类型间的转换。(例如，试图以这种方式从一个double初始化一个float是一个编译错误）。

**注意** reinterpret\_cast可能是必不可少的，但必不可少的使用（例如，将机器地址变成指针）并不是类型安全的。
```cpp
auto p = reinterpret_cast<Device_register>(0x800);  // inherently dangerous
```

**Enforcement**
- 所有C风格的转换都要标记出，包括转为void。
- 使用Type(value)进行函数式转换要标记出，使用Type{value}来代替，这不是narrowing。
- The type profile bans reinterpret\_cast.
- The type profile warns when using static\_cast between arithmetic types.

<br/>
### **ES.50:不要对const对象使用cast**

**原因** 它对const做了一个谎言。如果该变量实际上被声明为const，修改它就会导致未定义的行为。

**Example, bad**
```cpp
void f(const int& x)
{
    const_cast<int&>(x) = 42;   // BAD
}

static int i = 0;
static const int j = 0;

f(i); // silent side effect
f(j); // undefined behavior
```

**Example** 有时，你可能会想使用const\_cast来避免代码的重复，例如，当两个仅在常量上有差异的访问器函数有类似的实现时。比如说:
```cpp
class Bar;

class Foo {
public:
    // BAD, duplicates logic
    Bar& get_bar()
    {
        /* complex logic around getting a non-const reference to my_bar */
    }

    const Bar& get_bar() const
    {
        /* same complex logic around getting a const reference to my_bar */
    }
private:
    Bar my_bar;
};
```
相反，更愿意共享实现。通常，您可以让非const函数调用const函数。然而，当存在复杂的逻辑时，这可能会导致以下仍采用const\_cast的模式。
```cpp
class Foo {
public:
    // not great, non-const calls const version but resorts to const_cast
    Bar& get_bar()
    {
        return const_cast<Bar&>(static_cast<const Foo&>(*this).get_bar());
    }
    const Bar& get_bar() const
    {
        /* the complex logic around getting a const reference to my_bar */
    }
private:
    Bar my_bar;
};
```
虽然这种模式在正确应用时是安全的，因为调用者必须有一个非常量对象开始，但它并不理想，因为安全性很难作为检查规则自动执行。
相反，更倾向于把普通的代码放在一个普通的辅助函数中--并把它变成一个模板，这样它就可以推导出const。这根本就没有使用任何const\_cast。
```cpp
class Foo {
public:                         // good
          Bar& get_bar()       { return get_bar_impl(*this); }
    const Bar& get_bar() const { return get_bar_impl(*this); }
private:
    Bar my_bar;

    template<class T>           // good, deduces whether T is const or non-const
    static auto& get_bar_impl(T& t)
        { /* the complex logic around getting a possibly-const reference to my_bar */ }
};
```

注意：不要在模板内做大量的非依赖性工作，这会导致代码膨胀。例如，进一步的改进是，如果get\_bar\_impl的全部或部分可以是非依赖的，
并分解为一个通用的非模板函数，则可能会大大减少代码大小。

**例外**
```cpp
int compute(int x); // compute a value for x; assume this to be costly

class Cache {   // some type implementing a cache for an int->int operation
public:
    pair<bool, int> find(int x) const;   // is there a value for x?
    void set(int x, int v);             // make y the value for x
    // ...
private:
    // ...
};

class X {
public:
    int get_val(int x)
    {
        auto p = cache.find(x);
        if (p.first) return p.second;
        int val = compute(x);
        cache.set(x, val); // insert value for x
        return val;
    }
    // ...
private:
    Cache cache;
};
```
在这里，get\_val()在逻辑上是常量，所以我们想把它变成一个常量成员。为了做到这一点，我们仍然需要修改缓存，所以人们有时会采用const\_cast的方式。
```cpp
class X {   // Suspicious solution based on casting
public:
    int get_val(int x) const
    {
        auto p = cache.find(x);
        if (p.first) return p.second;
        int val = compute(x);
        const_cast<Cache&>(cache).set(x, val);   // ugly
        return val;
    }
    // ...
private:
    Cache cache;
};
```
幸运的是，有一个更好的解决方案。说明缓存是可变的，即使对于一个常量对象。
```cpp
class X {   // better solution
public:
    int get_val(int x) const
    {
        auto p = cache.find(x);
        if (p.first) return p.second;
        int val = compute(x);
        cache.set(x, val);
        return val;
    }
    // ...
private:
    mutable Cache cache;
};
```
另一个解决方案是存储一个指向cache的指针。
```cpp
class X {   // OK, but slightly messier solution
public:
    int get_val(int x) const
    {
        auto p = cache->find(x);
        if (p.first) return p.second;
        int val = compute(x);
        cache->set(x, val);
        return val;
    }
    // ...
private:
    unique_ptr<Cache> cache;
};
```
这个解决方案是最灵活的，但需要明确地构建和销毁\*cache（很可能在X的构造函数和析构函数中）。

在任何变体中，我们都必须防止多线程代码中缓存的data race，可能使用std::mutex。

**Enforcement**
- const\_cast都要标记出。

<br/>
### **ES.55:避免范围检查的需要**

**原因** 不能溢出的构造不会溢出（通常运行得更快）。

**Example**
```cpp
for (auto& x : v)      // print all elements of v
    cout << x << '\n';

auto p = find(v, x);   // find x in v
```

**Enforcement** 寻找明确的范围检查并启发式地建议替代方案。

<br/>
### **ES.56:只有当你需要明确地将一个对象移动到另一个范围时，才写std::move()**

**原因** 我们移动，而不是复制，是为了避免重复和提高性能。
移动通常会留下一个空对象（C.64），这可能是令人惊讶的，甚至是危险的，所以我们尽量避免从l值中移动（它们以后可能被访问）。

**注意** 当源是右值（例如，返回处理中的值或函数结果）时，移动是隐式完成的，因此在这些情况下，不要通过显式编写移动来毫无意义地使代码复杂化。
相反，编写返回值的短函数，函数的返回和调用者对返回的接受都会自然优化。

一般来说，遵循本文档中的指导原则（包括不使变量的作用域过大，编写返回值的短函数，返回局部变量）有助于消除对明确的std::move的大部分需要。

显式移动是需要的，以便显式地将一个对象移动到另一个作用域，特别是将其传递给一个sink函数，以及在移动操作本身的实现中（移动构造函数、移动赋值运算符）和交换操作。

**Example, bad**
```cpp
void sink(X&& x);   // sink takes ownership of x

void user()
{
    X x;
    // error: cannot bind an lvalue to a rvalue reference
    sink(x);
    // OK: sink takes the contents of x, x must now be assumed to be empty
    sink(std::move(x));

    // ...

    // probably a mistake
    use(x);
}
```
通常，std::move()是作为&&参数的一个参数使用的。而在你这样做之后，假设该对象已经被移出（见C.64），在你第一次将其设置为新值之前，不要再读取其状态。
```cpp
void f()
{
    string s1 = "supercalifragilisticexpialidocious";

    string s2 = s1;             // ok, takes a copy
    assert(s1 == "supercalifragilisticexpialidocious");  // ok

    // bad, if you want to keep using s1's value
    string s3 = move(s1);

    // bad, assert will likely fail, s1 likely changed
    assert(s1 == "supercalifragilisticexpialidocious");
}
```

**Example**
```cpp
void sink(unique_ptr<widget> p);  // pass ownership of p to sink()

void f()
{
    auto w = make_unique<widget>();
    // ...
    sink(std::move(w));               // ok, give to sink()
    // ...
    sink(w);    // Error: unique_ptr is carefully designed so that you cannot copy it
}
```

**注意** std::move()是对&&的伪装；它本身并不移动任何东西，而是将一个命名的对象标记为可以被移动的候选对象。
语言已经知道对象可以被移出的常见情况，特别是当从函数返回值时，所以不要用多余的std::move()来使代码复杂化。

千万不要因为听说 "它更有效率 "就写std::move()。一般来说，不要相信没有数据的 "效率 "的说法(???)。
一般来说，不要无缘无故地将你的代码复杂化(??)。永远不要在一个常量对象上写std::move()，它会被默默地转化为一个拷贝（见Meyers15中的第23条）。

**Example, bad**
```cpp
vector<int> make_vector()
{
    vector<int> result;
    // ... load result with data
    return std::move(result);       // bad; just write "return result;"
}
```
千万不要写return move(local\_variable);，因为语言已经知道这个变量是一个移动候选变量。在这段代码中写move不会有任何帮助，
实际上可能是有害的，因为在一些编译器中，它通过为本地变量创建一个额外的引用别名而干扰了RVO（返回值优化）。

**Example, bad**
```cpp
vector<int> v = std::move(make_vector());   // bad; the std::move is entirely redundant
```
永远不要在一个返回值上写move，比如x = move(f()); 其中f返回值。语言已经知道一个返回值是一个可以被移动的临时对象。

**Example**
```cpp
void mover(X&& x)
{
    call_something(std::move(x));         // ok
    call_something(std::forward<X>(x));   // bad, don't std::forward an rvalue reference
    call_something(x);                    // suspicious, why not std::move?
}

template<class T>
void forwarder(T&& t)
{
    call_something(std::move(t));         // bad, don't std::move a forwarding reference
    call_something(std::forward<T>(t));   // ok
    call_something(t);                    // suspicious, why not std::forward?
}
```

**Enforcement**
- 使用std::move(x)其中x是右值或语言已经将其视为右值，包括return std::move(local\_variable);和std::move(f())在按值返回的函数上，要标记出。
- 如果没有const S&重载来处理左值，而采用S&&参数的函数要标记出。
- 标记一个传递给参数的std::move参数，除非参数类型是X&& rvalue引用，或者类型是move-only，并且参数是通过值传递。
- 当std::move应用于转发引用（T&&，其中T是模板参数类型）时标志出。使用std::forward代替。
- 当std::move被应用于非const的rvalue引用之外的其他情况时要标记出。 (之前规则的更一般的情况，涵盖了非转发的情况。)
- 当std::forward被应用于rvalue引用（X&&，其中X是一个非模板参数类型）时要标记出。使用std::move代替。
- 当std::forward被应用于除转发引用以外的其他情况时的标志。(前面规则的更一般的情况，以涵盖非移动的情况)。
- 当一个对象有可能被移出，并且下一个操作是一个常量操作时，标志着；首先应该有一个中间的非常量操作，最好是赋值，以首先重置对象的值。

<br/>
### **ES.60:资源管理函数外避免new和delete**

**原因** 在应用程序代码中直接管理资源是很容易出错的，而且很繁琐。

**注意** This is also known as the rule of “No naked new!”

**Example, bad**
```cpp
void f(int n)
{
    auto p = new X[n];   // n default constructed Xs
    // ...
    delete[] p;
}
```
在...部分可能有代码，导致删除永远不会发生。

<br/>
### **ES.61:数组用delete[]，其他的用delete**

**原因** 这就是语言所要求的，错误可能导致资源释放错误和/或内存损坏。

**Example, bad**
```cpp
void f(int n)
{
    auto p = new X[n];   // n default constructed Xs
    // ...
    delete p;   // error: just delete the object p, rather than delete the array p[]
}
```

**注意** 这个例子不仅违反了前面例子中的no naked new规则，它还有更多的问题。

**Enforcement**
- 如果新建和删除在同一范围内，可以标记出错误。
- 如果new和delete是在一个构造函数/析构函数对中，可以标记出错误。

<br/>
### **ES.62:不要比较指向不同数组的指针**

**原因** 这样做的结果是未定义的。

**Example, bad**
```cpp
void f()
{
    int a1[7];
    int a2[9];
    if (&a1[5] < &a2[7]) {}       // bad: undefined
    if (0 < &a1[5] - &a2[7]) {}   // bad: undefined
}
```

<br/>
### **ES.63:不要切片**

**原因** 切片--即使用赋值或初始化只复制对象的一部分--最常导致错误，因为对象本应作为一个整体考虑。在极少数情况下，如果切片是故意的，那么代码就会令人吃惊。

**Example**
```cpp
class Shape { /* ... */ };
class Circle : public Shape { /* ... */ Point c; int r; };

Circle c { {0, 0}, 42};
Shape s {c};    // copy construct only the Shape part of Circle
s = c;          // or copy assign only the Shape part of Circle

void assign(const Shape& src, Shape& dest)
{
    dest = src;
}
Circle c2 { {1, 1}, 43};
assign(c, c2);   // oops, not the whole state is transferred
assert(c == c2); // if we supply copying, we should also provide comparison,
                 // but this will likely return false
```
结果将是毫无意义的，因为中心和半径不会从c复制到s。
防止这种情况的第一道防线是定义基类Shape不允许这样做。

**替代方案** 如果您打算切片，请定义一个显式操作来执行此操作。这使读者免于困惑。例如:
```cpp
class Smiley : public Circle {
    public:
    Circle copy_circle();
    // ...
};

Smiley sm { /* ... */ };
Circle c1 {sm};  // ideally prevented by the definition of Circle
Circle c2 {sm.copy_circle()};
```

**Enforcement** 切片要发出警告。

<br/>
### **ES.64:使用T{e}表示法进行构造**

**原因** T{e} 构造语法明确指出需要进行构造。T{e}构造语法不允许缩小范围。T{e}是唯一安全和通用的表达式，用于从表达式e构造T类型的值。

**Example** 对于内置类型，构造符号可防止缩小和重新解释。
```cpp
void use(char ch, int i, double d, char* p, long long lng)
{
    int x1 = int{ch};     // OK, but redundant
    int x2 = int{d};      // error: double->int narrowing; use a cast if you need to
    int x3 = int{p};      // error: pointer to->int; use a reinterpret_cast if you really need to
    int x4 = int{lng};    // error: long long->int narrowing; use a cast if you need to

    int y1 = int(ch);     // OK, but redundant
    int y2 = int(d);      // bad: double->int narrowing; use a cast if you need to
    int y3 = int(p);      // bad: pointer to->int; use a reinterpret_cast if you really need to
    int y4 = int(lng);    // bad: long long->int narrowing; use a cast if you need to

    int z1 = (int)ch;     // OK, but redundant
    int z2 = (int)d;      // bad: double->int narrowing; use a cast if you need to
    int z3 = (int)p;      // bad: pointer to->int; use a reinterpret_cast if you really need to
    int z4 = (int)lng;    // bad: long long->int narrowing; use a cast if you need to
}
```
整数到/从指针转换是在使用T(e)或(T)e符号时定义的实现，并且在具有不同整数和指针大小的平台之间不可移植。

**注意** 避免cast（显式类型转换），如果必须引用，最好是named cast。

**注意** 在不明确的情况下，T{e}中可以不加T。
```cpp
complex<double> f(complex<double>);

auto z = f({2*pi, 1});
```

**注意** 构造符号是最通用的初始化符号。

**例外** std::vector和其他容器是在我们将{}作为构造符号之前定义的。
```cpp
vector<string> vs {10};                           // ten empty strings
vector<int> vi1 {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};  // ten elements 1..10
vector<int> vi2 {10};                             // one element with the value 10
```
我们如何获得一个由10个默认初始化的int组成的vector？
```cpp
vector<int> v3(10); // ten elements with value 0
```
使用()而不是{}表示元素数量是传统做法（可追溯到1980年代初期），很难更改，但仍然是一个设计错误：对于元素类型可能与元素数量混淆的容器，
我们有一个必须解决的歧义。常规解析是将{10}解释为一个元素的列表，并使用（10）来区分大小。

这个错误不需要在新的代码中重复。我们可以定义一个类型来表示元素的数量。
```cpp
struct Count { int n; };

template<typename T>
class Vector {
public:
    Vector(Count n);                     // n default-initialized elements
    Vector(initializer_list<T> init);    // init.size() elements
    // ...
};

Vector<int> v1{10};
Vector<int> v2{Count{10}};
Vector<Count> v3{Count{10}};    // yes, there is still a very minor problem
```
剩下的主要问题是为Count找到一个合适的名字。

**Enforcement** C风格的(T)e和函数风格的T(e)转换要标记出。

<br/>
### **ES.65:不要对无效指针解引用**

**原因** 解引用无效的指针，如nullptr，是未定义的行为，通常会导致立即崩溃，错误的结果，或内存损坏。

**注意** 这条规则是一个明显的、众所周知的语言规则，但可能很难遵守。它需要良好的编码风格、库支持和静态分析来消除违反规则的情况，
而不需要很大的开销。这是讨论C++的类型和资源安全模型的一个主要部分。

**Example**
```cpp
void f()
{
    int x = 0;
    int* p = &x;

    if (condition()) {
        int y = 0;
        p = &y;
    } // invalidates p

    *p = 42;            // BAD, p might be invalid if the branch was taken
}
```
为了解决这个问题，要么延长指针所指向的对象的寿命，要么缩短指针的寿命（将解引用移到被指向的对象的寿命结束之前）。
```cpp
void f1()
{
    int x = 0;
    int* p = &x;

    int y = 0;
    if (condition()) {
        p = &y;
    }

    *p = 42;            // OK, p points to x or y and both are still in scope
}
```
不幸的是，大多数无效指针问题更难发现，也更难解决。

**Example**
```cpp
void f(int* p)
{
    int x = *p; // BAD: how do we know that p is valid?
}
```
有大量的此类代码。大多数工作--经过大量的测试--但在孤立的情况下，不可能知道p是否可能是nullptr。因此，这也是一个主要的错误来源。有很多方法来处理这个潜在的问题。
```cpp
void f1(int* p) // deal with nullptr
{
    if (!p) {
        // deal with nullptr (allocate, return, throw, make p point to something, whatever
    }
    int x = *p;
}
```
对nullptr的测试有两个潜在问题:
- 如果我们发现nullptr，该怎么做并不总是很明确。
- 该测试可能是多余的和/或相对昂贵的。
- 不明显的是，该测试是为了保护违反规定的逻辑或部分规定的逻辑。
```cpp
void f2(int* p) // state that p is not supposed to be nullptr
{
    assert(p);
    int x = *p;
}
```
只有当断言检查被启用时，这才会有成本，并且会给编译器/分析器提供有用的信息。如果/当C++直接支持契约时，这将会更加有效。
```cpp
void f3(int* p) // state that p is not supposed to be nullptr
    [[expects: p]]
{
    int x = *p;
}
```
另外，我们可以使用gsl::not\_null来确保p不是nullptr。
```cpp
void f(not_null<int*> p)
{
    int x = *p;
}
```
这些补救措施只照顾到了nullptr。请记住，还有其他获得无效指针的方法。

**Example**
```cpp
void f(int* p)  // old code, doesn't use owner
{
    delete p;
}

void g()        // old code: uses naked new
{
    auto q = new int{7};
    f(q);
    int x = *q; // BAD: dereferences invalid pointer
}
```
**Example**
```cpp
void f()
{
    vector<int> v(10);
    int* p = &v[5];
    v.push_back(99); // could reallocate v's elements
    int x = *p; // BAD: dereferences potentially invalid pointer
}
```

<br/>
## **ES.stmt:语句**
语句控制着控制流（除了函数调用和异常抛出，它们是表达式）。


<br/>
### **ES.70:在有选择的情况下，更倾向于使用switch语句而不是if语句**

**原因**
- 可读性。
- 高效：switch与常数进行比较，通常比if-then-else链中的一系列test更优化。
- switch可以实现一些启发式的一致性检查。例如，一个enum的所有值都被覆盖了吗？如果没有，是否有一个default？ 

**Example**
```cpp
void use(int n)
{
    switch (n) {   // good
    case 0:
        // ...
        break;
    case 7:
        // ...
        break;
    default:
        // ...
        break;
    }
}
```
而不是：
```cpp
void use2(int n)
{
    if (n == 0)   // bad: if-then-else chain comparing against a set of constants
        // ...
    else if (n == 7)
        // ...
}
```

**Enforcement** 检查常量的if-then-else链（仅）要标记出。

<br/>
### **ES.71:当有选择时，优先用range-for语句而不是for语句**

**原因** 可读性。防止错误。效率。

**Example**
```cpp
for (gsl::index i = 0; i < v.size(); ++i)   // bad
    cout << v[i] << '\n';

for (auto p = v.begin(); p != v.end(); ++p)   // bad
    cout << *p << '\n';

for (auto& x : v)    // OK
    cout << x << '\n';

for (gsl::index i = 1; i < v.size(); ++i) // touches two elements: can't be a range-for
    cout << v[i] + v[i - 1] << '\n';

for (gsl::index i = 0; i < v.size(); ++i) // possible side effect: can't be a range-for
    cout << f(v, &v[i]) << '\n';

for (gsl::index i = 0; i < v.size(); ++i) { // body messes with loop variable: can't be a range-for
    if (i % 2 != 0)
        cout << v[i] << '\n'; // output odd elements
}
```
人或一个好的静态分析器可能会确定在f(v, &v[i])中确实没有对v的副作用，这样就可以重写这个循环了。

通常情况下，最好避免在一个循环的主体中 "乱用循环变量"。

**注意** 不要在range-for循环中使用expensive拷贝的循环变量。
```cpp
for (string s : vs) // ...
```
这将把vs的每个元素复制到s中。
```cpp
for (string& s : vs) // ...
```
如果循环变量没有被修改或复制，以下更好。
```cpp
for (const string& s : vs) // ...
```

**Enforcement** 看一下循环，如果一个传统的循环只是访问一个序列中的每个元素，而且它对这些元素的处理没有任何副作用，那么就把这个循环改写成一个ranged-for循环。

<br/>
### **ES.72:当有明显的循环变量时，更倾向于使用for语句而不是while语句**

**原因** 可读性：循环的完整逻辑在"up front"可见。循环变量的范围可以被限制。

**Example**
```cpp
for (gsl::index i = 0; i < vec.size(); i++) {
    // do work
}
```

**Example, bad**
```cpp
int i = 0;
while (i < vec.size()) {
    // do work
    i++;
}
```

<br/>
### **ES.73:当没有明显的循环变量时，更倾向于使用while语句而不是for语句**

**原因** 可读性。

**Example**
```cpp
int events = 0;
for (; wait_for_event(); ++events) {  // bad, confusing
    // ...
}
```
event loop误导了，因为事件计数器与循环条件（wait\_for\_event()）没有关系。
```cpp
int events = 0;
while (wait_for_event()) {      // better
    ++events;
    // ...
}
```

**Enforcement** 在for-initializers和for-increments中的动作与for条件无关的要标记出。

<br/>
### **ES.74:倾向于在for语句的初始化部分声明一个循环变量**

参见[ES.6](#es6在for-statement初始化和条件中声明名称以限制范围)

<br/>
### **ES.75:避免使用do语句**

**原因** 可读性，避免错误。终止条件在最后（可能被忽略的地方），而且第一次没有检查该条件。

**Example**
```cpp
int x;
do {
    cin >> x;
    // ...
} while (x < 0);
```

**注意** 是的，确实有这样的例子，do-statement是对解决方案的明确陈述，但也有许多错误。

**Enforcement** do-statement要标记出。

<br/>
### **ES.76:避免使用goto**

**原因** 可读性，避免错误。对于人类来说，有更好的控制结构；goto是为机器生成的代码准备的。

**例外** 突破一个嵌套的循环。在这种情况下，总是往前跳。
```cpp
for (int i = 0; i < imax; ++i)
    for (int j = 0; j < jmax; ++j) {
        if (a[i][j] > elem_max) goto finished;
        // ...
    }
finished:
// ...
```

**Example, bad** 有相当多的人使用了C语言的goto-exit idiom。
```cpp
void f()
{
    // ...
        goto exit;
    // ...
        goto exit;
    // ...
exit:
    // ... common cleanup code ...
}
```
这是对析构的一种临时模拟。用带有析构函数的句柄来声明你的资源，并进行清理。如果由于某些原因，你不能用自毁函数来处理所有的清理工作，
可以考虑用gsl::finally()作为goto exit的一个更干净、更可靠的替代方案。

**Enforcement**
- goto要标记出。更好的是所有没有从嵌套循环跳到紧接嵌套循环之后的语句的goto都要标记出。

<br/>
### **ES.77:在循环中最小化使用break和continue**

**原因** 在一个非琐碎的循环体中，很容易忽视break或continue。
循环中的break与switch-statement中的break有很大的不同（你可以在循环中使用switch-statement，在switch-case中使用循环）。

**Example**
```cpp
switch(x) {
case 1 :
    while (/* some condition */) {
        // ...
    break;
    } // Oops! break switch or break while intended?
case 2 :
    // ...
    break;
}
```

**替代方案** 通常情况下，一个需要break的循环是一个很好的函数（算法）的候选者，在这种情况下，break变成了return。
```cpp
//Original code: break inside loop
void use1()
{
    std::vector<T> vec = {/* initialized with some values */};
    T value;
    for (const T item : vec) {
        if (/* some condition*/) {
            value = item;
            break;
        }
    }
    /* then do something with value */
}

//BETTER: create a function and return inside loop
T search(const std::vector<T> &vec)
{
    for (const T &item : vec) {
        if (/* some condition*/) return item;
    }
    return T(); //default value
}

void use2()
{
    std::vector<T> vec = {/* initialized with some values */};
    T value = search(vec);
    /* then do something with value */
}
```
通常情况下，一个使用continue的循环可以用if-statement来表达，而且同样清晰。
```cpp
for (int item : vec) {  // BAD
    if (item%2 == 0) continue;
    if (item == 5) continue;
    if (item > 10) continue;
    /* do something with item */
}

for (int item : vec) {  // GOOD
    if (item%2 != 0 && item != 5 && item <= 10) {
        /* do something with item */
    }
}
```

**注意** 如果你真的需要中断一个循环，break通常比修改循环变量或goto等替代方法更好。 

<br/>
### **ES.78:不要在switch语句中依赖隐式的fallthrough**

**原因** 总是用break来结束一个非空的case。不小心漏掉一个break是一个相当常见的错误。故意的fallthrough可能是一种维护上的危险，应该是罕见的、明确的。

**Example**
```cpp
switch (eventType) {
case Information:
    update_status_bar();
    break;
case Warning:
    write_event_log();
    // Bad - implicit fallthrough
case Error:
    display_error_window();
    break;
}
```
一条语句的多个case标签是可以的。
```cpp
switch (x) {
case 'a':
case 'b':
case 'f':
    do_something(x);
    break;
}
```

**例外** 在极少数情况下，如果认为fallthrough是适当的，应明确并使用[[fallthrough]]注释。
```cpp
switch (eventType) {
case Information:
    update_status_bar();
    break;
case Warning:
    write_event_log();
    [[fallthrough]];
case Error:
    display_error_window();
    break;
}
```

**Enforcement** 所有非空case的隐式fallthrough都要标记出。
 
<br/>
### **ES.79:使用default来处理通用的情况**

**原因** 代码的清晰性。提高了错误检测的机会。

**Example**
```cpp
enum E { a, b, c, d };

void f1(E x)
{
    switch (x) {
    case a:
        do_something();
        break;
    case b:
        do_something_else();
        break;
    default:
        take_the_default_action();
        break;
    }
}
```
在这里，很明显有一个default的action，case a和b是特殊的。

**Example** 但是如果没有default，而你只想处理特定的情况呢？在这种情况下，要有一个空的default动作，否则就不可能知道你是否要处理所有的情况。
```cpp
void f2(E x)
{
    switch (x) {
    case a:
        do_something();
        break;
    case b:
        do_something_else();
        break;
    default:
        // do nothing for the rest of the cases
        break;
    }
}
```
如果你不写default，维护者和/或编译器可能会合理地认为你打算处理所有情况。
```cpp
void f2(E x)
{
    switch (x) {
    case a:
        do_something();
        break;
    case b:
    case c:
        do_something_else();
        break;
    }
}
```
你是忘记了case d还是故意不加？忘记case通常发生在一个case被添加到一个枚举器，而这样做的人没有把它添加到每个枚举的switch case中。

**Enforcement** 枚举的switch语句，不处理所有的枚举，也没有default要标记出。这可能会在某些代码库中产生太多误报；
如果是这样，就只标记处理大多数但不是所有情况的switch（这就是第一个C++编译器的策略）。

<br/>
### **ES.84:不要试图声明一个没有名字的局部变量**

**原因** 没有这样的东西。在人类看来，是一个没有名字的变量，在编译器看来，是由一个立即超出范围的临时变量组成的语句。

**Example, bad**
```cpp
void f()
{
    lock<mutex>{mx};   // Bad
    // ...
}
```
这声明了一个未命名的锁对象，在分号处立即超出了范围。这并不是一个不常见的错误。特别是，这个特殊的例子会导致难以发现的race情况。

**注意** 未命名的函数参数是可以的。 

**Enforcement** 临时的语句都要标记出来。

<br/>
### **ES.85:让空语句可见**

**原因** 可读性。

**Example**
```cpp
for (i = 0; i < max; ++i);   // BAD: the empty statement is easily overlooked
v[i] = f(v[i]);

for (auto x : v) {           // better
    // nothing
}
v[i] = f(v[i]);
```

**Enforcement** 不是block的空语句，也不包含注释要标记出。

<br/>
### **ES.86:避免在for-loop的主体内修改循环控制变量**

**原因** 前面的循环控制应该能够正确推理出循环内发生的事情。在迭代表达式和循环体内部修改循环计数器是一个常年的意外和错误的来源。

**Example**
```cpp
for (int i = 0; i < 10; ++i) {
    // no updates to i -- ok
}

for (int i = 0; i < 10; ++i) {
    //
    if (/* something */) ++i; // BAD
    //
}

bool skip = false;
for (int i = 0; i < 10; ++i) {
    if (skip) { skip = false; continue; }
    //
    if (/* something */) skip = true;  // Better: using two variables for two concepts.
    //
}
```

**Enforcement** 同时在循环控制迭代表达式和循环主体中可能被更新的变量（non-const）要标记出。

<br/>
### **ES.87:不要在条件中添加多余的==或!=**

**原因** 这样做可以避免冗长，并消除一些犯错的机会。有助于使风格一致和常规。 

**Example** 根据定义，if-语句、while-语句或for-语句中的条件是在true和false之间进行选择。一个数值与0比较，一个指针值与nullptr比较。
```cpp
// These all mean "if p is not nullptr"
if (p) { ... }            // good
if (p != 0) { ... }       // redundant !=0, bad: don't use 0 for pointers
if (p != nullptr) { ... } // redundant !=nullptr, not recommended
```
通常，if (p)被认为是"如果p有效"，这是程序员意图的直接表达，而如果(p != nullptr)是冗长的。

**Example** 当声明被用作条件时，这个规则特别有用。
```cpp
if (auto pc = dynamic_cast<Circle>(ps)) { ... } // execute if ps points to a kind of Circle, good

if (auto pc = dynamic_cast<Circle>(ps); pc != nullptr) { ... } // not recommended
```

**Example** 隐式转换为bool可以用作条件。
```cpp
for (string s; cin >> s; ) v.push_back(s);
```
这将调用istream的运算符bool()。

**注意** 一般来说，将整数与0进行明确的比较并不是多余的。原因是（与指针和布尔运算相反）一个整数经常有两个以上的合理值。
此外，0经常被用来表示成功。因此，最好是对比较进行具体说明。
```cpp
void f(int i)
{
    if (i)            // suspect
    // ...
    if (i == success) // possibly better
    // ...
}
``` 
永远记住，一个整数可以有两个以上的值。

**Example, bad**
```cpp
if(strcmp(p1, p2)) { ... }   // are the two C-style strings equal? (mistake!)
```
是一个常见的初学者错误。如果你使用C风格的字符串，你必须熟知\<cstring\>函数。冗长的写法也不对。
```cpp
if(strcmp(p1, p2) != 0) { ... }   // are the two C-style strings equal? (mistake!)
```

**注意** 相反的条件最容易用一个否定词来表达。
```cpp
// These all mean "if p is nullptr"
if (!p) { ... }           // good
if (p == 0) { ... }       // redundant == 0, bad: don't use 0 for pointers
if (p == nullptr) { ... } // redundant == nullptr, not recommended
```

**Enforcement** 很简单，只要检查条件中是否有多余的!=和==的使用。

<br/>
## **运算**

<br/>
### **ES.100:不要混合有符号和无符号的计算**

**原因** 避免错误的结果。

**Example**
```cpp
int x = -3;
unsigned int y = 7;

cout << x - y << '\n';  // unsigned result, possibly 4294967286
cout << x + y << '\n';  // unsigned result: 4
cout << x * y << '\n';  // unsigned result, possibly 4294967275
```
在更现实的例子中更难发现这个问题。

**注意** 不幸的是，C++使用有符号的整数作为数组下标，而标准库使用无符号的整数作为容器下标。这就排除了一致性。
使用gsl::index来表示子标；见[ES.107](#es107不要对下标使用无符号最好使用gsl::index)。

**Enforcement**
- 编译器已经知道，有时还会发出警告。
- (为了避免噪音）不要在一个混合的有符号/无符号比较上做标记，其中一个参数是sizeof或对容器.size()的调用，另一个是ptrdiff\_t。

<br/>
### **ES.101:位操作要使用无符号类型**

**原因** 无符号类型支持位操作，而不会有sign bit的意外。

**Example**
```cpp
unsigned char x = 0b1010'1010;
unsigned char y = ~x;   // y == 0b0101'0101;
```

**注意** 无符号类型也可用于模运算。但是，如果您想要模运算，请根据需要添加注释，注意对环绕行为的依赖，因为这样的代码可能会让许多程序员感到惊讶。

**Enforcement** 
- 由于在标准库中使用了无符号下标，所以一般情况下几乎不可能。

<br/>
### **ES.102:算数计算要使用有符号类型**

**原因** 因为大多数算术都被认为是有符号的；当y\>x时，x-y产生一个负数，除非在极少数情况下，你真的需要模数算术。

**Example** 如果你没有预料到，无符号算术会产生令人惊讶的结果。对于有符号和无符号的混合算术来说，这更是如此。
```cpp
template<typename T, typename T2>
T subtract(T x, T2 y)
{
    return x - y;
}

void test()
{
    int s = 5;
    unsigned int us = 5;
    cout << subtract(s, 7) << '\n';       // -2
    cout << subtract(us, 7u) << '\n';     // 4294967294
    cout << subtract(s, 7u) << '\n';      // -2
    cout << subtract(us, 7) << '\n';      // 4294967294
    cout << subtract(s, us + 2) << '\n';  // -2
    cout << subtract(us, s + 2) << '\n';  // 4294967294
}
```
这里我们已经非常明确地说明了正在发生的事情，但是如果你看到us - (s + 2) or s += 2; ...; us - s, 
你会可靠地怀疑结果会打印成4294967294吗？

**Example** 标准库使用无符号类型来表示下标。内置数组使用有符号的类型来表示子符号。这使得惊喜（和错误）不可避免。
```cpp
int a[10];
for (int i = 0; i < 10; ++i) a[i] = i;
vector<int> v(10);
// compares signed to unsigned; some compilers warn, but we should not
for (gsl::index i = 0; i < v.size(); ++i) v[i] = i;

int a2[-2];         // error: negative size

// OK, but the number of ints (4294967294) is so large that we should get an exception
vector<int> v2(-2);
```
使用gsl::index进行下标；见[ES.107](#es107不要对下标使用无符号最好使用gsl::index)。

**Enforcement**
- 有符号和无符号混合运算要标记出
- 分配给或打印的无符号算术的结果为有符号要标记出
- 负数（如-2）作为容器下标使用要标记出
- (为了避免噪音）不要在一个混合的有符号/无符号比较上做标记，其中一个参数是sizeof或对容器.size()的调用，另一个是ptrdiff\_t。

<br/>
### **ES.103:不要溢出**

**原因** 溢出通常会使你的数字算法失去意义。增加一个超过最大值的数值会导致内存损坏和未定义的行为。 

**Example, bad**
```cpp
int a[10];
a[10] = 7;   // bad, array bounds overflow

for (int n = 0; n <= 10; ++n)
    a[n] = 9;   // bad, array bounds overflow
```

**Example, bad**
```cpp
int n = numeric_limits<int>::max();
int m = n + 1;   // bad, numeric overflow
```

**Example, bad**
```cpp
int area(int h, int w) { return h * w; }

auto a = area(10'000'000, 100'000'000);   // bad, numeric overflow
```

**例外** 如果你真的想进行模数运算，请使用无符号类型。

**替代方案** 对于可以承受一些开销的关键应用，可以使用一个范围检查的整数和/或浮点类型。

<br/>
### **ES.104:不要下溢**

**原因** 值递减超过最小值会导致内存损坏和未定义行为。

**Example, bad**
```cpp
int a[10];
a[-2] = 7;   // bad

int n = 101;
while (n--)
    a[n - 1] = 9;   // bad (twice)
```

**例外** 如果你真的想进行模数运算，请使用无符号类型。

<br/>
### **ES.105:不要被整数零除掉**

**原因** 其结果是未定义的，可能是crash。

**Example, bad**
```cpp
int divide(int a, int b)
{
    // BAD, should be checked (e.g., in a precondition)
    return a / b;
}
```

**Example, good**
```cpp
int divide(int a, int b)
{
    // good, address via precondition (and replace with contracts once C++ gets them)
    Expects(b != 0);
    return a / b;
}

double divide(double a, double b)
{
    // good, address via using double instead
    return a / b;
}
```
C语言中整数除0是未定义行为，早期计算机可能崩溃；如果0是常量，可能导致编译警告。浮点数除0为无穷大或NaN。

**替代方案** 对于可以承受一些开销的关键应用，可以使用范围检查的整数和/或浮点类型。

**Enforcement**
- 除以一个可能为零的整数要标记出

<br/>
### **ES.106:不要试图通过使用无符号来避免负值**

**原因** 选择无符号意味着对整数的通常行为的许多改变，包括模数运算，可以抑制与溢出有关的警告，并为与有符号/无符号混合有关的错误打开大门。
使用无符号实际上并不能消除负值的可能性。 

**Example**
```cpp
unsigned int u1 = -2;   // Valid: the value of u1 is 4294967294
int i1 = -2;
unsigned int u2 = i1;   // Valid: the value of u2 is 4294967294
int i2 = u2;            // Valid: the value of i2 is -2
```
这种（完全合法的）构造的这些问题在实际代码中很难发现，而且是许多现实世界中的错误的来源。
```cpp
unsigned area(unsigned height, unsigned width) { return height*width; } // [see also](#Ri-expects)
// ...
int height;
cin >> height;
auto a = area(height, 2);   // if the input is -2 a becomes 4294967292
```
请记住，当-1被分配为unsigned int时，会成为最大的unsigned int。另外，由于无符号算术是模数算术，所以乘法没有溢出，而是回绕。

**Example**
```cpp
unsigned max = 100000;    // "accidental typo", I mean to say 10'000
unsigned short x = 100;
while (x < max) x += 100; // infinite loop
```
如果x是一个signed short，我们可以对溢出时的未定义行为发出警告。

**替代方案**
- 使用有符号整数且判断x>=0
- 使用正整数类型
- 使用整数子范围类型
- Assert(-1 < x)

```cpp
struct Positive {
    int val;
    Positive(int x) :val{x} { Assert(0 < x); }
    operator int() { return val; }
};

int f(Positive arg) { return arg; }

int r1 = f(2);
int r2 = f(-2);  // throws
```

**Enforcement** 见[ES.100](#es100不要混合有符号和无符号的计算)

<br/>
### **ES.107:不要对下标使用无符号，最好使用gsl::index**

**原因** 为了避免有符号/无符号的混淆。实现更好的优化。实现更好的错误检测。避免auto和int的陷阱。

**Example, bad**
```cpp
vector<int> vec = /*...*/;

for (int i = 0; i < vec.size(); i += 2)                    // might not be big enough
    cout << vec[i] << '\n';
for (unsigned i = 0; i < vec.size(); i += 2)               // risk wraparound
    cout << vec[i] << '\n';
for (auto i = 0; i < vec.size(); i += 2)                   // might not be big enough
    cout << vec[i] << '\n';
for (vector<int>::size_type i = 0; i < vec.size(); i += 2) // verbose
    cout << vec[i] << '\n';
for (auto i = vec.size()-1; i >= 0; i -= 2)                // bug
    cout << vec[i] << '\n';
for (int i = vec.size()-1; i >= 0; i -= 2)                 // might not be big enough
    cout << vec[i] << '\n';
```

**Example, good**
```cpp
vector<int> vec = /*...*/;

for (gsl::index i = 0; i < vec.size(); i += 2)             // ok
    cout << vec[i] << '\n';
for (gsl::index i = vec.size()-1; i >= 0; i -= 2)          // ok
    cout << vec[i] << '\n';
```

**注意** 内置数组使用有符号的下标。标准库的容器使用无符号的下标。因此，没有完美和完全兼容的解决方案是可能的（除非在未来的某一天，标准库容器改变为使用有符号的下标）。
考虑到无符号和有符号/无符号混合的已知问题，最好坚持使用足够大的（有符号）整数，这是由gsl::index保证的。

**Example**
```cpp
template<typename T>
struct My_container {
public:
    // ...
    T& operator[](gsl::index i);    // not unsigned
    // ...
};
```

**替代方案**
- use algorithms
- use range-for
- use iterators/pointers

**Enforcement**
- 只要标准库的容器弄错了，就非常棘手。

<br/>
# **Per:性能**
	
- [Per.1:不要无故优化](#per1不要无故优化) 
- [Per.2:不要过早优化](#per2不要过早优化) 
- [Per.3:不要优化非关键部分](#per3不要优化非关键部分) 
- [Per.4:不要认为复杂的代码比简单代码快](#per4不要认为复杂的代码比简单代码快) 
- [Per.5:不要认为low-level代码一定比high-level代码快](#per5不要认为low-level代码一定比high-level代码快) 
- [Per.6:不要在没有测量情况下要求性能](#per6不要在没有测量情况要求性能) 
- [Per.7:设计以实现可优化](#per7设计以实现可优化) 
- [Per.10:依赖静态类型系统](#per10依赖静态类型系统) 
- [Per.11:将计算从运行时移到编译时](#per11将计算从运行时移到编译时) 
- [Per.12:消除多余的别名](#per12消除多余的别名) 
- [Per.13:消除多余的indirections](#per13消除多余的indirections) 
- [Per.14:最大限度减少allocations和deallocations](#per14最大限度减少allocations和deallocations)
- [Per.15:不要在关键分支上allocate](#per15不要在关键分支上allocate)
- [Per.16:使用紧凑数据结构](#per16使用紧凑数据结构)
- [Per.17:首先声明时间关键结构中最常用的成员](#per17首先声明时间关键结构中最常用的成员)
- [Per.18:空间就是时间](#per18空间就是时间)
- [Per.19:可预测地访问内存](#per19可预测地访问内存)
- [Per.30:在关键路径上避免上下文切换](#per30在关键路径上避免上下文切换)


<br/>
### **Per.1:不要无故优化**

<br/>
### **Per.2:不要过早优化**

<br/>
### **Per.3:不要优化非关键部分**

<br/>
### **Per.4:不要认为复杂的代码比简单代码快**

**Example,good**
```cpp
// clear expression of intent, fast execution

vector<uint8_t> v(100000);

for (auto& c : v)
    c = ~c;
```
**Example,bad**
```cpp
// intended to be faster, but is often slower

vector<uint8_t> v(100000);

for (size_t i = 0; i < v.size(); i += sizeof(uint64_t)) {
    uint64_t& quad_word = *reinterpret_cast<uint64_t*>(&v[i]);
    quad_word = ~quad_word;
}
```

<br/>
### **Per.5:不要认为low-level代码一定比high-level代码快**

<br/>
### **Per.6:不要在没有测量情况要求性能**

<br/>
### **Per.7:设计以实现可优化**

**Example**
```cpp
void qsort (void* base, size_t num, size_t size, int (*compar)(const void*, const void*));
```
什么时候想对内存进行排序？实际上，我们对元素的序列进行排序，通常存储在容器内。对qsort的调用丢掉了很多有用的信息（例如，元素类型），迫使用户重复已知的信息（例如，元素大小），迫使
用户写额外代码（例如比较两个double函数）。这意味着增加了程序员的工作，容易出错，并使编译器失去了优化所需的信息。
```cpp
double data[100];
// ... fill a ...

// 100 chunks of memory of sizeof(double) starting at
// address data using the order defined by compare_doubles
qsort(data, 100, sizeof(double), compare_doubles);
```
从接口设计角度看，qsort丢失了有用的信息。
以下代码更优(C++98)：
```cpp
template<typename Iter>
    void sort(Iter b, Iter e);  // sort [b:e)

sort(data, data + 100);
```
在这里，我们使用编译器关于数组的大小，元素类型以及如何比较两个double。
C++20:
```cpp
// sortable specifies that c must be a
// random-access sequence of elements comparable with <
void sort(sortable auto& c);

sort(c);
```
这里还有一个小缺点，sort隐含地需要元素间<比较操作。为了完善这个接口，我们需要第二个版本，它接受一个比较标准。
```cpp
// compare elements of c using p
template<random_access_range R, class C> requires sortable<R, C>
void sort(R&& r, C c);
```
标准库的sort提供了这两个版本，甚至更多。

**Example**
```cpp
template<class ForwardIterator, class T>
bool binary_search(ForwardIterator first, ForwardIterator last, const T& val);
```
binary\_search会查找7是否在c中，但不会告诉你7在哪里？或者7有几个？
因此，标准库也提供了：
```cpp
template<class ForwardIterator, class T>
ForwardIterator lower_bound(ForwardIterator first, ForwardIterator last, const T& val);
```
lower\_bound返回了第一个大于val的元素迭代器，如果有的话，没有则指向最后一个迭代器。
因此标准库还提供了：
```cpp
template<class ForwardIterator, class T>
pair<ForwardIterator, ForwardIterator>
equal_range(ForwardIterator first, ForwardIterator last, const T& val);
```
```cpp
auto r = equal_range(begin(c), end(c), 7);
for (auto p = r.first; p != r.second; ++p)
    cout << *p << '\n';
```
显然，这三个接口是由相同的基本代码实现的。它们只是向用户展示了三种二进制搜索算法。从最简单的，到返回完整的，但不一定需要的信息。

**注意**
事情是有成本的。不要对成本过于偏执，现代计算机确实非常快。但是你要对所使用的东西的成本数量级有一个大致的概念。例如，对一次内存访问、一次函数调用、一次字符串比较，一个系统调用、一次磁盘访问和一条网络消息
的成本又一个大致的概念。

<br/>
### **Per.10:依赖静态类型系统**

<br/>
### **Per.11:将计算从运行时移到编译时**

**原因** 减少代码大小以及运行时开销。通过常量来避免data races。在编译期捕捉错误，从而减少错误处理代码。

**Example**
```cpp
double square(double d) { return d*d; }
static double s2 = square(2);    // old-style: dynamic initialization

constexpr double ntimes(double d, int n)   // assume 0 <= n
{
        double m = 1;
        while (n--) m *= d;
        return m;
}
constexpr double s3 {ntimes(2, 3)};  // modern-style: compile-time initialization
```
s2初始化并不罕见，特别是对于square更复杂的初始化，然而相比s3，它有两个问题：
- 运行时函数调用的开销
- s2可能在初始化前就被其他线程访问了

**Example**
```cpp
constexpr int on_stack_max = 20;

template<typename T>
struct Scoped {     // store a T in Scoped
        // ...
    T obj;
};

template<typename T>
struct On_heap {    // store a T on the free store
        // ...
        T* objp;
};

template<typename T>
using Handle = typename std::conditional<(sizeof(T) <= on_stack_max),
                    Scoped<T>,      // first alternative
                    On_heap<T>      // second alternative
               >::type;

void f()
{
    Handle<double> v1;                   // the double goes on the stack
    Handle<std::array<double, 200>> v2;  // the array goes on the free store
    // ...
}
```
在编译期间就是可以计算出是Scoped还是On\_heap。

<br/>
### **Per.12:消除多余的别名**

<br/>
### **Per.13:消除多余的indirections**

<br/>
### **Per.14:最大限度减少allocations和deallocations**

<br/>
### **Per.15:不要在关键分支上allocate**

<br/>
### **Per.16:使用紧凑数据结构**

<br/>
### **Per.17:首先声明时间关键结构中最常用的成员**

<br/>
### **Per.18:空间就是时间**

<br/>
### **Per.19:可预测地访问内存**

**原因**
性能对于缓存非常敏感，而缓存算法更倾向于对相邻数据的简单访问。

**Example**
```cpp
int matrix[rows][cols];

// bad
for (int c = 0; c < cols; ++c)
    for (int r = 0; r < rows; ++r)
        sum += matrix[r][c];

// good
for (int r = 0; r < rows; ++r)
    for (int c = 0; c < cols; ++c)
        sum += matrix[r][c];
```

<br/>
### **Per.30:在关键路径上避免上下文切换**

<br/>
# **CP:并发和并行**

主要阐述使用ISO标准C++设施来表达基本的并发和并行原则和规则。
三大设计目标：
- 帮助编写适合在多线程环境下使用的代码
- 展示标准库提供的线程原语的干净、安全使用
- 如果并发和并行不能提高性能时提供指导

注意的是，C++并发是个未完成的状态，C++11引入了很多核心并发原语，C++14和C++17进一步改进，人们对编写C++并发代码愈发兴趣，我们期望与库相关的向导会随着时间推移而发生重大变化。

- 并发并行原则摘要：
	- [CP.1:写代码时，应假定你的代码会是多线程运行](#cp1假定你的代码会是多线程运行的一部分) 
	- [CP.2:避免数据争用](#cp2避免数据争用) 
	- [CP.3:写数据显式共享最小化](#cp3显式共享写数据最小化) 
	- [CP.4:多用task而不是线程思考问题](#cp4多用task而不是线程思考问题) 
	- [CP.8:不要用volatile来做同步](#cp8不要用volatile来做同步) 
	- [CP.9:可行的情况下，用工具验证代码并发的正确性](#cp9可行的情况下用工具验证代码并发的正确性) 
- 还有一些文章:	
	- [CP.con:并发](#cpcon并发) 
	- [CP.coro:Coroutines](#cpcorocoroutines) 
	- [CP.par:Parallelism](#cpparparallelism) 
	- [CP.mess:Message passing](#cpmessmessage-passing) 
	- [CP.vec:Vectorization](#cpvecvectorization) 
	- [CP.free:Lock-free programming](#cpfreelock-free-programming) 
	- [CP.etc:Etc.并发规则](#cpetcetc并发规则) 

<br/>
### **CP.1:假定你的代码会是多线程运行的一部分**

**Example,bad**
```cpp
double cached_computation(int x)
{
    // bad: these statics cause data races in multi-threaded usage
    static int cached_x = 0.0;
    static double cached_result = COMPUTATION_OF_ZERO;

    if (cached_x != x) {
        cached_x = x;
        cached_result = computation(x);
    }
    return cached_result;
}
```

**Example,good**
```cpp
struct ComputationCache {
    int cached_x = 0;
    double cached_result = COMPUTATION_OF_ZERO;

    double compute(int x) {
        if (cached_x != x) {
            cached_x = x;
            cached_result = computation(x);
        }
        return cached_result;
    }
};
```

<br/>
### **CP.2:避免数据争用**

如果两个线程对一份数据访问（没有同步），且至少有一个是writer，那么就存在数据争用(data race)。关于如何消除data race来同步数据，可以参考一本关于并发性的好书（参见文献）。

**Example,bad**
```cpp
int get_id()
{
  static int id = 1;
  return id++;
}
```
有几处错误：
- 线程A加载id，系统上下文切换，A切出去，其他线程同时创建了几百个id，线程A再被切回来，id数值被写回，但写回数值只是加载时数值+1。
- 线程A和线程B加载和写回同时进行，最终得到的是相同id。

**Example,bad**
```cpp
void f(fstream& fs, regex pattern)
{
    array<double, max> buf;
    int sz = read_vec(fs, buf, max);            // read from fs into buf
    gsl::span<double> s {buf};
    // ...
    auto h1 = async([&] { sort(std::execution::par, s); });     // spawn a task to sort
    // ...
    auto h2 = async([&] { return find_all(buf, sz, pattern); });   // spawn a task to find matches
    // ...
}
```
sort函数同时读写buf，所以再这个栈上存在data race，这不容易被发现。

**Example,bad**
```cpp
// code not controlled by a lock

unsigned val;

if (val < 5) {
    // ... other thread can change val here ...
    switch (val) {
    case 0: // ...
    case 1: // ...
    case 2: // ...
    case 3: // ...
    case 4: // ...
    }
}
```
编译器可能使用5个size的跳转表来实现switch，一旦多线程data race造成val数值超出size，那么会造成严重bug。

**Enforcement:**
可以用一些开源或商业软件可以检测出此类问题，注意的是它也有成本和盲点。静态工具有很多误报，运行态工具又有很大开销。使用多种工具比单一工具可以捕捉更多问题。
- 避免全局数据
- 避免static数据
- 在栈中多用具体类型数据（不过多传递指针）
- 多使用不变量（常量，constexpr，const）

<br/>
### **CP.3:显式共享写数据最小化**

**原因** 如果不共享写数据，就不会有data race。共享越少，就越少忘掉同步访问数据（数据争用）。共享越少，就越少在等待锁（性能提高）。

**Example**
```cpp
bool validate(const vector<Reading>&);
Graph<Temp_node> temperature_gradients(const vector<Reading>&);
Image altitude_map(const vector<Reading>&);
// ...

void process_readings(const vector<Reading>& surface_readings)
{
    auto h1 = async([&] { if (!validate(surface_readings)) throw Invalid_data{}; });
    auto h2 = async([&] { return temperature_gradients(surface_readings); });
    auto h3 = async([&] { return altitude_map(surface_readings); });
    // ...
    h1.get();
    auto v2 = h2.get();
    auto v3 = h3.get();
    // ...
}
```
如果没有标明const，我们就需要review每个异步函数调用是否存在data race。让surface\_readings成为const，对于这个函数来说，就相当于只在这个函数体内使用。

**注意** 不变数据可以安全有效地被共享，这不会发生data race。另外见CP.mess:数据传递，以及CP.31:优先传递value。

<br/>
### **CP.4:多用task而不是线程思考问题**

**原因** 线程是关于机器的实现，task则是关于应用的实现概念。基于应用的概念更容易理解。

**Example**
```cpp
void some_fun(const std::string& msg)
{
    std::thread publisher([=] { std::cout << msg; });      // bad: less expressive
                                                           //      and more error-prone
    auto pubtask = std::async([=] { std::cout << msg; });  // OK
    // ...
    publisher.join();
}
```

**注意** 除了async，标准库的基础设施都是low-level，面向机器，线程和锁的级别。这是基础性的，但我们需要努力提高抽象水平：为了生产力，为了可靠性，为了性能。这是使用high level，更面向应用的库（如果可能的话，建立在标准库上）的有力论据。

<br/>
### **CP.8:不要用volatile来做同步**

**原因** 在C++，不像其他语言，volatile关键字不提供原子性，不保证线程间同步，不保证指令不会乱序（不管是编译器还是机器），它与并发没有关系。

**Example, bad:**
```cpp
int free_slots = max_slots; // current source of memory for objects

Pool* use()
{
    if (int n = free_slots--) return &pool[n];
}
```
data race, fix:
```cpp
volatile int free_slots = max_slots; // current source of memory for objects

Pool* use()
{
    if (int n = free_slots--) return &pool[n];
}
```
没有改变任何并发相关，data race依旧，改为：
```cpp
atomic<int> free_slots = max_slots; // current source of memory for objects

Pool* use()
{
    if (int n = free_slots--) return &pool[n];
}
```
free\_slots--操作是原子的，而不是那种读取-自增-写入模式，其他线程在这三个操作中都可能介入。

**替代** 在使用volatile的地方尽量使用atomic，更复杂的地方用mutex。

**See also** （极少见）[非C++内存时使用volatile](r#非C++内存时使用volatile)

<br/>
### **CP.9:可行的情况下，用工具验证代码并发的正确性**

经验告诉我们并发代码很难写正确，编译时检查、运行时检查、单元测试等发现并发错误，不如在顺序代码中有效，微小的bug都会造成巨大的灾难，包括内存损坏、死锁和安全漏洞。

**注意**
- 静态检测工具：Clang和一些旧版本的GCC都对线程安全的静态annotation提供一些支持。坚持使用这些技术可以将许多class的线程安全错误转化为编译器错误。annotation通常是局部的，
将一个特定的成员变量标记为在一个特定mutex保护下，通常容易被学习。但是像许多静态工具一样，会出现很多false negative，有些错误需要被捕捉到却被通过了。
- 动态检测工具：Clang的Thread SaniTizer(TSAN)是个强有力的工具，它改变了程序的构建和执行以增加对内存访问记录的登记，在二进制文件中特定执行中有绝对的识别。但是代价就是内存增加5-10倍，CPU下降2-20倍。这种动态检测工具适用于集成测试、金丝雀推送和多线程单元测试。当TSAN发现一个问题时，通常就代表它有效地识别出一个真实会发生的data race，但是它只能在一个特定的执行路径上才会识别。

<br/>
### **CP.con:并发**

本节重点讨论通过共享数据进行多线程交互
- 并行算法[parallelism](r#cp并行)
- 不通过显式共享的任务间交互[messing](r#cp消息传递)
- 向量并行[vectorization](r#cp向量并行)
- 无锁编程[lock free](r#cp无锁编程)

并发规则摘要：
- [CP.20:使用RAII，而不直接调用lock()/unlock()](#cp20使用raii而不直接调用lockunlock)
- [CP.21:使用std::mutex或std::scoped\_lock来获得多个mutex](#cp21使用stdmutex或stdscoped_lock来获得多个mutex)
- [CP.22:不要在lock代码段中调用未知代码](#cp22不要在lock代码段中调用未知代码)
- [CP.23:把一个joining线程当成scoped container](#cp23把一个joining线程当成scoped-container)
- [CP.24:把thread当成一个全局的container](#cp24把thread当成一个全局的container)
- [CP.25:优先使用gsl::joining\_thread，而不是thread](#cp25优先使用gsljoining_thread而不是thread)
- [CP.26:不要deatch线程](#cp26不要deatch线程)
- [CP.31:线程间small amount data用value传递，而不用引用或指针传递](#cp31线程间small-amount-data用value传递而不用引用或指针传递)
- [CP.32:unreleated线程之间应该用shared\_ptr共享所有权](#cp32unreleated线程之间应该用shared_ptr共享所有权)
- [CP.40:context switching最小化](#cp40context-switching最小化)
- [CP.41:线程创建与线程销毁最小化](#cp41线程创建与线程销毁最小化)
- [CP.42:不要在没有condition情况下调用wait](#cp42不要在没有condition情况下调用wait)
- [CP.43:关键临界区开销最小化](#cp43关键临界区开销最小化)
- [CP.44:所有的lock\_guard和unique\_lock都要记得命名](#cp44所有的lock_guard和unique_lock都要记得命名)
- [CP.50:如果要guard某份数据，那么定义一个mutex，如果可能的话，使用synchronized\_value\<T\>](#cp50如果要guard某份数据那么定义一个mutex如果可能的话使用synchronized_valuet)

<br/>
### **CP.20:使用RAII，而不直接调用lock()/unlock()**

**原因** 避免忘记释放锁

**Example, bad**
```cpp
mutex mtx;

void do_stuff()
{
    mtx.lock();
    // ... do stuff ...
    mtx.unlock();
}
```
一旦有人在...中return或者抛个异常或其他，都会导致忘记mtx.unlock。
```cpp
mutex mtx;

void do_stuff()
{
    unique_lock<mutex> lck {mtx};
    // ... do stuff ...
}
```

<br/>
### **CP.21:使用std::mutex或std::scoped_lock来获得多个mutex**

**原因** 避免在多个mutex上死锁

**Example**
```cpp
// thread 1
lock_guard<mutex> lck1(m1);
lock_guard<mutex> lck2(m2);

// thread 2
lock_guard<mutex> lck2(m2);
lock_guard<mutex> lck1(m1);
```
以上代码死锁
```cpp
// thread 1
lock(m1, m2);
lock_guard<mutex> lck1(m1, adopt_lock);
lock_guard<mutex> lck2(m2, adopt_lock);

// thread 2
lock(m2, m1);
lock_guard<mutex> lck2(m2, adopt_lock);
lock_guard<mutex> lck1(m1, adopt_lock);
```
C++17更佳
```cpp
// thread 1
scoped_lock<mutex, mutex> lck1(m1, m2);

// thread 2
scoped_lock<mutex, mutex> lck2(m2, m1);
```
还有
```cpp
lock_guard lck1(m1, adopt_lock);
```
m1的mutex类型会被推导出。

<br/>
### **CP.22:不要在lock代码段中调用未知代码**

**原因** 未知代码调用可能导致死锁

**Example**
```cpp
void do_this(Foo* p)
{
    lock_guard<mutex> lck {my_mutex};
    // ... do something ...
    p->act(my_data);
    // ...
}
```
Foo::act可能是一个虚函数，最终调用的是暂未实现的子类函数，它可能重复调用do\_this，造成递归调用死锁。也可能在等待另外一个mutex的锁，它没有在合理时间返回，导致do\_this调用阻塞。

**Example**
调用未知代码的一个常见例子，比如对同个对象的重复锁，这种可以通过recursive\_mutex解决。
```cpp
recursive_mutex my_mutex;

template<typename Action>
void do_something(Action f)
{
    unique_lock<recursive_mutex> lck {my_mutex};
    // ... do something ...
    f(this);    // f will do something to *this
    // ...
}
```

**Enforcement**
- non-recursive锁保护代码中调用虚函数要标记出。
- non-recursive锁保护代码中调用callback要标记出。

<br/>
### **CP.23:把一个joining线程当成scoped container**

**原因** 为保证指针安全和避免泄漏，我们要考虑thread中使用了哪些指针。当一个线程join时，我们可以安全地把指针传递给线程内以及线程外层作用域内的对象。

**Example**
```cpp
void f(int* p)
{
    // ...
    *p = 99;
    // ...
}
int glob = 33;

void some_fct(int* p)
{
    int x = 77;
    joining_thread t0(f, &x);           // OK
    joining_thread t1(f, p);            // OK
    joining_thread t2(f, &glob);        // OK
    auto q = make_unique<int>(99);
    joining_thread t3(f, q.get());      // OK
    // ...
}
```
gsl::joining\_thread表示一个带有析构函数的std::thread，它可以join，但是不可以detach。OK的意思是，只要线程能使用指向某对象的指针，该对象就会在范围内存活。线程并发不影响这里的生命期或所有权问题。

**Enforcement**
确保joining\_thread不会被detach()

<br/>
### **CP.24:把thread当成一个全局的container**

**原因** 为了指针安全以及避免泄漏，我们要考虑thread使用了哪些指针，如果一个thread是detach的，我们可以安全地将指针传递给static对象和free store object。

**Example**
```cpp
void f(int* p)
{
    // ...
    *p = 99;
    // ...
}

int glob = 33;

void some_fct(int* p)
{
    int x = 77;
    std::thread t0(f, &x);           // bad
    std::thread t1(f, p);            // bad
    std::thread t2(f, &glob);        // OK
    auto q = make_unique<int>(99);
    std::thread t3(f, q.get());      // bad
    // ...
    t0.detach();
    t1.detach();
    t2.detach();
    t3.detach();
    // ...
}
```
OK表示线程使用的指针，指针所指的对象，在线程使用指针时都是活着的。BAD表示线程使用指针时，其所指对象可能已经destroyed。线程并发不影响其生命期和所有权。

**注意** 即使是指向static对象也可能存在问题，detach线程可能一直运行到程序结束，那时static对象将会销毁，这时detach线程中可能还有使用指向该对象的指针，那么会造成race。

**注意** 如果不使用detach线程或者仅使用gsl::joining\_thread，那么本规则是多余的。

**Enforcement** 试图将局部变量传递给detach线程会被标记出。

<br/>
### **CP.25:优先使用gsl::joining_thread，而不是thread**

**原因** joining\_thread会在最后join，detach线程不好监控，deatch线程很难保证不会出错。

**Example,bad**
```cpp
void f() { std::cout << "Hello "; }

struct F {
    void operator()() const { std::cout << "parallel world "; }
};

int main()
{
    std::thread t1{f};      // f() executes in separate thread
    std::thread t2{F()};    // F()() executes in separate thread
}  // spot the bugs
```
**Example**
```cpp
void f() { std::cout << "Hello "; }

struct F {
    void operator()() const { std::cout << "parallel world "; }
};

int main()
{
    std::thread t1{f};      // f() executes in separate thread
    std::thread t2{F()};    // F()() executes in separate thread

    t1.join();
    t2.join();
}  // one bad bug left
```

**注意** 把一些不朽的thread做成全局的，把它们放入一个封闭的作用域内，或者把它们放入free store上，而不是detach。

**注意** 因为很多旧代码以及第三方库使用的是std::thread，因此本规则很难引入。

**Enforcement**
- 建议使用gsl::joining\_thread或C++20的std::jthread。
- 如果线程detach，建议把所有权导出到一个封闭scoped。
- 一个线程是否join或detach不明，应发出警告。

<br/>
### **CP.26:不要deatch线程**

**原因** detach线程很难被监控以及交互，很难保证deatch线程按预期完成及生命期。

**Example**
```cpp
void heartbeat();

void use()
{
    std::thread t(heartbeat);             // don't join; heartbeat is meant to run forever
    t.detach();
    // ...
}
```
以上代码，我们如何保证detach线程是否还存活？heartbeat函数中可能出错，心跳丢失可能对一个系统来说是非常严重，所以需要和这个心跳线程交互，（比如通过condition\_variable）。

另一种且更好的解决方案是把它放在更外围的scope，来创建或激活，这样能更好地管理生命期。
```cpp
void heartbeat();

gsl::joining_thread t(heartbeat);             // heartbeat is meant to run "forever"
```
这个心跳线程，除非遇到错误、硬件问题等，不然会随着程序一直运行。

有时候我们需要分离线程创建与线程所有权。
```cpp
void heartbeat();

unique_ptr<gsl::joining_thread> tick_tock {nullptr};

void use()
{
    // heartbeat is meant to run as long as tick_tock lives
    tick_tock = make_unique<gsl::joining_thread>(heartbeat);
    // ...
}
```

<br/>
### **CP.31:线程间small amount data用value传递，而不用引用或指针传递**

**原因** 小数据copy比较便宜，对这小数据进行锁同步反而开销更大，copy天然表示unique所有权，消除了data race。

**Example**
```cpp
string modify1(string);
void modify2(string&);

void fct(string& s)
{
    auto res = async(modify1, s);
    async(modify2, s);
}
```
modify1拷贝，modify2不是，modify1单线程多线程都可用，modify2多线程则需要同步。如果string比较短，比如10个字符，那么调用modify1非常快，开销最多的将是线程切换。如果string长度超过100万个字符，拷贝显然不合适。

<br/>
### **CP.32:unreleated线程之间应该用shared_ptr共享所有权**

<br/>
### **CP.40:context switching最小化**

<br/>
### **CP.41:线程创建与线程销毁最小化**

**原因** 线程创建开销很大

**Example**
```cpp
void worker(Message m)
{
    // process
}

void dispatcher(istream& is)
{
    for (Message m; is >> m; )
        run_list.push_back(new thread(worker, m));
}
```
每一条message都要创建一个线程，run\_list最终也要负责将所有线程都销毁。

**Example**
```cpp
Sync_queue<Message> work;

void dispatcher(istream& is)
{
    for (Message m; is >> m; )
        work.put(m);
}

void worker()
{
    for (Message m; m = work.get(); ) {
        // process
    }
}

void workers()  // set up worker threads (specifically 4 worker threads)
{
    joining_thread w1 {worker};
    joining_thread w2 {worker};
    joining_thread w3 {worker};
    joining_thread w4 {worker};
}
```

**注意** 如果你的系统有更好的线程池，或者有更好的消息队列，那么直接用这取代。

<br/>
### **CP.42:不要在没有condition情况下调用wait**

**原因** wait如果没有等待在某个condition，可能会错过wakeup，或者被wakeup之后发现没有事可做。

**Example,bad**
```cpp
std::condition_variable cv;
std::mutex mx;

void thread1()
{
    while (true) {
        // do some work ...
        std::unique_lock<std::mutex> lock(mx);
        cv.notify_one();    // wake other thread
    }
}

void thread2()
{
    while (true) {
        std::unique_lock<std::mutex> lock(mx);
        cv.wait(lock);    // might block forever
        // do work ...
    }
}
```
如果其他线程消费了thread1的通知，那么thread2将永远等待。

**Example**
```cpp
template<typename T>
class Sync_queue {
public:
    void put(const T& val);
    void put(T&& val);
    void get(T& val);
private:
    mutex mtx;
    condition_variable cond;    // this controls access
    list<T> q;
};

template<typename T>
void Sync_queue<T>::put(const T& val)
{
    lock_guard<mutex> lck(mtx);
    q.push_back(val);
    cond.notify_one();
}

template<typename T>
void Sync_queue<T>::get(T& val)
{
    unique_lock<mutex> lck(mtx);
    cond.wait(lck, [this] { return !q.empty(); });    // prevent spurious wakeup
    val = q.front();
    q.pop_front();
}
```
现在如果get线程wakeup时发现queue为空（比如另外一个线程在之前先调用了get），它将立即sleep，等待wakeup。

**Enforcement** 所有wait没有等在某个condition将会被标记。

<br/>
### **CP.43:关键临界区开销最小化**

**原因** mutex临界区越小，其他线程越少等待。线程暂停和恢复开销比较大。

**Example**
```cpp
void do_something() // bad
{
    unique_lock<mutex> lck(my_lock);
    do0();  // preparation: does not need lock
    do1();  // transaction: needs locking
    do2();  // cleanup: does not need locking
}
```
改为
```cpp
void do_something() // bad
{
    do0();  // preparation: does not need lock
    my_lock.lock();
    do1();  // transaction: needs locking
    my_lock.unlock();
    do2();  // cleanup: does not need locking
}
```
最好用RAII
```cpp
void do_something() // OK
{
    do0();  // preparation: does not need lock
    {
        unique_lock<mutex> lck(my_lock);
        do1();  // transaction: needs locking
    }
    do2();  // cleanup: does not need locking
}
```

<br/>
### **CP.44:所有的lock_guard和unique_lock都要记得命名**

**原因** 所有未命名的local对象是一个立即超出scope的临时对象。

**Example**
```cpp
unique_lock<mutex>(m1);
lock_guard<mutex> {m2};
lock(m1, m2);
```

<br/>
### **CP.50:如果要guard某份数据，那么定义一个mutex，如果可能的话，使用synchronized_value\<T\>**

**原因** 使用synchronized\_value\<T\>可以确保数据有一个mutex，并且在数据访问时有正确的mutex锁定。参见WG21在未来TS或C++修订版中加入synchronized\_value\<T\>的建议。

**Example**
```cpp
struct Record {
    std::mutex m;   // take this mutex before accessing other members
    // ...
};

class MyClass {
    struct DataRecord {
       // ...
    };
    synchronized_value<DataRecord> data; // Protect the data with a mutex
};
```

<br/>
## **CP.coro:Coroutines**

Coroutine规则摘要：
- [CP.51:不要使用coroutines的捕获lambda](#cp51不要使用属于coroutines的捕获lambda)
- [CP.52:不要在横跨suspend points上持有锁或其他同步原语](#cp52不要在across-suspension-points上加锁或其他同步原语)
- [CP.53:coroutine的参数不应该传引用](#cp53coroutine的参数不应该传引用)

<br/>
### **CP.51:不要使用属于coroutines的捕获lambda**

**原因** 普通lambda所使用的模式在coroutine lambda中会变得危险。因为coroutine lambda有可能访问了freed memory在第一次suspension point之后，即使是smart pointer或者是可拷贝的类型。一个lambda会产生一个带有存储空间的闭包对象，它将在某个时间点上超出范围。闭包对象超出范围后，所捕获对象也会超出范围。普通的lambda在这个时间已经完成执行，所以不是个问题，而coroutine lambda可以在闭包对象析构后从暂停中恢复，这时所有的捕获都是free的内存访问。

**Example,bad**
```cpp
int value = get_value();
std::shared_ptr<Foo> sharedFoo = get_foo();
{
  const auto lambda = [value, sharedFoo]() -> std::future<void>
  {
    co_await something();
    // "sharedFoo" and "value" have already been destroyed
    // the "shared" pointer didn't accomplish anything
  };
  lambda();
} // the lambda closure object has now gone out of scope
```
**Example,better**
```cpp
int value = get_value();
std::shared_ptr<Foo> sharedFoo = get_foo();
{
  // take as by-value parameter instead of as a capture
  const auto lambda = [](auto sharedFoo, auto value) -> std::future<void>
  {
    co_await something();
    // sharedFoo and value are still valid at this point
  };
  lambda(sharedFoo, value);
} // the lambda closure object has now gone out of scope`
```
**Example,best**
```cpp
std::future<void> Class::do_something(int value, std::shared_ptr<Foo> sharedFoo)
{
  co_await something();
  // sharedFoo and value are still valid at this point
}

void SomeOtherFunction()
{
  int value = get_value();
  std::shared_ptr<Foo> sharedFoo = get_foo();
  do_something(value, sharedFoo);
}
```

**Enforcement** 一个lambda是coroutine，且有个非空的捕获列表，那应该标记出。

<br/>
### **CP.52:不要在across suspension points上加锁或其他同步原语**

**原因** 如果这样的话会有死锁风险。有一些wait需要当前线程做更多的工作，直到异步操作完成。如果持有锁的线程所做的工作需要相同的锁，那将会死锁，因为已经持有一个相同的锁。如果coroutine持有一个线程的锁，然后在另外一个线程处理完成那将是未定义的。coroutine即使在同个线程返回，但是在resume前抛出异常，那么lock guard不会被解构。

**Example,bad**
```cpp
std::mutex g_lock;

std::future<void> Class::do_something()
{
    std::lock_guard<std::mutex> guard(g_lock);
    co_await something(); // DANGER: coroutine has suspended execution while holding a lock
    co_await somethingElse();
}
```

**Example,good**
```cpp
std::mutex g_lock;

std::future<void> Class::do_something()
{
    {
        std::lock_guard<std::mutex> guard(g_lock);
        // modify data protected by lock
    }
    co_await something(); // OK: lock has been released before coroutine suspends
    co_await somethingElse();
}
```

**注意** 这种模式对性能也不好。如果达到一个suspension point，比如co\_await，那么当前函数就会停止，其他代码开始运行。而且\
这个coroutine恢复前可能要等待很久时间，这整个过程如果有锁，那么会一直被持有，而其他地方要获取这个锁却获取不上。

**Enforcement** 所有lock guard在coroutine suspend前没有destructed都应该被标记出来。

<br/>
### **CP.53:coroutine的参数不应该传引用**

**原因** 一旦coroutine达到一个suspend point，比如co\_wait，同步部分就会返回。那之后，任何通过引用传递的参数都是dangling的。任何对此操作都是未定义的，可能包括对已释放内存的操作。

**Example, Bad**
```cpp
std::future<int> Class::do_something(const std::shared_ptr<int>& input)
{
    co_await something();

    // DANGER: the reference to input may no longer be valid and may be freed memory
    co_return *input + 1;
}
```

**Example,Good**
```cpp
std::future<int> Class::do_something(std::shared_ptr<int> input)
{
    co_await something();
    co_return *input + 1; // input is a copy that is still valid here
}
```

**注意** 这个问题不单单是出现在第一次suspend point，后续代码如果添加或者删除suspend point都会重现这个bug，有些coroutine甚至在第一行代码执行前都有suspend point，这种情况下同样是unsafe的。所以一般传value，value拷贝后在整个coroutine内都是安全访问的。

**Enforcement** 所有传引用给coroutine都应该被标记。

<br/>
## **CP.mess:Message passing**

标准库的设施是low-level的，它专注于使用thread、mutex、atomic等接近硬件编程的需求。大多数人不应该工作在这一层面，因为它容易出错，而且开发效率不高。如果可能的话，使用更高层次的设施，比如消息库，并行算法，vectorization等。本节讨论消息传递，以便程序员不必进行显式同步。

消息传递原则摘要：
- [CP.60:使用future从并发的task中返回value](#cp60使用future从并发的task中返回value)
- [CP.61:使用async来新建并发任务](#cp61使用async来新建并发任务)
- 消息队列
- 消息传递库

<br/>
### **CP.60:使用future从并发的task中返回value**

<br/>
### **CP.61:使用async来新建并发任务**

**原因** 尽量避免用原生thread，promise。使用工厂函数比如std::async，它来处理创建和复用一个thread，而不必要得暴露原生thread。

**Example**
```cpp
int read_value(const std::string& filename)
{
    std::ifstream in(filename);
    in.exceptions(std::ifstream::failbit);
    int value;
    in >> value;
    return value;
}

void async_example()
{
    try {
        std::future<int> f1 = std::async(read_value, "v1.txt");
        std::future<int> f2 = std::async(read_value, "v2.txt");
        std::cout << f1.get() + f2.get() << '\n';
    } catch (const std::ios_base::failure& fail) {
        // handle exception here
    }
}
```

**注意** 不幸的是std::async不是完美的，它没有使用一个线程池，这意味着如果资源耗尽将会失败，而没有把你的任务排队起来，留着之后来处理。但是即使不能使用std::async，你也应该倾向于编写自己的返回future的工厂函数，而不是使用原生的promise。

**Example,bad**

这个例子展示了两个不同的方式，成功地使用了std::future，但在避免使用原生thread管理上失败了。
```cpp
void async_example()
{
    std::promise<int> p1;
    std::future<int> f1 = p1.get_future();
    std::thread t1([p1 = std::move(p1)]() mutable {
        p1.set_value(read_value("v1.txt"));
    });
    t1.detach(); // evil

    std::packaged_task<int()> pt2(read_value, "v2.txt");
    std::future<int> f2 = pt2.get_future();
    std::thread(std::move(pt2)).detach();

    std::cout << f1.get() + f2.get() << '\n';
}
```

**Example,good**
```cpp
void async_example(WorkQueue& wq)
{
    std::future<int> f1 = wq.enqueue([]() {
        return read_value("v1.txt");
    });
    std::future<int> f2 = wq.enqueue([]() {
        return read_value("v2.txt");
    });
    std::cout << f1.get() + f2.get() << '\n';
}
```

WorkQueue背后隐藏了调用read\_value所产生的线程创建等操作，用户只处理future，而不用面对原生的thread，promise，packaged\_task等。

<br/>
## **CP.free:Lock-free programming**

用mutex和condition\_variable来同步相对比较昂贵，此外还可能导致死锁。为了提高性能以及消除死锁可能性，不得不使用low-level的"lock-free"设施，这些设施依赖于短时独占内存（atomic）。无锁编程也用来实现更高级别的并发机制，比如thread和mutex。

无锁编程规则摘要：
- [CP.100:不要使用无锁编程，除非你绝对需要这样做](#cp100不要使用无锁编程除非你绝对需要这样做)
- [CP.101:不要信任你的硬件/编译器组合](#cp101不要信任你的硬件编译器组合)
- [CP.102:仔细研究文献](#cp102仔细研究文献)
- 怎么/什么时候用atomic
- 避免饥饿
- 使用无锁数据结构，而不是自己手工特定的无锁访问
- [CP.110:不要为初始化编写双重检查锁](#cp110不要为初始化编写双重检查锁)
- [CP.111:如果你真的需要双重检查锁，请使用常规模式](#cp111如果你真的需要双重检查锁请使用常规模式)
- 怎么/什么时候进行比较和互换

<br/>
### **CP.100:不要使用无锁编程，除非你绝对需要这样做**

**原因** 容易出错，需要对语言特性，机器结构和数据结构有专业水准的掌握。

**Example,bad**
```cpp
extern atomic<Link*> head;        // the shared head of a linked list

Link* nh = new Link(data, nullptr);    // make a link ready for insertion
Link* h = head.load();                 // read the shared head of the list

do {
    if (h->data <= data) break;        // if so, insert elsewhere
    nh->next = h;                      // next element is the previous head
} while (!head.compare_exchange_weak(h, nh));    // write nh to head or to h
```
此类代码中bug，通过测试来发现它真的很难。阅读关于ABA问题的文章。

**例外** 只要你使用顺序一致的内存模型（memory\_order\_seq\_cst），就可以简单安全使用atomic变量。

**注意** higher-level的并发机制，比如thread和mutex，是使用无锁编程实现的。

**替代方案** 使用他人实现的无锁数据结构作为某些库的一部分。

<br/>
### **CP.101:不要信任你的硬件/编译器组合**

**原因** 无锁编程所使用的low-level硬件接口是最难实现的，也是最微妙的出现可移植问题。

**注意** 指令reordering（静态和动态）使我们很难在这个层次上有效思考（尤其使用relaxed内存模型）。

<br/>
### **CP.102:仔细研究文献**

**原因** 除了原子和其他一些标准模式外，无锁编程实际上是一个专家话题。在给别人编写无锁代码前先成为专家。

**参考资料**
- Anthony Williams: C++ concurrency in action. Manning Publications.
- Boehm, Adve, You Don’t Know Jack About Shared Variables or Memory Models , Communications of the ACM, Feb 2012.
- Boehm, “Threads Basics”, HPL TR 2009-259.
- Adve, Boehm, “Memory Models: A Case for Rethinking Parallel Languages and Hardware”, Communications of the ACM, August 2010.
- Boehm, Adve, “Foundations of the C++ Concurrency Memory Model”, PLDI 08.
- Mark Batty, Scott Owens, Susmit Sarkar, Peter Sewell, and Tjark Weber, “Mathematizing C++ Concurrency”, POPL 2011.
- Damian Dechev, Peter Pirkelbauer, and Bjarne Stroustrup: Understanding and Effectively Preventing the ABA Problem in Descriptor-based Lock-free Designs. 13th IEEE Computer Society ISORC 2010 Symposium. May 2010.
- Damian Dechev and Bjarne Stroustrup: Scalable Non-blocking Concurrent Objects for Mission Critical Code. ACM OOPSLA’09. October 2009
- Damian Dechev, Peter Pirkelbauer, Nicolas Rouquette, and Bjarne Stroustrup: Semantically Enhanced Containers for Concurrent Real-Time Systems. Proc. 16th Annual IEEE International Conference and Workshop on the Engineering of Computer Based Systems (IEEE ECBS). April 2009.
- Maurice Herlihy, Nir Shavit, Victor Luchangco, Michael Spear, “The Art of Multiprocessor Programming”, 2nd ed. September 2020

<br/>
### **CP.110:不要为初始化编写双重检查锁**

**原因** C++11开始，static局部变量的初始化是线程安全实现的。当与RAII模式结合时，static局部变量可以取代自己编写双重检查初始化。std::call\_once同样也可以达到目的。

**Example** 
```cpp
void f()
{
    static std::once_flag my_once_flag;
    std::call_once(my_once_flag, []()
    {
        // do this only once
    });
    // ...
}
```
或者
```cpp
void f()
{
    // Assuming the compiler is compliant with C++11
    static My_class my_object; // Constructor called only once
    // ...
}

class My_class
{
public:
    My_class()
    {
        // do this only once
    }
};
```

<br/>
### **CP.111:如果你真的需要双重检查锁，请使用常规模式**

**Example,bad**
```cpp
mutex action_mutex;
volatile bool action_needed;

if (action_needed) {
    std::lock_guard<std::mutex> lock(action_mutex);
    if (action_needed) {
        take_action();
        action_needed = false;
    }
}
```

**Example,good**
```cpp
mutex action_mutex;
atomic<bool> action_needed;

if (action_needed) {
    std::lock_guard<std::mutex> lock(action_mutex);
    if (action_needed) {
        take_action();
        action_needed = false;
    }
}
```
以下更好，fine-tuned内存顺序比顺序一致的内存顺序，对于acquire load更有益。
```cpp
mutex action_mutex;
atomic<bool> action_needed;

if (action_needed.load(memory_order_acquire)) {
    lock_guard<std::mutex> lock(action_mutex);
    if (action_needed.load(memory_order_relaxed)) {
        take_action();
        action_needed.store(false, memory_order_release);
    }
}
```

<br/>
## **CP.etc:Etc.并发规则**

- [CP.200:非C++内存时使用volatile](#cp200非c内存时使用volatile)
- [CP.201:Signals](#cp201signals)

<br/>
### **CP.200:非C++内存时使用volatile**

**原因** volatile是用来指与"非C++"代码或不遵循C++内存模型的硬件共享的对象。

**Example**
```cpp
const volatile long clock;
```
这描述了一个由时钟电路持续更新的寄存器。时钟是易变的，使用它的C++程序没有任何动作情况下，它的值也会发生变化。\
例如，读取两次时钟往往会产生两个不同的值，所以优化器最好不要在这段代码中优化掉第二次读取。 
```cpp
long t1 = clock;
// ... no use of clock here ...
long t2 = clock;
```
clock是const的因为程序不会尝试往clock中写。

**注意** 除非你正在编写直接操作硬件的最底层的代码，将volatile视为最好可以避免的深奥问题。

**Example** 通常情况下，C++代码会收到属于其他地方（硬件或其他语言）的volatile内存。
```cpp
int volatile* vi = get_hardware_memory_location();
    // note: we get a pointer to someone else's memory here
    // volatile says "treat this with extra respect"
```
有时，C++代码分配了volatile内存，并通过转义指针与其他地方（硬件或其他语言）分享。
```cpp
static volatile long vl;
please_use_this(&vl);   // escape a reference to this to "elsewhere" (not C++)
```

**Example, bad**

volatile局部变量几乎总是错误的。如果它们是短暂的，那怎么可能与其他硬件或语言共享？这一点原因同样适用于成员变量。
```cpp
void f()
{
    volatile int i = 0; // bad, volatile local variable
    // etc.
}

class My_type {
    volatile int i = 0; // suspicious, volatile member variable
    // etc.
};
```

**注意** C++与其他语言不同，volatile与同步没有关系。

**Enforcement** 所有volatile的局部变量或成员变量都应该标记出，几乎可以肯定的是，可以使用atomic<T>来替代。

<br/>
### **CP.201:Signals**

可能更值得提醒的是很少代码是异步信号安全的，以及如何与信号处理函数交互。（最好的可能是根本不需要）

<br/>
### **F.60:当no argument也是一个选项时，优先使用T\*而不是T&**

原因：指针可能为空nullptr，但引用不会，不存在空引用。有时用nullptr来代表no object，如果不是这样，用引用会让代码更简单。

Example
```cpp
string zstring_to_string(zstring p) // zstring is a char*; that is a C-style string
{
    if (!p) return string{};    // p might be nullptr; remember to check
    return string{p};
}

void print(const vector<int>& r)
{
    // r refers to a vector<int>; no check needed
}
```

<br/>
# **Con:常量和不变性**

你不可能在一个常数上有一个race condition。当许多对象不能改变它们的值时，对程序进行推理会更容易。承诺"不变"作为参数传递的对象的接口大大增加了可读性。

常量规则摘要：
- [Con.1:默认情况下，使对象成为不可变的](#con1默认情况下使对象成为不可变的)
- [Con.2:默认情况下，使成员函数成为const](#con2默认情况下使成员函数成为const)
- [Con.3:默认情况下，将指针和引用传递为const](#con3默认情况下将指针和引用传递为const)
- [Con.4:使用const来定义对象，其值在构造后不会改变](#con4使用const来定义对象其值在构造后不会改变)
- [Con.5:对可以在编译时计算的值使用constexpr](#con5对可以在编译时计算的值使用constexpr)

<br/>
### **Con.1:默认情况下，使对象成为不可变的**

**原因** 不可变的对象更容易推理，所以只有在需要改变对象的值时才使其成为非const；防止意外或难以察觉的值改变。

**Example**
```cpp
for (const int i : c) cout << i << '\n';    // just reading: const

for (int i : c) cout << i << '\n';          // BAD: just reading
```

**例外** 通过值传递的函数参数很少被改变，但也很少声明为const。为了避免混乱和大量的误报，不要对函数参数执行这一规则。
```cpp
void f(const char* const p); // pedantic
void g(const int i) { ... }  // pedantic
```
注意，函数参数是一个局部变量，所以对它的改变是局部的。

**Enforcement** 不被修改的非const变量（参数除外，以避免许多误报）要标记出。

<br/>
### **Con.2:默认情况下，使成员函数成为const**

**原因** 一个成员函数应该被标记为const，除非它改变了对象的可观察状态。这给了设计意图一个更精确的声明，更好的可读性，更多的错误被编译器捕获，有时还有更多的优化机会。

**Example, bad**
```cpp
class Point {
    int x, y;
public:
    int getx() { return x; }    // BAD, should be const as it doesn't modify the object's state
    // ...
};

void f(const Point& pt)
{
    int x = pt.getx();          // ERROR, doesn't compile because getx was not marked const
}
```

**注意** 将指针或引用传递给non-const值本身并不坏，但只有在被调用的函数应该修改对象时才应该这样做。代码的阅读者必须假设一个接受"普通"T\*或T\&的函数会修改被引用的对象。
如果它现在不这样做，以后可能会这样做，而不会强迫重新编译。

**注意** 有一些代码/库提供了声明T\*的函数，尽管这些函数并不修改该T，这对正在进行代码现代化的人来说是个问题。你可以:
- 更新为const，首选的长期解决方案
- 把const cast掉，最好避免
- 提供一个封装函数
Example:
```cpp
void f(int* p);   // old code: f() does not modify `*p`
void f(const int* p) { f(const_cast<int*>(p)); } // wrapper
```
注意，这个封装方案是一个补丁，只有在不能修改f()的声明时才可以使用，例如，因为它在一个你不能修改的库中。

**注意** 一个const成员函数可以修改一个对象的值，该对象是可变的或通过指针成员访问。一个常见的用途是维护一个缓存，而不是重复做一个复杂的计算。例如，
```cpp
class Date {
public:
    // ...
    const string& string_ref() const
    {
        if (string_val == "") compute_string_rep();
        return string_val;
    }
    // ...
private:
    void compute_string_rep() const;    // compute string representation and place it in string_val
    mutable string string_val;
    // ...
};
```
另一种说法是，constness不是传递性的。一个常量成员函数有可能改变可变成员的值和通过非常量指针访问的对象的值。
类的工作是确保这种改变只在根据它提供给用户的语义（不变量）有意义的时候进行。

另见[pimpl](#i27为了稳定的库abi考虑用pimpl)

**Enforcement** 一个没有标明const的成员函数，但它没有对任何成员变量进行非const操作要标记出。 

<br/>
### **Con.3:默认情况下，将指针和引用传递为const**

**原因** 为了避免被调用的函数意外地改变数值。当被调用的函数不修改状态时，对程序进行推理就容易得多。

**Example**
```cpp
void f(char* p);        // does f modify *p? (assume it does)
void g(const char* p);  // g does not modify *p
```

**注意** 将指针或引用传递给非const本身并不是坏事，但这应该只在被调用的函数要修改对象时进行。

**Enforcement** 
- 一个不修改由指针或引用传递给non-const的对象的函数要标记出。
- 一个函数（使用转换）修改一个由指针或引用传递给const的对象要标记出。

<br/>
### **Con.4:使用const来定义对象，其值在构造后不会改变**

**原因** 防止意外改变的对象值。

**Example**
```cpp
void f()
{
    int x = 7;
    const int y = 9;

    for (;;) {
        // ...
    }
    // ...
}
```
由于x不是const，我们必须假设它在循环的某个地方被修改了。

**Enforcement** 未修改的non-const变量要标记出。

<br/>
### **Con.5:对可以在编译时计算的值使用constexpr**

更好的性能，更好的编译时检查，保证编译时评估，没有race condition的可能性。

**Example**
```cpp
double x = f(2);            // possible run-time evaluation
const double y = f(2);      // possible run-time evaluation
constexpr double z = f(2);  // error unless f(2) can be evaluated at compile time
```

另见[F.4](#f4如果函数能在编译时求值那应该声明为constexpr)

**Enforcement**
- 使用常量表达式初始值设定const定义的要标记出。

<br/>
# **T:模版和泛型编程**

泛型编程是使用由类型、值和算法参数化的类型和算法进行编程。在C++中，template语言机制支持泛型编程。

泛型函数的参数的特征在于对所涉及的参数类型和值的要求集。在C++中，这些要求由称为concept的编译时谓词表示。

模板也可用于元编程；也就是说，在编译时组成代码的程序。

泛型编程的一个核心概念是“concept”；也就是说，对作为编译时谓词呈现的模板参数的要求。 “concept”在C++20中被标准化，尽管它们在GCC 6.1中首次以稍旧的语法提供。

模板使用规则摘要:
- [T.1:使用模板提高代码的抽象层次](#t1使用模板提高代码的抽象层次)
- [T.2:使用模板来表达适用于多种参数类型的算法](#t2使用模板来表达适用于多种参数类型的算法)
- [T.3:使用模板来表达容器和范围](#t3使用模板来表达容器和范围)
- [T.4:使用模板来表达语法树操作](#t4使用模板来表达语法树操作)
- [T.5:结合通用技术和OO技术来增强它们的优势，而不是它们的成本](#t5结合通用技术和oo技术来增强它们的优势而不是它们的成本)

Concept使用规则摘要:
- [T.10:指定所有模板参数的概念](#t10指定所有模板参数的概念)
- [T.11:尽可能使用标准概念](#t11尽可能使用标准概念)
- [T.12:对于局部变量，优先使用概念名称而不是auto](#t12对于局部变量优先使用概念名称而不是auto)
- [T.13:对于简单的、单一类型的参数概念，更倾向于使用速记符号](#t13对于简单的单一类型的参数概念更倾向于使用速记符号)

Concept定义规则摘要:
- [T.20:避免没有有意义语义的“概念”](#t20避免没有有意义语义的概念)
- [T.21:要求一个概念有完整的操作集](#t21要求一个概念有完整的操作集)
- [T.22:为概念指定公理](#t22为概念指定公理)
- [T.23:通过添加新的使用模式将精炼的概念与更普遍的情况区分开来](#t23通过添加新的使用模式将精炼的概念与更普遍的情况区分开来)
- [T.24:使用标记类或特征来区分仅在语义上不同的概念](#t24使用标记类或特征来区分仅在语义上不同的概念)
- [T.25:避免互补约束](#t25避免互补约束)
- [T.26:优先根据使用模式而不是简单的语法来定义概念](#t26优先根据使用模式而不是简单的语法来定义概念)
- [T.30:谨慎使用概念否定(\!C\<T\>) 来表达微小差异](#t30谨慎使用概念否定ct-来表达微小差异)
- [T.31:谨慎使用概念分离(C1\<T\>\|\|C2\<T\>)来表达备选方案](#t31谨慎使用概念分离c1t--c2t来表达备选方案)

模板接口规则摘要:
- [T.40:使用函数对象将操作传递给算法](#t40使用函数对象将操作传递给算法)
- [T.41:只需要模板概念中的基本属性](#t41只需要模板概念中的基本属性)
- [T.42:使用模板别名来简化符号并隐藏实现细节](#t42使用模板别名来简化符号并隐藏实现细节)
- [T.43:优先使用using而不是typedef来定义别名](#t43优先使用using而不是typedef来定义别名)
- [T.44:使用函数模板推导类模板参数类型（在可行的情况下）](#t44使用函数模板推导类模板参数类型在可行的情况下)
- [T.46:要求模板参数至少是semiregular的](#t46要求模板参数至少是semiregular的)
- [T.47:避免具有通用名称的高度可见的无约束模板](#t47避免具有通用名称的高度可见的无约束模板)
- [T.48:如果你的编译器不支持概念，用enable\_if伪装它们](#t48如果你的编译器不支持概念用enable_if伪装它们)
- [T.49:尽可能避免类型擦除](#t49尽可能避免类型擦除)

模板定义规则摘要：
- [T.60:最小化模板的上下文依赖](#t60最小化模板的上下文依赖)
- [T.61:不要过度参数化成员（SCARY）](#t61不要过度参数化成员scary)
- [T.62:将非依赖类模板成员放在非模板化基类中](#t62将非依赖类模板成员放在非模板化基类中)
- [T.64:使用特例化来提供类模板的替代实现](#t64使用特例化来提供类模板的替代实现)
- [T.65:使用标签分派来提供功能的替代实现](#t65使用标签分派来提供功能的替代实现)
- [T.67:使用特例化为不规则类型提供替代实现](#t67使用特例化为不规则类型提供替代实现)
- [T.68:在模板中使用\{\}而不是()以避免歧义](#t68在模板中使用而不是以避免歧义)
- [T.69:在模板内，不要进行不合格的非成员函数调用，除非您打算将其作为自定义点](#t69在模板内不要进行不合格的非成员函数调用除非您打算将其作为自定义点)

模板和层次结构规则摘要：
- [T.80:不要天真地模板化类层次结构](#t80不要天真地模板化类层次结构)
- [T.81:不要混合层次结构和数组](#t81不要混合层次结构和数组)
- [T.82:当虚拟函数不可取时，要线性化层次结构](#t82当虚拟函数不可取时要线性化层次结构)
- [T.83:不要将成员函数模板声明为virtual](#t83不要将成员函数模板声明为virtual)
- [T.84:使用非模板核心实现来提供ABI稳定的接口](#t84使用非模板核心实现来提供abi稳定的接口)

Variadic模板规则摘要：
- [T.100:当你需要一个函数接受各种类型的可变数量的参数时，使用可变参数模板](#t100当你需要一个函数接受各种类型的可变数量的参数时使用可变参数模板)
- [T.101:如何将参数传递给可变参数模板](#t101如何将参数传递给可变参数模板)
- [T.102:如何处理可变参数模板的参数](#t102如何处理可变参数模板的参数)
- [T.103:不要对同类参数列表使用可变参数模板](#t103不要对同类参数列表使用可变参数模板)

元编程规则摘要：
- [T.120:只有在真正需要时才使用模板元编程](#t120只有在真正需要时才使用模板元编程)
- [T.121:主要使用模板元编程来模拟概念](#t121主要使用模板元编程来模拟概念)
- [T.122:使用模板（通常是模板别名）在编译时计算类型](#t122使用模板通常是模板别名在编译时计算类型)
- [T.123:使用constexpr函数在编译时计算值](#t123使用constexpr函数在编译时计算值)
- [T.124:优先使用标准库TMP设施](#t124优先使用标准库tmp设施)
- [T.125:如果您需要超越标准库TMP设施，请使用现有库](#t125如果您需要超越标准库tmp设施请使用现有库)

其他模版规则摘要：
- [T.140:如果一个操作可以重用，给它一个名字](#t140如果一个操作可以重用给它一个名字)
- [T.141:如果只在一个地方需要一个简单的函数对象，请使用未命名的lambda](#t141如果只在一个地方需要一个简单的函数对象请使用未命名的lambda)
- [T.142:使用模板变量来简化符号TMP设施](#t142使用模板变量来简化符号tmp设施)
- [T.143:不要无意中编写非泛型代码](#t143不要无意中编写非泛型代码)
- [T.144:不要特化函数模板](#t144不要特化函数模板)
- [T.150:使用static\_assert检查类是否匹配概念](#t150使用static_assert检查类是否匹配概念)

<br/>
## **T.gp:泛型编程**

泛型编程是使用由类型、值和算法参数化的类型和算法进行编程。

<br/>
### **T.1:使用模板提高代码的抽象层次**

**原因** 通用性。重复使用。效率。鼓励对用户类型的一致定义。

**Example, bad** 从概念上讲，以下要求是错误的，因为我们想要的T不仅仅是“可以递增”或“可以添加”的非常低级的概念：
```cpp
template<typename T>
    requires Incrementable<T>
T sum1(vector<T>& v, T s)
{
    for (auto x : v) s += x;
    return s;
}

template<typename T>
    requires Simple_number<T>
T sum2(vector<T>& v, T s)
{
    for (auto x : v) s = s + x;
    return s;
}
```
假设Incrementable不支持+，Simple\_number不支持+=，我们对sum1和sum2的实现者进行了过度约束。而且，在这种情况下，错过了一个泛化的机会。

<br/>
### **T.2:使用模板来表达适用于多种参数类型的算法**

**原因** 通用性。尽量减少源代码的数量。互操作性。复用。

**Example** 这就是STL的基础。一个单一的find算法很容易适用于任何种类的input range。
```cpp
template<typename Iter, typename Val>
    // requires Input_iterator<Iter>
    //       && Equality_comparable<Value_type<Iter>, Val>
Iter find(Iter b, Iter e, Val v)
{
    // ...
}
```

**注意** 除非你确实需要不止一种模板参数类型，不要使用模板。不要过度抽象。

<br/>
### **T.3:使用模板来表达容器和范围**

**注意** 容器需要一个元素类型，将其表示为模板参数是通用的、可重用的和类型安全的。它还避免了脆弱或低效的解决方法。惯例，这就是STL的做法。

**Example**
```cpp
template<typename T>
    // requires Regular<T>
class Vector {
    // ...
    T* elem;   // points to sz Ts
    int sz;
};

Vector<double> v(10);
v[7] = 9.9;
```

**Example, bad**
```cpp
class Container {
    // ...
    void* elem;   // points to size elements of some type
    int sz;
};

Container c(10, sizeof(double));
((double*) c.elem)[7] = 9.9;
```
这并不直接表达程序员的意图，而且对类型系统和优化器隐藏了程序的结构。

将void\*隐藏在宏后面只是掩盖了问题，并引入了新的混淆的机会。

**例外** 如果你需要一个ABI稳定的接口，你可能需要提供一个基础实现，并以该实现来表达（类型安全）模板。[T.84](#t84)

**Enforcement** 在low-level实现代码之外使用void\*和cast要标记出。

<br/>
### **T.4使用模板来表达语法树操作**

<br/>
### **T.5:结合通用技术和OO技术来增强它们的优势，而不是它们的成本**

**原因** 泛型和OO技术是互补的。

**Example** 静态帮助动态：使用静态多态来实现动态多态接口。
```cpp
class Command {
    // pure virtual functions
};

// implementations
template</*...*/>
class ConcreteCommand : public Command {
    // implement virtuals
};
```

**Example** 动态帮助静态：提供一个通用的、舒适的、静态绑定的接口，但在内部动态调度，所以你提供了一个统一的对象布局。示例包括与std::shared\_ptr的deleter一样的类型擦除（但不要过度使用类型擦除）。
```cpp
#include <memory>

class Object {
public:
    template<typename T>
    Object(T&& obj)
        : concept_(std::make_shared<ConcreteCommand<T>>(std::forward<T>(obj))) {}

    int get_id() const { return concept_->get_id(); }

private:
    struct Command {
        virtual ~Command() {}
        virtual int get_id() const = 0;
    };

    template<typename T>
    struct ConcreteCommand final : Command {
        ConcreteCommand(T&& obj) noexcept : object_(std::forward<T>(obj)) {}
        int get_id() const final { return object_.get_id(); }

    private:
        T object_;
    };

    std::shared_ptr<Command> concept_;
};

class Bar {
public:
    int get_id() const { return 1; }
};

struct Foo {
public:
    int get_id() const { return 2; }
};

Object o(Bar{});
Object o2(Foo{});
```

**注意** 在类模板中，非虚拟函数只有在被使用时才会被实例化，但虚拟函数每次都会被实例化。这可能会使代码规模膨胀，并可能通过实例化从未需要的功能来过度限制泛型。避免这种情况，即使是标准库面也犯了这个错误。 

<br/>
## **T.concepts:Concept规则**

Concepts是一个C++20设施，用于指定模板参数的要求。它们对于泛型编程的思考以及未来C++库（标准和其他）的大量工作的基础至关重要。

Concept使用规则摘要：
- [T.10:指定所有模板参数的概念](#t10指定所有模板参数的概念)
- [T.11:尽可能使用标准概念](#t11尽可能使用标准概念)
- [T.12:对于局部变量，优先使用概念名称而不是auto](#t12对于局部变量优先使用概念名称而不是auto)
- [T.13:对于简单的、单一类型的参数概念，更倾向于使用速记符号](#t13对于简单的单一类型的参数概念更倾向于使用速记符号)

Concept定义规则摘要:
- [T.20:避免没有有意义语义的“概念”](#t20避免没有有意义语义的概念)
- [T.21:要求一个概念有完整的操作集](#t21要求一个概念有完整的操作集)
- [T.22:为概念指定公理](#t22为概念指定公理)
- [T.23:通过添加新的使用模式将精炼的概念与更普遍的情况区分开来](#t23通过添加新的使用模式将精炼的概念与更普遍的情况区分开来)
- [T.24:使用标记类或特征来区分仅在语义上不同的概念](#t24使用标记类或特征来区分仅在语义上不同的概念)
- [T.25:避免互补约束](#t25避免互补约束)
- [T.26:优先根据使用模式而不是简单的语法来定义概念](#t26优先根据使用模式而不是简单的语法来定义概念)
- [T.30:谨慎使用概念否定(!C\<T\>) 来表达微小差异](#t30谨慎使用概念否定Ct来表达微小差异)
- [T.31:谨慎使用概念分离(C1\<T\> || C2\<T\>)来表达备选方案](#t31谨慎使用概念析取c1tc2t来表达备选方案)

<br/>
## **T.con-use:Concept使用**

<br/>
### **T.10:指定所有模板参数的概念**

**原因** 正确性和可读性。模板参数的假定含义（语法和语义）是模板接口的基础。一个concept可以极大地改善模板的文档和错误处理。为模板参数指定concept是一个强大的设计工具。 

**Example**
```cpp
template<typename Iter, typename Val>
    requires input_iterator<Iter>
             && equality_comparable_with<iter_value_t<Iter>, Val>
Iter find(Iter b, Iter e, Val v)
{
    // ...
}
```
或等效且更简洁:
```cpp
template<input_iterator Iter, typename Val>
    requires equality_comparable_with<iter_value_t<Iter>, Val>
Iter find(Iter b, Iter e, Val v)
{
    // ...
}
```

**注意** 普通的typename（或auto）是约束性最小的概念。只有在只能假定“它是一种类型”时，才应该很少使用它。
通常只有在（作为模板元编程代码的一部分）我们操作纯表达式树，推迟类型检查时才需要这样做。

**Enforcement** 模板类型参数没有concept要标记出。

<br/>
### **T.11:尽可能使用标准概念**

**原因** “标准”concept（由GSL和ISO标准本身提供）节省了我们思考自己concept的工作，比我们匆忙完成的要好，并且提高了互操作性。

**注意** 除非你正在创建一个新的通用库，否则你需要的大部分概念都已经由标准库定义了。

**Example**
```cpp
template<typename T>
    // don't define this: sortable is in <iterator>
concept Ordered_container = Sequence<T> && Random_access<Iterator<T>> && Ordered<Value_type<T>>;

void sort(Ordered_container auto& s);
```
这个Ordered\_container是相当合理的，但它与标准库中的sortable概念非常相似。它更好吗？它是正确的吗？它是否准确地反映了标准中对排序的要求？直接使用sortable会更好更简单。
```cpp
void sort(sortable auto& s);   // better
```

**注意** 随着我们接近包含concept的ISO标准，“标准”concept集也在不断发展。

**注意** 设计一个有用的concept是具有挑战性的。

**Enforcement** 很难。
- 寻找不受约束的参数，使用 "不寻常"/非标准概念的模板，使用没有公理的"自制"concept的模板。
- 开发一个发现concept的工具。

<br/>
### **T.12:对于局部变量，优先使用概念名称而不是auto**

**原因** auto是最弱的concept。concept名字比auto更表达意义。

**Example**
```cpp
vector<string> v{ "abc", "xyz" };
auto& x = v.front();        // bad
String auto& s = v.front(); // good (String is a GSL concept)
```

<br/>
### **T.13:对于简单的、单一类型的参数概念，更倾向于使用速记符号**

**原因** 可读性。直接表达一个想法。

**Example** To say “T is sortable”:
```cpp
template<typename T>       // Correct but verbose: "The parameter is
    requires sortable<T>   // of type T which is the name of a type
void sort(T&);             // that is sortable"

template<sortable T>       // Better: "The parameter is of type T
void sort(T&);             // which is Sortable"

void sort(sortable auto&); // Best: "The parameter is Sortable"
```
较短的版本更符合我们说话的方式。请注意，许多模板不需要使用模板关键字。

**Enforcement**
- 当人们从\<typename T\>和\<class T\>符号转换时，短期内不可行。
- 标记声明首先引入typename，然后使用简单的单一类型参数concept对其进行约束。

<br/>
# **T.concepts.def:Concept定义规则**

定义好的concept并不简单。Concept是为了代表一个应用领域的基本概念（因此被称为concept）。同样地，把一组句法约束扔在一起，用于一个单一的类或算法的参数，并不是concept的设计目的，也不会带来该机制的全部好处。

显然，定义concept对于可以使用实现（例如，C++20 或更高版本）的代码最有用，但定义concept本身就是一种有用的设计技术，有助于捕获概念性错误并清理实现的概念（原文如此！）。

- [T.20:避免没有有意义语义的“概念”](#t20避免没有有意义语义的概念)
- [T.21:要求一个概念有完整的操作集](#t21要求一个概念有完整的操作集)
- [T.22:为概念指定公理](#t22为概念指定公理)
- [T.23:通过添加新的使用模式将精炼的概念与更普遍的情况区分开来](#t23通过添加新的使用模式将精炼的概念与更普遍的情况区分开来)
- [T.24:使用标记类或特征来区分仅在语义上不同的概念](#t24使用标记类或特征来区分仅在语义上不同的概念)
- [T.25:避免互补约束](#t25避免互补约束)
- [T.26:优先根据使用模式而不是简单的语法来定义概念](#t26优先根据使用模式而不是简单的语法来定义概念)
- [T.30:谨慎使用概念否定(!C\<T\>) 来表达微小差异](#t30谨慎使用概念否定Ct来表达微小差异)
- [T.31:谨慎使用概念分离(C1\<T\> || C2\<T\>)来表达备选方案](#t31谨慎使用概念析取c1tc2t来表达备选方案)

<br/>
### **T.20:避免没有有意义语义的“概念”**

**原因** Concept旨在表达语义概念，例如“数字”、“一系列”元素和“完全有序”。简单的约束，例如“有一个+运算符”和“有一个\>运算符”不能孤立地有意义地指定，应该只用作有意义concept的构建块，而不是在用户代码中。

**Example, bad**
```cpp
template<typename T>
// bad; insufficient
concept Addable = requires(T a, T b) { a+b; };

template<Addable N>
auto algo(const N& a, const N& b) // use two numbers
{
    // ...
    return a + b;
}

int x = 7;
int y = 9;
auto z = algo(x, y);   // z = 16

string xx = "7";
string yy = "9";
auto zz = algo(xx, yy);   // zz = "79"
```
也许串联是意料之中的事。更有可能的是，它是一个意外。以等价的方式定义减法，会得到截然不同的可接受类型集。这个Addable违反了数学规则，即加法应该是交换性的：a+b == b+a。

**注意** 指定有意义的语义的能力是真正concept的定义特征，而不是句法约束。

**Example**
```cpp
template<typename T>
// The operators +, -, *, and / for a number are assumed to follow the usual mathematical rules
concept Number = requires(T a, T b) { a+b; a-b; a*b; a/b; };

template<Number N>
auto algo(const N& a, const N& b)
{
    // ...
    return a + b;
}

int x = 7;
int y = 9;
auto z = algo(x, y);   // z = 16

string xx = "7";
string yy = "9";
auto zz = algo(xx, yy);   // error: string is not a Number
```

**注意** 具有多种操作的概念比单一操作的概念意外地匹配一个类型的机会要低得多。

**Enforcement** 
- 在其他concept的定义之外使用单一操作的concept要标记出。
- enable\_if的使用，似乎是为了模拟单一操作的concept要标记出。

<br/>
### **T.21:要求一个概念有完整的操作集**

**原因** 易于理解。改善互操作性。有助于实施者和维护者。

**Example, bad**
```cpp
template<typename T> concept Subtractable = requires(T a, T b) { a-b; };
```
这在语义上没有意义。你至少需要+来使-有意义和有用。
完整集合例子：
- Arithmetic: +, -, \*, /, +=, -=, \*=, /=
- Comparable: <, >, <=, >=, ==, !=

**注意** 无论我们是否对concept使用直接的语言支持，他的规则都适用。这是一条通用的设计规则，甚至适用于非模板。
```cpp
class Minimal {
    // ...
};

bool operator==(const Minimal&, const Minimal&);
bool operator<(const Minimal&, const Minimal&);

Minimal operator+(const Minimal&, const Minimal&);
// no other operators

void f(const Minimal& x, const Minimal& y)
{
    if (!(x == y)) { /* ... */ }    // OK
    if (x != y) { /* ... */ }       // surprise! error

    while (!(x < y)) { /* ... */ }  // OK
    while (x >= y) { /* ... */ }    // surprise! error

    x = x + y;          // OK
    x += y;             // surprise! error
}
```
这是最小的，但对用户来说是令人惊讶和制约的。它甚至可能是更低的效率。
该规则支持这样的观点：一个concept应该反映一套（数学上）连贯的操作。

**Example**
```cpp
class Convenient {
    // ...
};

bool operator==(const Convenient&, const Convenient&);
bool operator<(const Convenient&, const Convenient&);
// ... and the other comparison operators ...

Convenient operator+(const Convenient&, const Convenient&);
// ... and the other arithmetic operators ...

void f(const Convenient& x, const Convenient& y)
{
    if (!(x == y)) { /* ... */ }    // OK
    if (x != y) { /* ... */ }       // OK

    while (!(x < y)) { /* ... */ }  // OK
    while (x >= y) { /* ... */ }    // OK

    x = x + y;     // OK
    x += y;        // OK
}
```
定义所有运算符可能很麻烦，但并不难。理想情况下，该规则应该是默认为您提供比较运算符的语言支持的。

**Enforcement**
- 标记那些支持一组运算符的 "奇数 "子集的类，例如，==但不支持！=，或者+但不支持-。是的，std::string是 "奇怪的"，但现在改变已经太晚了。

<br/>
### **T.22:为概念指定公理**

**原因** 有意义/有用的概念具有语义意义。以非正式、半正式或正式的方式表达这些语义可以使读者理解概念，并且表达它的努力可以捕捉概念错误。指定语义是一个强大的设计工具。 

**Example**
```cpp
template<typename T>
    // The operators +, -, *, and / for a number are assumed to follow the usual mathematical rules
    // axiom(T a, T b) { a + b == b + a; a - a == 0; a * (b + c) == a * b + a * c; /*...*/ }
    concept Number = requires(T a, T b) {
        {a + b} -> convertible_to<T>;
        {a - b} -> convertible_to<T>;
        {a * b} -> convertible_to<T>;
        {a / b} -> convertible_to<T>;
    };
```

**注意** 这是数学意义上的公理：无需证明就可以假设的东西。一般来说，公理是无法证明的，即使是证明，也往往超出了编译器的能力。一个公理可能并不普遍，但模板编写者可以假设它对所有实际使用的输入都是成立的（类似于一个前提条件）。

**注意** 在这种情况下，公理是布尔表达式。请看Palo Alto TR的例子。目前，C++不支持公理（即使是ISO Concepts TS），所以我们不得不在很长一段时间内使用注释。一旦有了语言支持，公理前面的//就可以去掉了。

**注意** GSL概念有明确的语义，见Palo Alto TR和Ranges TS。

**例外** 仍在开发中的新 "概念 "的早期版本往往只是定义了简单的约束集，而没有明确的语义。找到好的语义需要努力和时间。一套不完整的约束条件仍然可以非常有用。
```cpp
// balancer for a generic binary tree
template<typename Node> concept Balancer = requires(Node* p) {
    add_fixup(p);
    touch(p);
    detach(p);
};
```
因此，一个Balancer必须在树节点上至少提供这些操作，但我们还没有准备好指定详细的语义，因为一种新的平衡树可能需要更多的操作，而且在设计的早期阶段，所有节点的精确的一般语义很难确定。
一个不完整的或没有明确语义的 "概念 "仍然可以是有用的。例如，它允许在初始实验期间进行一些检查。然而，它不应该被认为是稳定的。每个新的用例都可能需要对这样一个不完整的概念进行改进。

**Enforcement**
- 在概念定义的注释中寻找"公理"一词

<br/>
### **T.23:通过添加新的使用模式将精炼的概念与更普遍的情况区分开来**

**原因** 否则，编译器无法自动区分它们。

**Example**
```cpp
template<typename I>
// Note: input_iterator is defined in <iterator>
concept Input_iter = requires(I iter) { ++iter; };

template<typename I>
// Note: forward_iterator is defined in <iterator>
concept Fwd_iter = Input_iter<I> && requires(I iter) { iter++; };
```
编译器可以根据所需操作的集合（这里是后缀++）来确定细化。这减少了这些类型的实现者的负担，因为他们不需要任何特殊的声明来 "钩住这个概念"。如果两个概念有完全相同的要求，它们在逻辑上是等同的（没有细化）。

**Enforcement**
- 标记一个与另一个已经看到的概念具有完全相同要求的概念（都没有更精炼）。要消除它们的歧义，见T.24。

<br/>
### **T.24:使用标记类或特征来区分仅在语义上不同的概念**

**原因** 两个需要相同语法但具有不同语义的概念会导致歧义，除非程序员对它们进行区分。

**Example**
```cpp
template<typename I>    // iterator providing random access
// Note: random_access_iterator is defined in <iterator>
concept RA_iter = ...;

template<typename I>    // iterator providing random access to contiguous data
// Note: contiguous_iterator is defined in <iterator>
concept Contiguous_iter =
    RA_iter<I> && is_contiguous_v<I>;  // using is_contiguous trait
```
程序员（在库中）必须适当地定义is\_contiguous（特征）。
将一个标签类包裹到一个概念中，可以更简单地表达这个想法。
```cpp
template<typename I> concept Contiguous = is_contiguous_v<I>;

template<typename I>
concept Contiguous_iter = RA_iter<I> && Contiguous<I>;
```
程序员（在库中）必须适当地定义is\_contiguous（特征）。

**注意** 特征可以是特征类或类型特征。这些可以是用户定义的或标准库的。优先使用标准库的。

**Enforcement**
- 编译器会标记对相同概念的模糊使用。
- 标记相同概念的定义。

<br/>
### **T.25:避免互补约束**

**原因** 清晰度。可维护性。用否定句表达的具有互补要求的函数是很脆弱的。

**Example** 最初，人们会尝试用互补的要求来定义功能。
```cpp
template<typename T>
    requires !C<T>    // bad
void f();

template<typename T>
    requires C<T>
void f();
```
以下更优：
```cpp
template<typename T>   // general template
    void f();

template<typename T>   // specialization by concept
    requires C<T>
void f();
```
只有当C\<T\>不满足时，编译器才会选择无约束的模板。如果你不想（或不能）定义f()的无约束版本，那么就删除它。
```cpp
template<typename T>
void f() = delete;
```
编译器将选择重载版本，或发出适当的错误。

**注意** 不幸的是，互补性约束在enable\_if代码中很常见。
```cpp
template<typename T>
enable_if<!C<T>, void>   // bad
f();

template<typename T>
enable_if<C<T>, void>
f();
```

**注意** 对一项要求的互补性要求有时（错误地）被认为是可管理的。然而，对于两个或更多需求，定义需求的数量可能呈指数增长（2、4、8、16，……）
```cpp
C1<T> && C2<T>
!C1<T> && C2<T>
C1<T> && !C2<T>
!C1<T> && !C2<T>
```
现在，出错的机会成倍增加。

**Enforcement**
- 具有C\<T\>和!C\<T\>约束的函数对要标记出。

<br/>
### **T.26:优先根据使用模式而不是简单的语法来定义概念**

**原因** 该定义更具可读性，直接对应于用户要写的内容。转换被考虑到了。你不需要记住所有类型特征的名称。

**Example** 你可能很想这样定义一个概念Equality:
```cpp
template<typename T> concept Equality = has_equal<T> && has_not_equal<T>;
```
显然，使用标准的equality\_comparable会更好、更容易，但只是作为一个例子，如果你必须定义这样一个概念，更好的是：
```cpp
template<typename T> concept Equality = requires(T a, T b) {
    { a == b } -> std::convertible_to<bool>;
    { a != b } -> std::convertible_to<bool>;
    // axiom { !(a == b) == (a != b) }
    // axiom { a = b; => a == b }  // => means "implies"
};
```
而不是定义两个无意义的概念has\_equal和has\_not\_equal，只是作为Equality定义中的辅助。我们所说的"无意义"是指我们不能孤立地指定has\_equal的语义。

<br/>
### **T.30:谨慎使用概念否定(!C\<T\>) 来表达微小差异**

<br/>
### **T.31:谨慎使用概念分离(C1\<T\> || C2\<T\>)来表达备选方案**

<br/>
## **模版接口**

多年来，使用模板进行编程一直受到模板接口及其实现之间微弱区分的困扰。在概念之前，这种区别没有直接的语言支持。然而，模板的接口是一个关键概念——用户和实施者之间的契约——应该仔细设计。

- [T.40:使用函数对象将操作传递给算法](#t40使用函数对象将操作传递给算法)
- [T.41:只需要模板概念中的基本属性](#t41只需要模板概念中的基本属性)
- [T.42:使用模板别名来简化符号并隐藏实现细节](#t42使用模板别名来简化符号并隐藏实现细节)
- [T.43:优先使用using而不是typedef来定义别名](#t43优先使用using而不是typedef来定义别名)
- [T.44:使用函数模板推导类模板参数类型（在可行的情况下）](#t44使用函数模板推导类模板参数类型在可行的情况下)
- [T.46:要求模板参数至少是semiregular的](#t46要求模板参数至少是semiregular的)
- [T.47:避免具有通用名称的高度可见的无约束模板](#t47避免具有通用名称的高度可见的无约束模板)
- [T.48:如果你的编译器不支持概念，用enable\_if伪装它们](#t48如果你的编译器不支持概念用enable_if伪装它们)
- [T.49:尽可能避免类型擦除](#t49尽可能避免类型擦除)

<br/>
### **T.40:使用函数对象将操作传递给算法**

**原因** 与一个 "普通"的函数指针相比，函数对象可以通过接口携带更多的信息。一般来说，传递函数对象比传递函数的指针有更好的性能。

**Example**
```cpp
bool greater(double x, double y) { return x > y; }
sort(v, greater);                                    // pointer to function: potentially slow
sort(v, [](double x, double y) { return x > y; });   // function object
sort(v, std::greater{});                             // function object

bool greater_than_7(double x) { return x > 7; }
auto x = find_if(v, greater_than_7);                 // pointer to function: inflexible
auto y = find_if(v, [](double x) { return x > 7; }); // function object: carries the needed data
auto z = find_if(v, Greater_than<double>(7));        // function object: carries the needed data
```
当然，你可以用auto或概念来概括这些功能。比如：
```cpp
auto y1 = find_if(v, [](totally_ordered auto x) { return x > 7; }); // require an ordered type
auto z1 = find_if(v, [](auto x) { return x > 7; });                 // hope that the type has a >
```

**注意** lambda生成函数对象。

**注意** 性能参数取决于编译器和优化器技术。

**Enforcement**
- 指向函数模板参数的指针要标记出。
- 指向作为参数传递给模板的函数的指针（有误报的风险）要标记出。

<br/>
### **T.41:只需要模板概念中的基本属性**

**原因** 保持接口简单和稳定。

**Example** 考虑一下，sort用（过度简化的）简单调试支持来检测。
```cpp
void sort(sortable auto& s)  // sort sequence s
{
    if (debug) cerr << "enter sort( " << s <<  ")\n";
    // ...
    if (debug) cerr << "exit sort( " << s <<  ")\n";
}
```
是否应该改写为：
```cpp
template<sortable S>
    requires Streamable<S>
void sort(S& s)  // sort sequence s
{
    if (debug) cerr << "enter sort( " << s <<  ")\n";
    // ...
    if (debug) cerr << "exit sort( " << s <<  ")\n";
}
```
毕竟，sortable中没有任何东西需要iostream支持。另一方面，排序的基本思想中没有任何关于调试的内容。

**注意** 如果我们要求每一个使用的操作都被列在需求中，那么接口就会变得不稳定。每当我们改变调试设施、使用数据收集、测试支持、错误报告等，模板的定义就需要改变，模板的每一次使用都必须重新编译。这很麻烦，而且在某些环境下是不可行的。
相反，如果我们在实现中使用了一个没有被概念检查所保证的操作，我们可能会得到一个迟到的编译时错误。
通过不对不被认为是必要的模板参数的属性使用概念检查，我们将检查推迟到实例化时间。我们认为这是一个值得的权衡。
请注意，使用非局部的、非依赖性的名称（如debug和cerr）也会引入上下文的依赖性，可能导致"神秘"的错误。

**注意** 要决定一个类型的哪些属性是必要的，哪些是不重要的，可能很困难。

<br/>
### **T.42:使用模板别名来简化符号并隐藏实现细节**

**原因** 提高了可读性。实现隐藏。请注意，模板别名取代了许多使用trait来计算类型的方法。它们也可以用来包装trait。

**Example**
```cpp
template<typename T, size_t N>
class Matrix {
    // ...
    using Iterator = typename std::vector<T>::iterator;
    // ...
};
```
这使Matrix的用户不必知道它的元素被存储在一个vector中，也使用户不必重复输入typename std::vector\<T\>::。

**Example**
```cpp
template<typename T>
void user(T& c)
{
    // ...
    typename container_traits<T>::value_type x; // bad, verbose
    // ...
}

template<typename T>
using Value_type = typename container_traits<T>::value_type;
```
这使Value\_type的用户不必知道用于实现value\_types的技术。
```cpp
template<typename T>
void user2(T& c)
{
    // ...
    Value_type<T> x;
    // ...
}
```

**Enforcement**
- 在using声明之外使用typename作为消除歧义符的要标记出。

<br/>
### **T.43:优先使用using而不是typedef来定义别名**

**原因** 改善可读性。有了using，新的名字就会出现在前面，而不是在声明中的某个地方嵌入。通用性：using可以用于模板别名，而类型定义则不容易成为模板。统一性：using在语法上与auto相似。

**Example**
```cpp
typedef int (*PFI)(int);   // OK, but convoluted

using PFI2 = int (*)(int);   // OK, preferred

template<typename T>
typedef int (*PFT)(T);      // error

template<typename T>
using PFT2 = int (*)(T);   // OK
```

<br/>
### **T.44:使用函数模板推导类模板参数类型（在可行的情况下）**

**原因** 明确写出模板参数类型可能会很繁琐，而且不必要地冗长。

**Example**
```cpp
tuple<int, string, double> t1 = {1, "Hamlet", 3.14};   // explicit type
auto t2 = make_tuple(1, "Ophelia"s, 3.14);         // better; deduced type
```
注意使用s后缀，以确保该字符串是std::string，而不是C风格的字符串。

**注意** 由于你可以毫不费力地写一个make\_T函数，编译器也可以。因此，make\_T函数在未来可能成为多余的。

**例外** 有时没有一个很好的方法来推导模板参数，有时，你想明确指定参数：
```cpp
vector<double> v = { 1, 2, 3, 7.9, 15.99 };
list<Record*> lst;
```

**注意** 请注意，C++17将允许直接从构造函数参数中推导出模板参数，从而使这条规则变得多余。
```cpp
tuple t1 = {1, "Hamlet"s, 3.14}; // deduced: tuple<int, string, double>
```

**Enforcement**
- 在显式专用类型与使用的参数类型完全匹配的地方使用要标记出。

<br/>
### **T.46:要求模板参数至少是semiregular的**

**原因** 可读性。防止意外和错误。无论如何，大多数用途都支持这一点。

**Example**
```cpp
class X {
public:
    explicit X(int);
    X(const X&);            // copy
    X operator=(const X&);
    X(X&&) noexcept;        // move
    X& operator=(X&&) noexcept;
    ~X();
    // ... no more constructors ...
};

X x {1};              // fine
X y = x;              // fine
std::vector<X> v(10); // error: no default constructor
```

**注意** Semiregular要求默认可构建。

**Enforcement**
- 用作模板参数的类型至少不是semiregular的要标记出。

<br/>
### **T.47:避免具有通用名称的高度可见的无约束模板**

**原因** 一个不受约束的模板参数对任何东西都是完美匹配的，所以这样的模板可以优先于需要轻微转换的更具体的类型。当使用ADL时，这尤其令人讨厌/危险。常见的名字使这个问题更容易发生。

**Example**
```cpp
namespace Bad {
    struct S { int m; };
    template<typename T1, typename T2>
    bool operator==(T1, T2) { cout << "Bad\n"; return true; }
}

namespace T0 {
    bool operator==(int, Bad::S) { cout << "T0\n"; return true; }  // compare to int

    void test()
    {
        Bad::S bad{ 1 };
        vector<int> v(10);
        bool b = 1 == bad;
        bool b2 = v.size() == bad;
    }
}
```
这将输出T0和Bad。
现在，Bad中的==被设计成了麻烦，但是在真正的代码中你会发现这个问题吗？问题在于v.size()返回的是一个无符号的整数，所以需要转换来调用本地的==；
Bad中的==不需要转换。现实的类型，比如标准库中的迭代器，也可以表现出类似的反社会倾向。

**注意** 如果一个不受约束的模板与一个类型定义在同一个命名空间，那么这个不受约束的模板可以被ADL找到（就像例子中发生的那样）。也就是说，它是高度可见的。

**注意** 这条规则应该是没有必要的，但委员会不能同意将不受约束的模板排除在ADL之外。
不幸的是，这将得到许多误报；标准库广泛地违反了这一点，它将许多不受约束的模板和类型放入单一的命名空间std。

**Enforcement**
- 在命名空间中定义的模板，其中还定义了具体类型要标记出（在我们有概念之前可能不可行）。

<br/>
### **T.48:如果你的编译器不支持概念，用enable_if伪装它们**

**原因** 因为这是我们在没有直接概念支持的情况下所能做到的最好的了。enable\_if可以用来有条件地定义函数，并在一组函数中进行选择。

**Example**
```cpp
template<typename T>
enable_if_t<is_integral_v<T>>
f(T v)
{
    // ...
}

// Equivalent to:
template<Integral T>
void f(T v)
{
    // ...
}
```

**注意** 小心互补性约束。使用enable\_if伪造概念重载，有时会迫使我们使用这种容易出错的设计技术。

<br/>
### **T.49:尽可能避免类型擦除**

**原因** 通过将类型信息隐藏在一个单独的编译边界后面，类型清除产生了一个额外的间接性。

**例外** 类型擦除有时是合适的，比如对于std::function。

<br/>
## **T.def:模版定义**

模板定义（类或函数）可以包含任意的代码，所以只有对C++编程技术的全面回顾才会涵盖这一主题。然而，本节的重点是模板实现的具体内容。特别是，它关注模板定义对其上下文的依赖性。

<br/>
### **T.60:最小化模板的上下文依赖**

**原因** 便于理解。最大限度地减少意外依赖的错误。简化工具创建。

**Example**
```cpp
template<typename C>
void sort(C& c)
{
    std::sort(begin(c), end(c)); // necessary and useful dependency
}

template<typename Iter>
Iter algo(Iter first, Iter last)
{
    for (; first != last; ++first) {
        auto x = sqrt(*first); // potentially surprising dependency: which sqrt()?
        helper(first, x);      // potentially surprising dependency:
                               // helper is chosen based on first and x
        TT var = 7;            // potentially surprising dependency: which TT?
    }
}
```

**注意** 模板通常出现在头文件中，所以它们的上下文依赖比.cpp文件中的函数更容易受到#include顺序依赖的影响。 

**注意** 让模板只对其参数进行操作是将依赖关系的数量减少到最小的一种方法，但这通常是无法管理的。
例如，算法通常使用其他算法并调用不专门对参数进行操作的操作。而且，不要让我们开始讨论宏的问题!

<br/>
### **T.61:不要过度参数化成员（SCARY）**

**原因** 除了特定的模板参数外，不能使用不依赖于模板参数的成员。这限制了使用并且通常会增加代码大小。

**Example, bad**
```cpp
template<typename T, typename A = std::allocator<T>>
    // requires Regular<T> && Allocator<A>
class List {
public:
    struct Link {   // does not depend on A
        T elem;
        Link* pre;
        Link* suc;
    };

    using iterator = Link*;

    iterator first() const { return head; }

    // ...
private:
    Link* head;
};

List<int> lst1;
List<int, My_allocator> lst2;
```
这看起来很简单，但现在Link正式依赖于分配器（尽管它并没有使用分配器）。这就迫使我们进行多余的实例化，在一些现实世界的场景中，其代价是惊人的。
通常情况下，解决方案是使原本的嵌套类成为非局部类，并拥有自己的最小模板参数集。
```cpp
template<typename T>
struct Link {
    T elem;
    Link* pre;
    Link* suc;
};

template<typename T, typename A = std::allocator<T>>
    // requires Regular<T> && Allocator<A>
class List2 {
public:
    using iterator = Link<T>*;

    iterator first() const { return head; }

    // ...
private:
    Link<T>* head;
};

List2<int> lst1;
List2<int, My_allocator> lst2;
```
有些人认为链接不再隐藏在列表中的想法很可怕，所以我们把这项技术命名为"SCARY"。来自那篇学术论文。
"SCARY这个缩写描述了看似错误的赋值和初始化（看起来受制于冲突的通用参数），但实际上在正确的实现下是可行的（由于最小化的依赖关系而不受冲突的约束）。

**注意** 这也适用于那些不依赖所有模板参数的lambdas。

**Enforcement**
- 不依赖每个模板参数的成员类型标记出
- 不依赖每个模板参数的成员函数标记出
- 不依赖每个模板参数的lambdas或变量模板要标记出

<br/>
### **T.62:将非依赖类模板成员放在非模板化基类中**

**原因** 允许基类成员在不指定模板参数和不进行模板实例化的情况下被使用。

**Example**
```cpp
template<typename T>
class Foo {
public:
    enum { v1, v2 };
    // ...
};
```
```cpp
struct Foo_base {
    enum { v1, v2 };
    // ...
};

template<typename T>
class Foo : public Foo_base {
public:
    // ...
};
```

**注意** 这个规则的一个更普遍的版本是："如果一个类的模板成员只依赖于M个模板参数中的N个，那么把它放在一个只有N个参数的基类中。"
对于N == 1，我们可以选择周围范围内的一个类的基类，如T.61。

<br/>
### **T.64:使用特例化来提供类模板的替代实现**

**原因** 模板定义了一个通用接口。特例化提供了一个强大的机制来提供该接口的替代实现。

<br/>
### **T.65:使用标签分派来提供功能的替代实现**

**原因**
- 一个模板定义了一个通用的接口
- 标签分派允许我们根据参数类型的特定属性来选择实现方式
- 性能

**Example** 这是std::copy的简化版本（忽略了非连续序列的可能性）。
```cpp
struct pod_tag {};
struct non_pod_tag {};

template<class T> struct copy_trait { using tag = non_pod_tag; };   // T is not "plain old data"

template<> struct copy_trait<int> { using tag = pod_tag; };         // int is "plain old data"

template<class Iter>
Out copy_helper(Iter first, Iter last, Iter out, pod_tag)
{
    // use memmove
}

template<class Iter>
Out copy_helper(Iter first, Iter last, Iter out, non_pod_tag)
{
    // use loop calling copy constructors
}

template<class Iter>
Out copy(Iter first, Iter last, Iter out)
{
    return copy_helper(first, last, out, typename copy_trait<Value_type<Iter>>::tag{})
}

void use(vector<int>& vi, vector<int>& vi2, vector<string>& vs, vector<string>& vs2)
{
    copy(vi.begin(), vi.end(), vi2.begin()); // uses memmove
    copy(vs.begin(), vs.end(), vs2.begin()); // uses a loop calling copy constructors
}
```
这是一种用于编译时算法选择的通用而强大的技术。

**注意** 当concept变得广泛可用时，这样的替代品就可以直接区分出来。
```cpp
template<class Iter>
    requires Pod<Value_type<Iter>>
Out copy_helper(In first, In last, Out out)
{
    // use memmove
}

template<class Iter>
Out copy_helper(In first, In last, Out out)
{
    // use loop calling copy constructors
}
```

<br/>
### **T.67:使用特例化为不规则类型提供替代实现**

<br/>
### **T.68:在模板中使用{}而不是()以避免歧义**

**原因** ()容易受到语法歧义的影响。

**Example**
```cpp
template<typename T, typename U>
void f(T t, U u)
{
    T v1(T(u));    // mistake: oops, v1 is a function not a variable
    T v2{u};       // clear:   obviously a variable
    auto x = T(u); // unclear: construction or cast?
}

f(1, "asdf"); // bad: cast from const char* to int
```

**Enforcement**
- ()初始化标记出
- 函数类型转换标记出

<br/>
### **T.69:在模板内，不要进行不合格的非成员函数调用，除非您打算将其作为自定义点**

**原因**
- 只提供预期的灵活性。
- 避免易受意外环境变化的影响。

**Example** 有三种主要方式可以让调用代码定制模板。
```cpp
template<class T>
    // Call a member function
void test1(T t)
{
    t.f();    // require T to provide f()
}

template<class T>
void test2(T t)
    // Call a non-member function without qualification
{
    f(t);     // require f(/*T*/) be available in caller's scope or in T's namespace
}

template<class T>
void test3(T t)
    // Invoke a "trait"
{
    test_traits<T>::f(t); // require customizing test_traits<>
                          // to get non-default functions/types
}
```
特征通常是用于计算类型的类型别名、用于计算值的constexpr函数，或者是专门针对用户类型的传统特征模板。

**注意** 如果你打算用一个依赖于模板类型参数的值t来调用你自己的辅助函数helper(t)，把它放在::detail命名空间中，并把调用限定为detail::helper(t)。
一个未限定的调用成为一个自定义点，在这里可以调用t的类型的命名空间中的任何函数helper。这可能会引起一些问题，比如无意中调用了不受约束的函数模板。

**Enforcement**
- 在模板中，当模板的命名空间中存在同名的非成员函数时，对传递依赖类型变量的非成员函数的非限定调用要标记出。

<br/>
# **T.temp-hier:模版和层次结构规则**

模板是C++支持泛型编程的支柱，而类的层次结构是支持面向对象编程的支柱。这两种语言机制可以有效地结合使用，但必须避免一些设计上的误区。

<br/>
### **T.80:不要天真地模板化类层次结构**

**原因** 模板化的类层次结构有很多函数，尤其是很多虚函数，会导致代码臃肿。

**Example, bad**
```cpp
template<typename T>
struct Container {         // an interface
    virtual T* get(int i);
    virtual T* first();
    virtual T* next();
    virtual void sort();
};

template<typename T>
class Vector : public Container<T> {
public:
    // ...
};

Vector<int> vi;
Vector<string> vs;
```
将一个sort定义为一个容器的成员函数可能是个坏主意，但这并非闻所未闻，它是一个很好的例子，说明了什么是不应该做的。

鉴于此，编译器无法知道vector\<int\>::sort()是否被调用，所以它必须为其生成代码。对于vector\<string\>::sort()也是如此。
除非这两个函数被调用，否则就是代码膨胀了。想象一下这对一个有几十个成员函数和几十个有许多实例的派生类的层次结构会有什么影响。

**注意** 在许多情况下，你可以通过不对基类进行参数化来提供一个稳定的接口；见 "稳定的基类"和OO与GP。

**Enforcement**
- 依赖于模板参数的虚函数要标记出。

<br/>
### **T.81:不要混合层次结构和数组**

**原因** 派生类的数组可以隐含地退化到基类的指针上，可能会造成灾难性的结果。

**Example**
```cpp
void maul(Fruit* p)
{
    *p = Pear{};     // put a Pear into *p
    p[1] = Pear{};   // put a Pear into p[1]
}

Apple aa [] = { an_apple, another_apple };   // aa contains Apples (obviously!)

maul(aa);
Apple& a0 = &aa[0];   // a Pear?
Apple& a1 = &aa[1];   // a Pear?
```
可能，aa[0]将是一个梨（没有使用cast！）。如果sizeof(Apple) != sizeof(Pear)，对aa[1]的访问将不会对准数组中一个对象的正确起点。
我们有一个类型违规，并且可能（很可能）有一个内存损坏。千万不要写这样的代码。
请注意，maul()违反了一个T\*指向一个单独对象的规则。

**替代** 使用适当的（模板化）容器。
```cpp
void maul2(Fruit* p)
{
    *p = Pear{};   // put a Pear into *p
}

vector<Apple> va = { an_apple, another_apple };   // va contains Apples (obviously!)

maul2(va);       // error: cannot convert a vector<Apple> to a Fruit*
maul2(&va[0]);   // you asked for it

Apple& a0 = &va[0];   // a Pear?
```
请注意，maul2()中的赋值违反了不slice规则。[ES.63: Don’t slice](#es63不要切片)

<br/>
### **T.82:当虚拟函数不可取时，要线性化层次结构**

<br/>
### **T.83:不要将成员函数模板声明为virtual**

**原因** C++并不支持这一点。如果它支持，vtbls在链接时才会被生成。而且一般来说，实现必须处理动态链接。

**Example, don’t**
```cpp
class Shape {
    // ...
    template<class T>
    virtual bool intersect(T* p);   // error: template cannot be virtual
};
```

**注意** 我们需要一个规则，因为人们一直在问这个问题。

**替代方案** Double dispatch, visitors, 计算要调用哪个函数。

**Enforcement** 编译器会处理这个问题。

<br/>
### **T.84:使用非模板核心实现来提供ABI稳定的接口**

**原因** 提高代码的稳定性。避免代码臃肿。

**Example**
```cpp
struct Link_base {   // stable
    Link_base* suc;
    Link_base* pre;
};

template<typename T>   // templated wrapper to add type safety
struct Link : Link_base {
    T val;
};

struct List_base {
    Link_base* first;   // first element (if any)
    int sz;             // number of elements
    void add_front(Link_base* p);
    // ...
};

template<typename T>
class List : List_base {
public:
    void put_front(const T& e) { add_front(new Link<T>{e}); }   // implicit cast to Link_base
    T& front() { static_cast<Link<T>*>(first).val; }   // explicit cast back to Link<T>
    // ...
};

List<int> li;
List<string> ls;
```
现在只有一份链接和解除链接List元素的操作。Link和List类只做类型操作。
另一种常用技术不是使用单独的base类型，另一种常见的技术是对void或void\*进行特化处理，并使T的通用模板只是安全封装的与void实现的转换。

**替代** Pimpl

<br/>
## **T.var:可变模版规则**

<br/>
### **T.100:当你需要一个函数接受各种类型的可变数量的参数时，使用可变参数模板**

**原因** 可变模板是最通用的机制，它既高效又类型安全。不要使用C语言的varargs。

**Enforcement**
- 用户代码使用va\_arg要标记出。

<br/>
### **T.101:如何将参数传递给可变参数模板**

<br/>
### **T.102:如何处理可变参数模板的参数**

<br/>
### **T.103:不要对同类参数列表使用可变参数模板**

<br/>
## **T.meta:模板元编程(TMP)**

模板为编译时编程提供了一种通用机制。
元编程是指至少有一个输入或一个结果是一个类型的编程。模板在编译时提供了图灵完全（内存容量模数）的鸭子类型。所需的语法和技术是相当可怕的。

<br/>
### **T.120:只有在真正需要时才使用模板元编程**

**原因** 模板元编程很难做到正确，会减慢编译速度，而且通常很难维护。然而，有一些真实世界的例子，其中模板元编程提供了比专家级汇编代码之外的任何替代方案更好的性能。
此外，还有一些现实世界的例子，其中模板元编程比运行时代码更好地表达了基本思想。例如，如果你真的需要在编译时进行 AST 操作（例如，用于可选的矩阵操作折叠），那么在 C++ 中可能没有其他方法。 

**替代** 如果结果是一个值，而不是一个类型，请使用一个constexpr函数。

<br/>
### **T.121:主要使用模板元编程来模拟概念**

**原因** 在C++20不可用的地方，我们需要用TMP来模拟它们。需要概念的用例（如基于概念的重载）是TMP最常见（和简单）的用途之一。

**Example**
```cpp
template<typename Iter>
    /*requires*/ enable_if<random_access_iterator<Iter>, void>
advance(Iter p, int n) { p += n; }

template<typename Iter>
    /*requires*/ enable_if<forward_iterator<Iter>, void>
advance(Iter p, int n) { assert(n >= 0); while (n--) ++p;}
```

**注意** 使用概念，这样的代码就简单多了。
```cpp
void advance(random_access_iterator auto p, int n) { p += n; }

void advance(forward_iterator auto p, int n) { assert(n >= 0); while (n--) ++p;}
```

<br/>
### **T.122:使用模板（通常是模板别名）在编译时计算类型**

**原因** 模板元编程是在编译时生成类型的唯一直接支持和半途原则的方法。

**注意** "Traits"技术大多被计算类型的模板别名和计算数值的constexpr函数所取代。

<br/>
### **T.123:使用constexpr函数在编译时计算值**

**原因** 一个函数是表达一个值的计算的最明显和最传统的方式。通常情况下，constexpr函数意味着比其他方法更少的编译时开销。

**注意** "Traits"技术大多被计算类型的模板别名和计算数值的constexpr函数所取代。

**Example**
```cpp
template<typename T>
    // requires Number<T>
constexpr T pow(T v, int n)   // power/exponential
{
    T res = 1;
    while (n--) res *= v;
    return res;
}

constexpr auto f7 = pow(pi, 7);
```

**Enforcement**
- 产生一个值的模板元程序要标记出。这些应该被替换成constexpr函数。

<br/>
### **T.124:优先使用标准库TMP设施**

**原因** 标准中定义的设施，conditional,enable\_if,tuple，是可移植的，可以假定是已知的。

<br/>
### **T.125:如果您需要超越标准库TMP设施，请使用现有库**

**原因** 获得先进的TMP设施并不容易，使用一个库使你成为一个（希望是支持的）社区的一部分。只有在你真的需要的情况下，才写你自己的"高级TMP支持"。

<br/>
## **其他模版规则**

<br/>
### **T.140:如果一个操作可以重用，给它一个名字**

见F.10

<br/>
### **T.141:如果只在一个地方需要一个简单的函数对象，请使用未命名的lambda**

见F.11

<br/>
### **T.142:使用模板变量来简化符号TMP设施**

**原因** 提高了可读性。

<br/>
### **T.143:不要无意中编写非泛型代码**

**原因** 通用性。可重复使用。不要无缘无故地致力于细节；使用最通用的设施。

**Example** 使用!=而不是<来比较迭代器；!=对更多的对象有效，因为它不依赖于排序。
```cpp
for (auto i = first; i < last; ++i) {   // less generic
    // ...
}

for (auto i = first; i != last; ++i) {   // good; more generic
    // ...
}
```
当然一般来说range-for更好。

**Example** 使用具有你所需功能的最小派生类。
```cpp
class Base {
public:
    Bar f();
    Bar g();
};

class Derived1 : public Base {
public:
    Bar h();
};

class Derived2 : public Base {
public:
    Bar j();
};

// bad, unless there is a specific reason for limiting to Derived1 objects only
void my_func(Derived1& param)
{
    use(param.f());
    use(param.g());
}

// good, uses only Base interface so only commit to that
void my_func(Base& param)
{
    use(param.f());
    use(param.g());
}
```

**Enforcement**
- 迭代器的比较使用<而不是!=标记出。
- 当x.empty()或x.is\_empty()可用时，x.size() == 0使用要标记出。empty比size()对更多的容器起作用，因为有些容器不知道自己的大小，或者在概念上是无界的大小。
- 函数接受一个指向更多派生类型的指针或引用，但只使用基类型中声明的函数要标记出。

<br/>
### **T.144:不要特化函数模板**

**原因** 根据语言规则，你不能部分地特化一个函数模板。你可以完全特化一个函数模板，但你几乎肯定想用重载来代替--因为函数模板的特化不参与重载，它们不会像你所希望的那样行动。
很少的情况下，你应该通过委托给一个你可以正确特化的类模板来进行特化。

**例外** 如果你确实有合理的理由来特化一个函数模板，只需写一个委托给类模板的函数模板，然后特化这个类模板（包括写部分特化的能力）。

**Enforcement**
- 标记一个函数模板的所有特化。用重载代替。

<br/>
### **T.150:使用static_assert检查类是否匹配概念**

**原因** 如果你打算让一个类与一个概念相匹配，那么尽早验证就能减轻用户的痛苦。

**Example**
```cpp
class X {
public:
    X() = delete;
    X(const X&) = default;
    X(X&&) = default;
    X& operator=(const X&) = default;
    // ...
};
```
在某个地方，可能是在一个实现文件中，让编译器检查X的预期属性。
```cpp
static_assert(Default_constructible<X>);    // error: X has no default constructor
static_assert(Copyable<X>);                 // error: we forgot to define X's move constructor
```

<br/>
# **CPL:C风格编程**

C和C++是密切相关的语言。它们都起源于1978年的“Classic C”，并从那时起在ISO委员会中发展。已经进行了许多尝试来保持它们的兼容性，但两者都不是另一个的子集。

C规则摘要：
- [CPL.1:优先选择C++而不是C](#cpl1优先选择c而不是c)
- [CPL.2:如果一定要用C，就用C和C++的公共子集，把C代码编译成C++](#cpl2如果一定要用c就用c和c的公共子集把c代码编译成c)
- [CPL.3:如果必须使用C作为接口，请在使用此类接口的调用代码中使用C++](#cpl3如果必须使用c作为接口请在使用此类接口的调用代码中使用c)

<br/>
### **CPL.1:优先选择C++而不是C**

**原因** C++提供了更好的类型检查和更多的符号支持。它为高级编程提供了更好的支持，并且通常可以生成更快的代码。

**Example**
```cpp
char ch = 7;
void* pv = &ch;
int* pi = pv;   // not C++
*pi = 999;      // overwrite sizeof(int) bytes near &ch
```
在C中隐式转换为void\*是微妙且未强制执行的。特别是，此示例违反了禁止转换为具有更严格对齐的类型的规则。

**Enforcement** 使用C++编译器。

<br/>
### **CPL.2如果一定要用C，就用C和C++的公共子集，把C代码编译成C++**

**原因** 该子集可以用C和C++编译器编译，并且当编译为C++时，类型检查比“纯C”更好。

**Example**
```cpp
int* p1 = malloc(10 * sizeof(int));                      // not C++
int* p2 = static_cast<int*>(malloc(10 * sizeof(int)));   // not C, C-style C++
int* p3 = new int[10];                                   // not C
int* p4 = (int*) malloc(10 * sizeof(int));               // both C and C++
```

**Enforcement** 如果使用将代码编译为C的构建模式，则标记出。
C++编译器将强制代码是有效的C++，除非您使用C扩展选项。 

<br/>
### **CPL.3:如果必须使用C作为接口，请在使用此类接口的调用代码中使用C++**

**原因** C++比C更具表现力，并为许多类型的编程提供更好的支持。

**Example** 例如，要使用第三方C库或C系统接口，请在C和C++的公共子集中定义low-level接口，以便更好地进行类型检查。
尽可能将低级接口封装在遵循C++准则的接口中（以实现更好的抽象、内存安全和资源安全），并在C++代码中使用该C++接口。

**Example** 您可以从C++调用C：
```cpp
// in C:
double sqrt(double);

// in C++:
extern "C" double sqrt(double);

sqrt(2);
```

**Example** 您可以从C调用C++：
```cpp
// in C:
X call_f(struct Y*, int);

// in C++:
extern "C" X call_f(Y* p, int i)
{
    return p->f(i);   // possibly a virtual function call
}
```

<br/>
# **SF:源文件**

区分声明（作为接口使用）和定义（作为实现使用）。使用头文件来表示接口并强调逻辑结构。

源文件规则摘要：
- [SF.1:如果你的项目还没有遵循其他惯例的话，代码文件使用.cpp后缀，接口文件使用.h后缀](#sf1如果你的项目还没有遵循其他惯例的话代码文件使用cpp后缀接口文件使用h后缀)
- [SF.2:头文件不得包含对象定义或非内联函数定义](#sf2头文件不得包含对象定义或非内联函数定义)
- [SF.3:对多个源文件中使用的所有声明使用头文件](#sf3对多个源文件中使用的所有声明使用头文件)
- [SF.4:在文件中的其他声明之前包含头文件](#sf4在文件中的其他声明之前包含头文件)
- [SF.5:一个.cpp文件必须包括定义其接口的头文件](#sf5一个cpp文件必须包括定义其接口的头文件)
- [SF.6:使用命名空间指令进行转换，用于基础库（如std），或在本地范围内使用（仅）](#sf6使用命名空间指令进行转换用于基础库如std或在本地范围内使用仅)
- [SF.7:不要在头文件中全局范围内使用using namespace](#sf7不要在头文件中全局范围内使用using-namespace)
- [SF.8:对所有头文件使用#include保护](#sf8对所有头文件使用include保护)
- [SF.9:避免源文件之间的循环依赖](#sf9避免源文件之间的循环依赖)
- [SF.10:避免依赖于隐式#included名称](#sf10避免依赖于隐式included名称)
- [SF.11:头文件应该是自成一体的](#sf11头文件应该是自成一体的)
- [SF.12:相对于包含文件和其他地方的尖括号形式，对于文件更喜欢#include的引号形式](#sf12相对于包含文件和其他地方的尖括号形式对于文件更喜欢include的引号形式)
- [SF.20:使用命名空间来表达逻辑结构](#sf20使用命名空间来表达逻辑结构)
- [SF.21:不要在头中使用未命名（匿名）的命名空间](#sf21不要在头中使用未命名匿名的命名空间)
- [SF.22:对所有内部/非出口的实体使用一个未命名（匿名）的命名空间](#sf22对所有内部非出口的实体使用一个未命名匿名的命名空间)

<br/>
### **SF.1:如果你的项目还没有遵循其他惯例的话，代码文件使用.cpp后缀，接口文件使用.h后缀**

见[NL.27](#nl27)

<br/>
### **SF.2:头文件不得包含对象定义或非内联函数定义**

**原因** 包含受单一定义规则约束的实体会导致链接错误。

**Example**
```cpp
// file.h:
namespace Foo {
    int x = 7;
    int xx() { return x+x; }
}

// file1.cpp:
#include <file.h>
// ... more ...

 // file2.cpp:
#include <file.h>
// ... more ...
```
链接file1.cpp和file2.cpp会出现两个链接器错误。

**替代方案** 一个头文件必须只包含：
- \#包含其他头文件
- templates
- 类定义
- 函数声明
- extern声明
- inline函数定义
- constexpr定义
- const定义
- using别名定义 

<br/>
### **SF.3:对多个源文件中使用的所有声明使用头文件**

**原因** 可维护性。可读性。

**Example, bad**
```cpp
// bar.cpp:
void bar() { cout << "bar\n"; }

// foo.cpp:
extern void bar();
void foo() { bar(); }
```
如果bar的类型需要改变，bar的维护者无法找到bar的所有声明。bar的用户无法知道所使用的接口是否完整和正确。充其量，错误信息会（延迟）来自链接器。

**Enforcement**
- 其他源文件中的实体声明没有放在.h中要标记出。

<br/>
### **SF.4:在文件中的其他声明之前包含头文件**

**原因** 尽量减少上下文的依赖性，提高可读性。

**Example**
```cpp
#include <vector>
#include <algorithm>
#include <string>

// ... my code here ...
```

**Example, bad**
```cpp
#include <vector>

// ... my code here ...

#include <algorithm>
#include <string>
```

**注意** 这同时适用于.h和.cpp文件。

**注意** 有一种观点认为，通过在我们想要保护的代码之后\#包含头文件，可以将代码与头文件中的声明和宏隔离开来（如标有"bad"的例子）。然而
- 这只适用于一个文件（在一个级别）。在一个与其他头文件一起包含的头文件中使用这种技术，漏洞就会重新出现。
- 一个命名空间（一个 "实现命名空间"）可以保护许多上下文的依赖关系。
- 完整的保护和灵活性需要module。

<br/>
### **SF.5:一个.cpp文件必须包括定义其接口的头文件**

**原因** 这使得编译器能够进行早期的一致性检查。

**Example, bad**
```cpp
// foo.h:
void foo(int);
int bar(long);
int foobar(int);

// foo.cpp:
void foo(int) { /* ... */ }
int bar(double) { /* ... */ }
double foobar(int);
```
在调用bar或foobar的程序的链接时间之前，这些错误不会被发现。

**Example**
```cpp
// foo.h:
void foo(int);
int bar(long);
int foobar(int);

// foo.cpp:
#include "foo.h"

void foo(int) { /* ... */ }
int bar(double) { /* ... */ }
double foobar(int);   // error: wrong return type
```
现在，当编译foo.cpp时，foobar的返回类型错误被立即捕获。由于重载的可能性，bar的参数类型错误在链接时才会被发现，但系统地使用.h文件增加了它被程序员提前发现的可能性。

<br/>
### **SF.6:使用命名空间指令进行转换，用于基础库（如std），或在本地范围内使用（仅）**

**原因** 使用命名空间可能会导致名称冲突，所以应该尽量少用。然而，在用户代码中，并不总是能够对命名空间的每一个名字进行限定（例如，在过渡期间），
有时一个命名空间在代码库中是非常基本和普遍的，一致的限定将是冗长和分散注意力的。

**Example**
```cpp
#include <string>
#include <vector>
#include <iostream>
#include <memory>
#include <algorithm>

using namespace std;

// ...
```
在这里（很明显），标准库被普遍使用，显然没有使用其他库，所以到处要求std::会让人分心。

**Example** 使用命名空间std，使程序员有可能与标准库中的名称发生冲突。
```cpp
#include <cmath>
using namespace std;

int g(int x)
{
    int sqrt = 7;
    // ...
    return sqrt(x); // error
}
```
然而，这并不特别容易导致一个决议，这个决议不是错误的，使用命名空间std的人应该知道std和这种风险。

**注意** .cpp文件是局部范围的一种形式。在一个包含using namespace X的N行.cpp中，一个包含using namespace X的N行函数，
以及M个包含using namespace X的函数，名称冲突的机会几乎没有差异，总共有N行代码。

**注意** [不要在头文件中的全局范围内使用命名空间进行编写](#sf7要在头文件中全局范围内使用usingnamespace)

**Enforcement** 单个源文件中不同命名空间的多个using命名空间指令要标记出。

<br/>
### **SF.7:不要在头文件中全局范围内使用using namespace**

**原因** 这样做剥夺了#include有效消除歧义和使用替代方案的能力。这也使得#include头文件的顺序受到影响，因为它们在不同的顺序中被包含时可能有不同的意义。

**Example**
```cpp
// bad.h
#include <iostream>
using namespace std; // bad

// user.cpp
#include "bad.h"

bool copy(/*... some parameters ...*/);    // some function that happens to be named copy

int main()
{
    copy(/*...*/);    // now overloads local ::copy and std::copy, could be ambiguous
}
```

**注意** 一个例外是使用命名空间std::literals。这对于在头文件中使用字符串常量是必要的，而且考虑到规则--用户需要命名他们自己的UDL运算符""\_x--他们将不会与标准库发生冲突。

**Enforcement** 在头文件中使用全局范围的命名空间要标记出。

<br/>
### **SF.8:对所有头文件使用#include保护**

**原因** 为了避免文件被多次#include。
为了避免包含guard的碰撞，不要只在文件名后命名guard。要确保还包括一个关键的、好的区分符，比如头文件的库名或组件名，这是头文件的一部分。

**Example**
```cpp
// file foobar.h:
#ifndef LIBRARY_FOOBAR_H
#define LIBRARY_FOOBAR_H
// ... declarations ...
#endif // LIBRARY_FOOBAR_H
```

**Enforcement** 没有#include防护的.h文件要标记出。

**注意** 一些实现提供了供应商的扩展，如#pragma once，作为include guard的替代。这不是标准的，也是不可移植的。
它将主机的文件系统语义注入到你的程序中，此外还将你锁定在一个供应商那里。我们的建议是用ISO C++编写：见规则P.2。

<br/>
### **SF.9:避免源文件之间的循环依赖**

**原因** 循环使理解变得复杂，并减慢编译速度。它们还使转换为使用语言支持的module（当它们可用时）变得复杂。

**注意** 消除循环；不要只是用#include guard来打破它们。

**Example, bad**
```cpp
// file1.h:
#include "file2.h"

// file2.h:
#include "file3.h"

// file3.h:
#include "file1.h"
```
**Enforcement** 所有的循环依赖都要标记出。

<br/>
### **SF.10:避免依赖于隐式#included名称**

**原因** 避免意外。避免在#included头文件更改时更改#includes。避免意外地依赖于实现细节和逻辑上包含在头文件中的独立实体。

**Example, bad**
```cpp
#include <iostream>
using namespace std;

void use()
{
    string s;
    cin >> s;               // fine
    getline(cin, s);        // error: getline() not defined
    if (s == "surprise") {  // error == not defined
        // ...
    }
}
```
\<iostream\>暴露了std::string的定义（"为什么？"是一个有趣的小问题），但它不需要通过转换为包括整个\<string\>头来做到这一点，
导致流行的初学者问题 "为什么getline(cin,s);不工作？"甚至偶尔出现 "字符串不能用==来比较"。
解决办法是明确地#include \<string\>。

**Example, good**
```cpp
#include <iostream>
#include <string>
using namespace std;

void use()
{
    string s;
    cin >> s;               // fine
    getline(cin, s);        // fine
    if (s == "surprise") {  // fine
        // ...
    }
}
```

**注意** 有些头文件的存在正是为了从各种头文件中收集一套一致的声明。比如:
```cpp
// basic_std_lib.h:

#include <string>
#include <map>
#include <iostream>
#include <random>
#include <vector>
```
用户现在可以通过一个#include来获得这组声明
```cpp
#include "basic_std_lib.h"
```
这条反对隐性包含的规则并不是为了防止这种故意的聚合。

**Enforcement** 将需要一些知识，了解头文件中的哪些内容是要 "输出 "给用户的，哪些是为了实现的。在我们拥有module之前，没有真正好的解决方案是可能的。

<br/>
### **SF.11:头文件应该是自成一体的**

**原因** 可用性，头文件应该简单易用，并且在单独包含时可以使用。头文件应该封装它们提供的功能。避免头文件的客户不得不管理该头文件的依赖关系。

**Example**
```cpp
#include "helpers.h"
// helpers.h depends on std::string and includes <string>
```

**注意** 如果不遵循这一点，就会导致头文件的客户难以诊断出错误。

**注意** 一个头文件应该包括它的所有依赖关系。要小心使用相对路径，因为C++实现在它们的含义上存在差异。

**Enforcement** 一个测试应该验证头文件本身是否可以编译，或者验证一个只包括头文件的cpp文件是否可以编译。

<br/>
### **SF.12:相对于包含文件和其他地方的尖括号形式，对于文件更喜欢#include的引号形式**

**原因** 标准为编译器提供了灵活性，可以实现使用角度(<>) 或引号("")语法选择的两种形式的#include。供应商利用这一点并使用不同的搜索算法和方法来指定包含路径。
然而，指导意见是使用带引号的形式来包含那些存在于包含#include语句的文件的相对路径上的文件（在同一个组件或项目中），在其他地方尽可能使用角括号形式。
这鼓励我们清楚地了解文件相对于包含它的文件的位置，或需要不同搜索算法的情况。这使得人们很容易理解头文件是来自本地相对文件，还是来自标准库的头文件，
或是来自替代搜索路径的头文件（例如，来自另一个库的头文件或一套通用的包含文件）。

**Example**
```cpp
// foo.cpp:
#include <string>                // From the standard library, requires the <> form
#include <some_library/common.h> // A file that is not locally relative, included from another library; use the <> form
#include "foo.h"                 // A file locally relative to foo.cpp in the same project, use the "" form
#include "foo_utils/utils.h"     // A file locally relative to foo.cpp in the same project, use the "" form
#include <component_b/bar.h>     // A file in the same project located via a search path, use the <> form
```

**注意** 如果不遵循这一点，就会导致难以诊断的错误，原因是在包含文件时不正确地指定了范围，从而捡到了错误的文件。
例如，在一个典型的案例中，#include "" 搜索算法可能会首先搜索一个存在于本地相对路径的文件，那么使用这种形式来引用一个非本地相对的文件可能意味着，
如果一个文件在本地相对路径下出现（例如，包括的文件被移动到一个新的位置），它现在会在之前的包括文件之前被发现，并且包括的集合会以一种意外的方式改变。
库的创建者应该把他们的头文件放在一个文件夹里，让客户使用相对路径来包含这些文件 \#include \<some\_library/common.h\> 。

**Enforcement** 一个测试应该确定通过""引用的头文件是否可以用\<\>来引用。

<br/>
### **SF.20:使用命名空间来表达逻辑结构**

<br/>
### **SF.21:不要在头中使用未命名（匿名）的命名空间**

**原因** 在头文件中提到一个未命名的命名空间，几乎都是一个错误。

**Example**
```cpp
// file foo.h:
namespace
{
    const double x = 1.234;  // bad

    double foo(double y)     // bad
    {
        return y + x;
    }
}

namespace Foo
{
    const double x = 1.234; // good

    inline double foo(double y)        // good
    {
        return y + x;
    }
}
```

**Enforcement** 在头文件中使用任何匿名命名空间都应标记出。

<br/>
### **SF.22:对所有内部/非出口的实体使用一个未命名（匿名）的命名空间**

**原因** 任何外部事物都不能依赖于嵌套的未命名命名空间中的实体。考虑将实现源文件中的每一个定义放在一个未命名的命名空间中，除非是定义一个"外部/导出的"实体。

**Example, bad**
```cpp
static int f();
int g();
static bool h();
int k();
```

**Example, good**
```cpp
namespace {
    int f();
    bool h();
}
int g();
int k();
```

**Example** 一个API类及其成员不能放在一个未命名的命名空间中；但任何在实现源文件中定义的"辅助"类或函数应该处于一个未命名的命名空间范围。

<br/>
# **SL:标准库**

C++标准库组件摘要：
- [SL.con:容器](#slcon容器)
- [SL.str:String](#slstrstring)
- [SL.io:Iostream](#slioiostream)
- [SL.regex:Regex](#slregexregex)
- [SL.chrono:Time](#slchronotime)
- [SL.C:C标准库](#slcc标准库)

标准库规则摘要：
- [SL.1:尽可能使用库](#sl1尽可能使用库)
- [SL.2:优先使用标准库](#sl2优先使用标准库)
- [SL.3:不要在std命名空间内增加非标准实体](#sl3不要在std命名空间内增加非标准实体)
- [SL.4:以类型安全方式使用标准库](#sl4以类型安全方式使用标准库)

<br/>
## **SL.con:容器**

容器规则摘要：
- [SL.con1:优先使用STL的数组或vector取代C风格数组](#slcon1优先使用stl的数组或vector取代c风格数组)
- [SL.con2:除非有理由，优先使用STL的vector](#slcon2除非有理由优先使用stl的vector)
- [SL.con3:避免越界](#slcon3避免越界)
- [SL.con4:不是简单可拷贝的参数不要用memset或memcpy](#slcon4不是简单可拷贝的参数不要用memset或memcpy)

<br/>
### **SL.con1:优先使用STL的数组或vector取代C风格数组**

**原因** C风格数组不安全，与std::array和std::vector比没有优势。对于固定长度的数组，使用std::array，它在传递给函数时不会退化为指针，且它知道大小。
另外，像内置数组一样，堆栈分配的std::array将其元素保留在堆栈中。可以可变长度的数组，使用std::vector，它可以改变其大小并处理内存分配。

**Example**
```cpp
int v[SIZE];                        // BAD

std::array<int, SIZE> w;            // ok
```

**Example**
```cpp
int* v = new int[initial_size];     // BAD, owning raw pointer
delete[] v;                         // BAD, manual delete

std::vector<int> w(initial_size);   // ok
```

**注意** 使用gsl::span来引用非所有权的容器。

**注意** 比较一个栈上分配的固定长度的数组和一个在堆中分配元素的vector的性能是错误的。你也可以把栈上分配的std::array和通过指针访问的malloc结果比较。
对于大多数代码来说，栈分配和堆分配的区别并不重要，但是vector的便利性和安全性却很重要。如果认为这两种差异比较影响的话，那他也一定有能力在数组和vector之间做出选择。

**Enforcement**
- 在一个声明STL容器的类或函数中，同时声明一个C风格数组，应该标记出。解决：把C风格数组改为std::array。

<br/>
### **SL.con2:除非有理由，优先使用STL的vector**

**原因** vector和array是标准库中唯一具有以下优点的容器：
- 最快的通用访问（随机访问，包括对矢量化友好）。
- 最快的默认访问模式（从头到尾或从尾到头的访问模式对预存器有利）。
- 最低的空间开销（连续布局的每元素开销为零，缓存友好）。 

通常情况下你需要添加和删除元素，所以默认使用vector；如果你不需要修改容器大小，那么使用array。

即使其他容器看起来更适合，比如map查找O(logN)，list的中间插入比较高效，但对于大小不超过几KB的容器，vector通常有更好的表现。

**注意** string不应该被用作单字符的容器。string是个文本字符串。如果你想要一个字符容器，请使用vector/</*char_type*//>或array/</*char_type*//>代替。

**例外** 如果你有充分的理由使用另一个容器，就用它来代替。比如：
- 如果容器适合你的需求，但你不需要容器是可变的，就用std::array代替。
- 如果你需要一个字典式的查找容器，保证O(k)和O(logN)的查找，容器会比较大，比如超过几KB，而且你经常需要插入，以至于维护一个排序的vector是不可行的，
那么继续使用unordered\_map或map来代替。

**注意** 要初始化一个有多元素的vector，使用()初始化。要用一个元素的列表来初始化一个vector，使用{}初始化。
```cpp
vector<int> v1(20);  // v1 has 20 elements with the value 0 (vector<int>{})
vector<int> v2 {20}; // v2 has 1 element with the value 20
```

**Enforcement**
- 如果vector构造之后它的size不会改变（比如因为它是const，或者是因为没有非const函数被调用），那么应该标记出，建议用array替换。

<br/>
### **SL.con3:避免越界**

**原因** 读写超出元素的分配范围，通常会导致不好的错误，错误的结果，崩溃，和破坏安全。

**注意** 适用于元素范围的标准库函数都有（或可以有）边界安全的重载，可以用span传递。标准类型如vector，可以被修改为在边界配置文件下执行边界检查
（以一种兼容的方式，如添加合约），或与at()一起使用。
理想情况下，界内保证应该被静态地强制执行。比如说：
- 一个range-for的循环不能超出它所应用的容器的范围
- v.begin()，v.end()很容易被认定为是边界安全的

这样的循环和任何未经检查/不安全的代码一样快。
通常情况下，简单的预先检查可以消除对个别索引的检查需求。例如：
- 对于v.begin()，v.begin()+i，可以很容易地通过v.size()来检查i
这样的循环可以比单独检查元素访问快得多。

**Example,bad**
```cpp
void f()
{
    array<int, 10> a, b;
    memset(a.data(), 0, 10);         // BAD, and contains a length error (length = 10 * sizeof(int))
    memcmp(a.data(), b.data(), 10);  // BAD, and contains a length error (length = 10 * sizeof(int))
}
```
另外，std::array<>::fill()或std::fill()甚至是一个空的初始化器都比memset()更好。

**Example,good**
```cpp
void f()
{
    array<int, 10> a, b, c{};       // c is initialized to zero
    a.fill(0);
    fill(b.begin(), b.end(), 0);    // std::fill()
    fill(b, 0);                     // std::ranges::fill()

    if ( a == b ) {
      // ...
    }
}
```

**Example**
如果代码使用的是未经修改的标准库，那么仍有一些变通方法，能够以边界安全的方式使用std::array和std::vector。代码可以在每个类上调用.at()成员函数，
这将导致抛出一个std::out\_of\_range异常。或者，代码可以调用at()自由函数，这将导致在违反边界时的快速失败（或自定义动作）。
```cpp
void f(std::vector<int>& v, std::array<int, 12> a, int i)
{
    v[0] = a[0];        // BAD
    v.at(0) = a[0];     // OK (alternative 1)
    at(v, 0) = a[0];    // OK (alternative 2)

    v.at(0) = a[i];     // BAD
    v.at(0) = a.at(i);  // OK (alternative 1)
    v.at(0) = at(a, i); // OK (alternative 2)
}
```

**Enforcement**
- 对任何未进行边界检查的标准库函数的调用发出诊断？？？对于被禁止的功能列表插入链接。

本规则也是[bounds profile的一部分](#boundsprofile)

<br/>
### **SL.con4:不是简单可拷贝的参数不要用memset或memcpy**

**原因** 这样做会混乱对象的语义（例如，通过覆盖一个vptr）

**注意** Similarly for (w)memset, (w)memcpy, (w)memmove, and (w)memcmp

**Example**
```cpp
struct base {
    virtual void update() = 0;
};

struct derived : public base {
    void update() override {}
};

void f(derived& a, derived& b) // goodbye v-tables
{
    memset(&a, 0, sizeof(derived));
    memcpy(&a, &b, sizeof(derived));
    memcmp(&a, &b, sizeof(derived));
}
```
相反，定义适当的默认初始化、复制和比较函数
```cpp
void g(derived& a, derived& b)
{
    a = {};    // default initialize
    b = a;     // copy
    if (a == b) do_something(a, b);
}
```

**Enforcement**
- 函数用于不可琐碎复制的类应该被标记出

<br/>
## **SL.str:String**

文本操作是一个巨大的话题，std::string并没有涵盖所有的内容。本节主要试图澄清std::string与char\*、zstring、string\_view和gsl::span\<char\>的关系。
非ASCII字符集和编码（例如wchar\_t、Unicode和UTF-8）的重要问题将在其他地方讨论。

String摘要：
- [SL.str.1:使用std::string来拥有字符序列](#slstr1使用stdstring来拥有字符序列)
- [SL.str.2:使用std::string\_view或gsl::span\<char\>来引用字符序列](#slstr2使用stdstring_view或gslspanchar来引用字符序列)
- [SL.str.3:使用zstring或czstring来引用一个C风格的、以零为结尾的字符序列](#slstr3使用zstring或czstring来引用一个c风格的以零为结尾的字符序列)
- [SL.str.4:使用char\*来引用单字符](#slstr4使用char来引用单字符)
- [SL.str.5:使用std::byte来指代不一定代表字符的字节](#slstr5使用stdbyte来指代不一定代表字符的字节)
- [SL.str.10:当你需要执行locale敏感的字符串操作时，使用std::string](#slstr10当你需要执行locale敏感的字符串操作时使用stdstring)
- [SL.str.11:当你需要改变一个字符串时，使用gsl::span\<char\>而不是std::string\_view](#slstr11当你需要改变一个字符串时使用gslspanchar而不是stdstring_view)
- [SL.str.12:将s后缀用于表示标准库字符串的字符串文字](#slstr12将s后缀用于表示标准库字符串的字符串文字)

另外还有：
- [F.24:span](#f24span)
- [F.24:zstring](#f24zstring)

<br/>
### **SL.str.1:使用std::string来拥有字符序列**

**原因** string正确地处理分配、所有权、复制、逐步扩展，并提供各种有用的操作。

**Example**
```cpp
vector<string> read_until(const string& terminator)
{
    vector<string> res;
    for (string s; cin >> s && s != terminator; ) // read a word
        res.push_back(s);
    return res;
}
```
注意>>和!=是如何为字符串提供的（作为有用操作的例子），并且没有显式分配、去分配或范围检查（字符串负责这些）。

在C++17中，我们可能使用string\_view作为参数，而不是const string&，以便让调用者有更多的灵活性。
```cpp
vector<string> read_until(string_view terminator)   // C++17
{
    vector<string> res;
    for (string s; cin >> s && s != terminator; ) // read a word
        res.push_back(s);
    return res;
}
```

**Example,bad** 不要在需要非琐碎内存管理的操作中使用C风格的字符串
```cpp
char* cat(const char* s1, const char* s2)   // beware!
    // return s1 + '.' + s2
{
    int l1 = strlen(s1);
    int l2 = strlen(s2);
    char* p = (char*) malloc(l1 + l2 + 2);
    strcpy(p, s1, l1);
    p[l1] = '.';
    strcpy(p + l1 + 1, s2, l2);
    p[l1 + l2 + 1] = 0;
    return p;
}
```
我们做得对吗？调用者会记得释放返回的指针吗？这段代码能通过安全审查吗？

**注意** 在没有测量的情况下，不要认为字符串比low-level技术慢，要记住不是所有的代码都是性能关键的。不要过早地优化。

<br/>
### **SL.str.2:使用std::string\_view或gsl::span\<char\>来引用字符序列**

**原因** std::string\_view或gsl::span\<char\>提供了对字符序列的简单和（潜在的）安全的访问，与这些序列的分配和存储方式无关。

**Example**
```cpp
vector<string> read_until(string_view terminator);

void user(zstring p, const string& s, string_view ss)
{
    auto v1 = read_until(p);
    auto v2 = read_until(s);
    auto v3 = read_until(ss);
    // ...
}
```

**注意** std::string\_view (C++17) is read-only.

<br/>
### **SL.str.3:使用zstring或czstring来引用一个C风格的、以零为结尾的字符序列**

**原因** 可读性。说明意图。一个普通的char\*可以是一个指向单个字符的指针，一个指向字符数组的指针，一个指向C风格（零结尾）字符串的指针，
甚至是一个小整数。区分这些选择可以防止误解和错误。

**Example**
```cpp
void f1(const char* s); // s is probably a string
```
我们所知道的是，它应该是nullptr或者至少指向一个字符。
```cpp
void f1(zstring s);     // s is a C-style string or the nullptr
void f1(czstring s);    // s is a C-style string constant or the nullptr
void f1(std::byte* s);  // s is a pointer to a byte (C++17)
```

**注意** 除非有理由，否则不要将C风格的字符串转换为字符串。

**注意** 像其他的 "普通指针 "一样，zstring不应该代表所有权。

**注意** 有数十亿行的C++"在那里"，大多数使用char\*和const char\*而没有记录意图。它们的使用方式多种多样，
包括代表所有权和作为内存的通用指针（而不是void\*）。很难将这些用途分开，所以这条准则也很难遵守。
这是C和C++程序中错误的主要来源之一，所以在可行的情况下，值得遵循这一准则。 

**Enforcement**
- Flag uses of [] on a char\*
- Flag uses of delete on a char\*
- Flag uses of free() on a char\*

<br/>
### **SL.str.4:使用char\*来引用单字符**

**原因** 在目前的代码中，char\*的各种用法是一个主要的错误来源。

**Example,bad**
```cpp
char arr[] = {'a', 'b', 'c'};

void print(const char* p)
{
    cout << p << '\n';
}

void use()
{
    print(arr);   // run-time error; potentially very bad
}
```
数组arr不是一个C风格的字符串，因为它不是以零为结尾。

**Enforcement**
- Flag uses of [] on a char\*

<br/>
### **SL.str.5:使用std::byte来指代不一定代表字符的字节**

**原因** 使用char\*来表示一个不一定是字符的指针，会造成混乱，并使有价值的优化失效。

<br/>
### **SL.str.10:当你需要执行locale敏感的字符串操作时，使用std::string**

**原因** std::string支持标准库中的locale设施。

<br/>
### **SL.str.11:当你需要改变一个字符串时，使用gsl::span\<char\>而不是std::string_view**

**原因** std::string\_view只读。

**Enforcement** 编译器将对试图写到string\_view的行为进行标记。

<br/>
### **SL.str.12:将s后缀用于表示标准库字符串的字符串文字**

**原因** 直接表达一个想法可以最大限度地减少错误。

**Example**
```cpp
auto pp1 = make_pair("Tokyo", 9.00);         // {C-style string,double} intended?
pair<string, double> pp2 = {"Tokyo", 9.00};  // a bit verbose
auto pp3 = make_pair("Tokyo"s, 9.00);        // {std::string,double}    // C++14
pair pp4 = {"Tokyo"s, 9.00};                 // {std::string,double}    // C++17
```

<br/>
## **SL.io:Iostream**

iostream是一个类型安全、可扩展、格式化和非格式化的流式I/O库。它支持多种（和用户可扩展的）缓冲策略和多种locale。
它可以用于传统的I/O、对内存的读写（字符串流）以及用户定义的扩展，如跨网络的流（ASIO：尚未标准化）。

iostream规则摘要：
- [SL.io.1:只有在必须使用时才使用字符级输入](#slio1只有在必须使用时才使用字符级输入)
- [SL.io.2:在读取时，考虑不符合格式的输入](#slio2在读取时考虑不符合格式的输入)
- [SL.io.3:优先使用iostream做IO](#slio3优先使用iostream做io)
- [SL.io.10:除非你使用printf-family函数，否则请调用ios\_base::sync\_with\_stdio(false)](#slio10除非你使用printf-family函数否则请调用ios_basesync_with_stdiofalse)
- [SL.io.50:避免endl](#slio50避免endl)

<br/>
### **SL.io.1:只有在必须使用时才使用字符级输入**

**原因** 除非你真的只是处理单个字符，否则使用字符级输入会导致用户代码对字符进行潜在的错误和潜在的低效率的符号组成。

**Example**
```cpp
char c;
char buf[128];
int i = 0;
while (cin.get(c) && !isspace(c) && i < 128)
    buf[i++] = c;
if (i == 128) {
    // ... handle too long string ....
}
```
以下更简单也可能更快：
```cpp
string s;
s.reserve(128);
cin >> s;
```
reserve(128)可能是不值得的。

<br/>
### **SL.io.2:在读取时，考虑不符合格式的输入**

**原因** 错误通常最好尽快处理。如果输入没有被验证，每个函数都必须被写成应付坏数据（而这是不实际的）。 

<br/>
### **SL.io.3:优先使用iostream做IO**

**原因** 安全，灵活且可扩展。

**Example**
```cpp
// write a complex number:
complex<double> z{ 3, 4 };
cout << z << '\n';
```
complex是一个用户定义的类型，其I/O的定义不需要修改iostream库。

**Example**
```cpp
// read a file of complex numbers:
for (complex<double> z; cin >> z; )
    v.push_back(z);
```

**讨论** iostreams与printf()系列的比较 人们经常（而且经常正确地）指出，printf()系列与iostreams相比有两个优势：格式化的灵活性和性能。
这必须与iostreams处理用户定义的类型的可扩展性、对安全违规的弹性、隐式内存管理和locale处理等优势进行权衡。
如果你需要I/O性能，你几乎总是可以做得比printf()更好。

gets()、scanf()使用的%s和printf()使用%s存在安全隐患（容易发生缓冲区溢出，一般来说容易出错）。
C11定义了一些 "可选扩展"，对其参数进行额外检查。如果在你的C库中存在，gets\_s()、scanf\_s()和printf\_s()可能是更安全的替代方法，但它们仍然不是类型安全的。

**Enforcement** \<cstdio\>和\<stdio.h\>选择性标记出。

<br/>
### **SL.io.10:除非你使用printf-family函数，否则请调用ios_base::sync_with_stdio(false)**

**原因** 将iostreams与printf风格的I/O同步可能代价很高。cin和cout默认与printf同步。

**Example**
```cpp
int main()
{
    ios_base::sync_with_stdio(false);
    // ... use iostreams ...
}
```

<br/>
### **SL.io.50:避免endl**

**原因** endl操纵器主要等同于'\n'和"\n"；最常用的是，它只是通过做多余的flush()来减慢输出速度。与printf风格的输出相比，这种减慢可能很明显。

**Example**
```cpp
cout << "Hello, World!" << endl;    // two output operations and a flush
cout << "Hello, World!\n";          // one output operation and no flush
```

**注意** 对于cin/cout（和类似的）交互，没有理由进行flush，那是自动完成的。对于向文件的写入，很少需要flush。

**注意** 对于字符串流（特别是ostringstream），插入一个endl完全等同于插入一个'\n'字符，但同样在这种情况下，endl可能会明显慢一些。
endl并不负责产生一个平台特定的行结束序列（比如Windows上的"\r\n"）。所以对于一个字符串流，s << endl只是插入一个字符，'\n'。

**注意** 除了性能问题（偶尔很重要），'\n' 和 endl 之间的选择几乎完全是审美的。

<br/>
## **SL.regex:Regex**

\<regex\>是标准的C++正则表达式库。它支持各种正则表达式模式约定。

<br/>
## **SL.chrono:Time**

\<chrono\>（定义于std::chono命名空间）提供了time\_point和duration的概念，以及输出各种单位的时间的函数。它提供了用于注册time\_point的时钟。

<br/>
## **SL.C:C标准库**

C标准库规则摘要：
- [S.C.1:不要使用setjmp/longjmp](#sc1不要使用setjmplongjmp)

<br/>
### **S.C.1:不要使用setjmp/longjmp**

**原因** longjmp忽略了析构，从而使所有依赖RAII的资源管理策略失效。

**Enforcement** 所有出现的longjmp和setjmp的情况都要标记出。

<br/>
### **SL.1:尽可能使用库**

<br/>
### **SL.2:优先使用标准库**

稳定、良好维护和广泛可用。

<br/>
### **SL.3:不要在std命名空间内增加非标准实体**

**原因** 对std的添加可能会改变其他符合标准的代码的含义。对std的添加可能会与标准的未来版本发生冲突。

<br/>
### **SL.4:以类型安全方式使用标准库**

**原因** 因为，很明显，打破这个规则会导致未定义行为、内存损坏和其他各种不良错误。

<br/>
# **A:架构理念**
本节包含了关于更高层次的架构理念和库的想法。

架构理念摘要：
- [A.1:将稳定的代码与不太稳定的代码分开](#a1将稳定的代码与不太稳定的代码分开)
- [A.2:将潜在的可重复使用的部件表达为一个库](#a2将潜在的可重复使用的部件表达为一个库)
- [A.4:库之间不应该有循环](#a4库之间不应该有循环)

<br/>
### **A.1:将稳定的代码与不太稳定的代码分开**

隔离不太稳定的代码有利于其单元测试、接口改进、重构和最终的废弃。

<br/>
### **A.2:将潜在的可重复使用的部件表达为一个库**

一个库是一个声明和定义的集合，它被维护、记录，并被运送到一起。一个库可以是一组头文件（"纯头文件库"）或一组头文件加上一组对象文件。
你可以静态或动态地将一个库链接到一个程序中，或者你可以#include一个纯头文件的库。

<br/>
### **A.4:库之间不应该有循环**

**原因**
- 库循环使构建过程复杂化。
- 库循环很难理解并且可能会引入不确定性（未指定的行为）。

**注意** 一个库可以在其组件的定义中包含循环引用。比如？？？
然而，一个库不应该依赖另一个依赖它的库。

<br/>
# **NR:非规则和神话**
本节包含在某地流行的规则和准则，但我们故意不推荐。我们非常清楚，在某些时候和某些地方，这些规则是有意义的，而且我们自己有时也会使用它们。
然而，在我们推荐和支持的准则的编程风格中，这些"非规则"会造成伤害。

即使在今天，也会有一些背景下的规则是合理的。例如，缺乏合适的工具支持会使异常在硬实时系统中不适用，但请不要天真地相信"普通智慧"
（例如，关于"效率"的无根据的陈述）；这种"智慧"可能是基于几十年前的信息或来自与C++属性完全不同的语言（例如，C或Java）的经验。

非规则摘要：
- [NR.1:不要坚持认为所有的声明都应该放在函数的顶部](#nr1不要坚持认为所有的声明都应该放在函数的顶部)
- [NR.2:不要坚持在一个函数中只有一个返回语句](#nr2不要坚持在一个函数中只有一个返回语句)
- [NR.3:不要避免异常](#nr3不要避免异常)
- [NR.4:不要坚持把每个类的定义放在自己的源文件中](#nr4不要坚持把每个类的定义放在自己的源文件中)
- [NR.5:不要使用两阶段初始化](#nr5不要使用两阶段初始化)
- [NR.6:不要把所有的清理动作放在一个函数的结尾，不要goto exit](#nr6不要把所有的清理动作放在一个函数的结尾不要goto-exit)
- [NR.7:不要让所有的数据成员成为protected](#nr7不要让所有的数据成员成为protected)

<br/>
### **NR.1:不要坚持认为所有的声明都应该放在函数的顶部**

**原因** 所有的声明都在上面的规则是旧的编程语言的遗留问题，它不允许在语句之后初始化变量和常量。这导致了程序的冗长，以及由未初始化和错误初始化的变量引起的更多错误。

**Example, bad**
```cpp
int use(int x)
{
    int i;
    char c;
    double d;

    // ... some stuff ...

    if (x < i) {
        // ...
        i = f(x, d);
    }
    if (i < x) {
        // ...
        i = g(x, c);
    }
    return i;
}
```
未初始化的变量和它的使用之间的距离越大，出现错误的机会就越大。幸运的是，编译器可以捕捉到许多"先使用后设置"的错误。
不幸的是，编译器无法捕捉到所有这样的错误，而且不幸的是，这些错误并不总是像这个小例子中那样简单地被发现。

**替代**
- 总是初始化一个对象
- 在你需要使用一个变量（或常数）之前，不要引入它。

<br/>
### **NR.2:不要坚持在一个函数中只有一个返回语句**

**原因** 单return规则可能会导致不必要的复杂的代码和引入额外的状态变量。特别是，单return规则使得错误检查更难集中在函数的顶部。

**Example**
```cpp
template<class T>
//  requires Number<T>
string sign(T x)
{
    if (x < 0)
        return "negative";
    if (x > 0)
        return "positive";
    return "zero";
}
```
如果只使用单return，我们将不得不采取以下措施
```cpp
template<class T>
//  requires Number<T>
string sign(T x)        // bad
{
    string res;
    if (x < 0)
        res = "negative";
    else if (x > 0)
        res = "positive";
    else
        res = "zero";
    return res;
}
```
这既长又可能效率较低。函数越大、越复杂，变通的方法就越痛苦。当然，许多简单的函数由于其固有的逻辑比较简单，自然只有一个返回。

**Example**
```cpp
int index(const char* p)
{
    if (!p) return -1;  // error indicator: alternatively "throw nullptr_error{}"
    // ... do a lookup to find the index for p
    return i;
}
```
如果我们应用这个规则，我们会得到这样的结果
```cpp
int index2(const char* p)
{
    int i;
    if (!p)
        i = -1;  // error indicator
    else {
        // ... do a lookup to find the index for p
    }
    return i;
}
```
注意，我们（故意）违反了反对未初始化变量的规则，因为这种风格通常会导致这种情况。另外，这种风格也是对使用goto exit非规则的一种诱惑。

**替代**
- 保持函数简短简单
- 请自由使用多个返回语句（并抛出异常）。

<br/>
### **NR.3:不要避免异常**

**原因**
- 异常低效
- 容易导致错误和泄漏
- 性能不可预测
- 异常处理的运行时支持占用了太多的空间

我们没有办法解决这个问题，让所有人都满意。毕竟，关于异常的讨论已经持续了40多年了。有些语言没有异常就不能使用，但有些语言却不支持异常。
这就导致了使用和不使用异常的强烈传统，也导致了激烈的争论。

然而，我们可以简单地概述一下为什么我们认为异常是通用编程的最佳选择，并在这些准则的背景下。支持和反对的简单论点往往是不确定的。
在一些专门的应用中，异常确实可能是不合适的（例如，不支持对处理异常的成本进行可靠估计的硬实时系统）。

依次考虑对异常情况的主要反对意见：
- 异常情况是低效的。与什么相比？在比较的时候，要确保处理的是同一组错误，而且是同等的处理。特别是，不要将一个看到错误就立即终止的程序与一个在记录错误之前仔细清理资源的程序进行比较。是的，有些系统的异常处理实现很差；
有时，这种实现迫使我们使用其他的错误处理方法，但这并不是异常的根本问题。当使用效率论证时--在任何情况下--都要注意你是否有好的数据，能够真正深入地了解所讨论的问题。
- 异常会导致泄漏和错误。它们不会。如果你的程序是一个由指针组成的老鼠窝，而没有一个资源管理的整体策略，那么无论你做什么都会有问题。如果你的系统由一百万行这样的代码组成，你可能无法使用异常，但这是过度和无纪律地使用指针的问题，而不是异常的问题。在我们看来，你需要RAII来使基于异常的错误处理变得简单和安全--比替代方案更简单和安全。
- 异常性能是不可预测的。如果你是在一个硬实时系统中，你必须保证在给定的时间内完成任务，你需要工具来支持这种保证。据我们所知，这样的工具是不存在的（至少对大多数程序员来说不存在）
- 异常处理的运行时支持占用了太多的空间。这可能是小型（通常是嵌入式）系统的情况。然而，在放弃异常处理之前，考虑一下使用错误代码的一致的错误处理会需要多少空间，以及未能捕捉到一个错误会造成什么损失。

许多，可能是大多数，与异常有关的问题都源于与混乱的旧代码交互的历史需求。

使用异常的基本论据是：
- 他们明确区分了错误的return和普通的return
- 他们不能被遗忘或忽视
- 它们可以被系统地使用

记住
- 异常是用来报告错误的（在C++中；其他语言对异常可能有不同的用途）。
- 异常不是针对可以在局部处理的错误。
- 不要试图在每个函数中捕捉每一个异常（这很乏味，很笨拙，而且会导致代码变慢）。
- 异常不是针对那些在发生不可恢复的错误后需要立即终止模块/系统的错误。

**替代**
- RAII
- 契约/断言。使用GSL的Expects和Ensures（直到我们得到语言支持的contract）

<br/>
### **NR.4:不要坚持把每个类的定义放在自己的源文件中**

**原因** 将每个类放在自己的文件中所产生的文件数量很难管理，而且会降低编译速度。单独的类很少是一个好的维护和分发的逻辑单位。

**替代** 使用包含逻辑上连贯的类和函数集的命名空间。

<br/>
### **NR.5:不要使用两阶段初始化**

**原因** 将初始化分成两部分会导致较弱的不变性，更复杂的代码（必须处理半结构化的对象），和错误（当我们没有一致地正确处理半结构化的对象时）。

**Example, bad**
```cpp
// Old conventional style: many problems

class Picture
{
    int mx;
    int my;
    int * data;
public:
    // main problem: constructor does not fully construct
    Picture(int x, int y)
    {
        mx = x;         // also bad: assignment in constructor body
                        // rather than in member initializer
        my = y;
        data = nullptr; // also bad: constant initialization in constructor
                        // rather than in member initializer
    }

    ~Picture()
    {
        Cleanup();
    }

    // ...

    // bad: two-phase initialization
    bool Init()
    {
        // invariant checks
        if (mx <= 0 || my <= 0) {
            return false;
        }
        if (data) {
            return false;
        }
        data = (int*) malloc(mx*my*sizeof(int));   // also bad: owning raw * and malloc
        return data != nullptr;
    }

    // also bad: no reason to make cleanup a separate function
    void Cleanup()
    {
        if (data) free(data);
        data = nullptr;
    }
};

Picture picture(100, 0); // not ready-to-use picture here
// this will fail..
if (!picture.Init()) {
    puts("Error, invalid picture");
}
// now have an invalid picture object instance.
```

**Example, good**
```cpp
class Picture
{
    int mx;
    int my;
    vector<int> data;

    static int check_size(int size)
    {
        // invariant check
        Expects(size > 0);
        return size;
    }

public:
    // even better would be a class for a 2D Size as one single parameter
    Picture(int x, int y)
        : mx(check_size(x))
        , my(check_size(y))
        // now we know x and y have a valid size
        , data(mx * my) // will throw std::bad_alloc on error
    {
        // picture is ready-to-use
    }

    // compiler generated dtor does the job. (also see C.21)

    // ...
};

Picture picture1(100, 100);
// picture is ready-to-use here...

// not a valid size for y,
// default contract violation behavior will call std::terminate then
Picture picture2(100, 0);
// not reach here...
```

**替代** 
- 始终在构造函数中建立一个类的不变量。
- 不要在需要之前定义一个对象。

<br/>
### **NR.6:不要把所有的清理动作放在一个函数的结尾，不要goto exit**

**原因** goto是容易出错的。这种技术是一种类似RAII的资源和错误处理的预异常技术。

**Example, bad**
```cpp
void do_something(int n)
{
    if (n < 100) goto exit;
    // ...
    int* p = (int*) malloc(n);
    // ...
    if (some_error) goto_exit;
    // ...
exit:
    free(p);
}
```

**替代**
- 使用异常和RAII
- 非RAII资源，使用finally

<br/>
### **NR.7:不要让所有的数据成员成为protected**

**原因** protected的数据是错误的来源。protected的数据可以在不同的地方从无限制的代码中被操纵。protected数据是相当于global数据的类层次结构。

**替代** 见C.133

<br/>
# **Pro:profiles**

理想情况下，我们会遵循所有的准则。这样就能得到最干净、最规范、最不容易出错的代码，而且往往是最快的。不幸的是，这通常是不可能的，因为我们必须把我们的代码放入大型的代码库中，并使用现有的库。通常，这样的代码已经写了几十年了，并不遵循这些准则。我们必须以逐步采用为目标。

无论我们采取什么样的逐步采用的策略，我们都需要能够应用相关的指南集来首先解决一些问题，而把其他的问题留到以后。当一些但不是所有的指南被认为与代码库相关，或者如果一套专门的指南要应用于专门的应用领域，类似的"相关指南"的想法就变得很重要。我们把这样一组相关的指南称为"profiles"。我们的目标是让这样的准则集是连贯的，以便它们一起帮助我们达到一个特定的目标，例如"没有范围错误"或"静态类型安全"。每个配置文件都是为了消除一类错误。孤立地执行"随机"规则比提供明确的改进更有可能对代码库造成破坏。

一个"profile"是一组确定的、可移植的、可执行的规则子集（即限制），旨在实现特定的保证。"确定性"意味着它们只需要局部分析，可以在编译器中实现（尽管它们不需要）。"可移植的执行"意味着它们就像语言规则一样，所以程序员可以指望不同的执行工具对相同的代码给出相同的答案。

使用这种语言配置文件编写的无警告代码被认为是符合该配置文件的。符合规范的代码被认为是安全的，因为该规范所针对的安全属性。符合规范的代码不会成为该属性错误的根本原因，尽管这些错误可能是由其他代码、库或外部环境引入程序的。配置文件还可能引入额外的库类型来简化一致性并鼓励正确的代码。

Profiles summary:
- [Pro.type: Type safety](#prosafetytype-safety-profile)
- [Pro.bounds: Bounds safety](#proboundsbounds-safety-profile)
- [Pro.lifetime: Lifetime safety](#prolifetimelifetime-safety-profile)

在未来，我们预计将定义更多的配置文件，并为现有的配置文件增加更多的检查。候选者包括：
- 缩小算术提升/转换（可能是单独的安全算术配置文件的一部分）。
- 从负数浮点到无符号积分类型的算术转换（同上）。
- 选定的未定义行为。从Gabriel Dos Reis为WG21研究小组制定的UB列表开始。
- 选定的未指定的行为。解决可移植性问题。
- 违反const。大部分是由编译器完成的，但我们可以抓住不恰当的cast和对const的不足使用。

启用一个配置文件是由实施定义的；通常，它是在使用的分析工具中设置的。
要抑制配置文件检查的执行，请在语言合同上放置一个抑制注释。比如说。
```cpp
[[suppress(bounds)]] char* raw_find(char* p, int n, char x)    // find x in p[0]..p[n - 1]
{
    // ...
}
```
现在，raw\_find()可以尽情地窜改内存。很明显，抑制应该是非常罕见的。

<br/>
## **Pro.safety:Type-safety profile**

这个profile使构建正确使用类型的代码变得更加容易，并避免了无意中的类型双关。它通过关注消除类型违规的主要来源，包括不安全地使用cast和union，来做到这一点。

出于本节的目的，类型安全被定义为不以不遵守其定义类型规则的方式使用变量的属性。作为类型 T 访问的内存不应是实际包含不相关类型 U 对象的有效内存。请注意，当与边界安全和生命期安全相结合时，安全的目的是完整的。

本规范的实现应将源代码中的以下模式识别为不符合要求，并发出诊断。

Type safety profile summary:
- Type.1: Avoid casts:
 1. Don’t use reinterpret\_cast; A strict version of Avoid casts and prefer named casts.
 2. Don’t use static\_cast for arithmetic types; A strict version of Avoid casts and prefer named casts.
 3. Don’t cast between pointer types where the source type and the target type are the same; A strict version of Avoid casts.
 4. Don’t cast between pointer types when the conversion could be implicit; A strict version of Avoid casts.
- Type.2: Don’t use static\_cast to downcast: Use dynamic\_cast instead.
- Type.3: Don’t use const\_cast to cast away const (i.e., at all): Don’t cast away const.
- Type.4: Don’t use C-style (T)expression or functional T(expression) casts: Prefer construction or named casts or T{expression}.
- Type.5: Don’t use a variable before it has been initialized: always initialize.
- Type.6: Always initialize a member variable: always initialize, possibly using default constructors or default member initializers.
- Type.7: Avoid naked union: Use variant instead.
- Type.8: Avoid varargs: Don’t use va\_arg arguments.

**影响** 
使用类型安全配置文件，您可以相信每个操作都应用于有效对象。可以抛出异常以指示无法静态检测到的错误（在编译时）。请注意，只有当我们同时拥有边界安全和生命安全时，这种类型安全才是完整的。如果没有这些保证，一个内存区域可以独立于存储在其中的一个、多个对象或对象的一部分而被访问。

<br/>
## **Pro.bounds:Bounds safety profile**

这个配置文件使得在分配的内存块的范围内操作的代码更容易构建。它通过专注于消除违反边界的主要来源：指针算术和数组索引来实现。这个配置文件的核心特征之一是限制指针只能指代单个对象，而不是指代数组。
我们将边界安全定义为程序不使用对象访问分配给它的范围之外的内存的属性。边界安全只有在与类型安全和生命期安全相结合时才是完整的，后者涵盖了其他允许违反边界的不安全操作。

Bounds safety profile summary:
- Bounds.1: Don’t use pointer arithmetic. Use span instead: Pass pointers to single objects (only) and Keep pointer arithmetic simple.
- Bounds.2: Only index into arrays using constant expressions: Pass pointers to single objects (only) and Keep pointer arithmetic simple.
- Bounds.3: No array-to-pointer decay: Pass pointers to single objects (only) and Keep pointer arithmetic simple.
- Bounds.4: Don’t use standard-library functions and types that are not bounds-checked: Use the standard library in a type-safe manner.

**影响**
界限安全意味着对一个对象的访问--特别是对数组的访问--不会超出该对象的内存分配。这消除了一大类阴险和难以发现的错误，包括（不）著名的"缓冲区溢出"错误。这就堵住了安全漏洞，也堵住了内存损坏的一个重要来源（当越界写入时）。即使越界访问"只是一个读"，它也会导致违反不变性（当被访问的不是假定的类型时）和"神秘的值"。

<br/>
## **Pro.lifetime:Lifetime safety profile**

通过一个不指向任何东西的指针进行访问是一个主要的错误来源，在许多传统的C或C++风格的编程中很难避免。例如，一个指针可能是未初始化的，是nullptr，指向一个数组的范围之外，或者指向一个被删除的对象。

Lifetime safety profile summary:
- Lifetime.1: Don’t dereference a possibly invalid pointer: detect or avoid.

**影响**
一旦通过样式规则、静态分析和库支持的组合完全强制执行，该配置文件
- 消除了C++中令人讨厌的错误的主要来源之一
- 消除了一个潜在的安全漏洞的主要来源
- 通过消除多余的"偏执狂"检查来提高性能
- 增加对代码正确性的信心
- 通过执行关键的C++语言规则，避免了未定义的行为 

<br/>
# **NL:命名和布局建议**
一致的命名和布局是有帮助的。如果没有其他原因的话，因为它最大限度地减少了 "我的风格比你的风格好 "的争论。然而，现在有很多很多不同的风格，人们对它们充满热情（赞成和反对）。此外，大多数现实世界的项目包括来自许多来源的代码，所以为所有代码统一一种风格往往是不可能的。在许多用户请求指导之后，我们提出了一套规则，如果你没有更好的想法，你可以使用这套规则，但真正的目的是一致性，而不是任何特定的规则集。集成开发环境和工具可以帮助（也可以阻碍）。

- [NL.1:不要在注释中写可以在代码中明确说明的内容](#nl1不要在注释中写可以在代码中明确说明的内容)
- [NL.2:在注释中说明意图](#nl2在注释中说明意图)
- [NL.3:保持注释的简洁性](#nl3保持注释的简洁性)
- [NL.4:保持一致的缩进风格](#nl4保持一致的缩进风格)
- [NL.5:避免在名称中对类型信息进行编码](#nl5避免在名称中对类型信息进行编码)
- [NL.7:使名字的长度与它的范围的长度大致成正比](#nl7使名字的长度与它的范围的长度大致成正比)
- [NL.8:使用一致的命名方式](#nl8使用一致的命名方式)
- [NL.9:只对宏名称使用ALL\_CAPS](#nl9只对宏名称使用all_caps)
- [NL.10:倾向于下划线风格的名称](#nl10倾向于下划线风格的名称)
- [NL.11:让文字可读](#nl11让文字可读)
- [NL.15:少用空格](#nl15少用空格)
- [NL.16:使用常规的类成员声明顺序](#nl16使用常规的类成员声明顺序)
- [NL.17:使用K&R衍生的布局](#nl17使用kr衍生的布局)
- [NL.18:使用C++风格的声明布局](#nl18使用c风格的声明布局)
- [NL.19:避免使用容易被误读的名称](#nl19避免使用容易被误读的名称)
- [NL.20:不要将两个语句放在同一行中](#nl20不要将两个语句放在同一行中)
- [NL.21:每个声明声明一个名称（仅）](#nl21每个声明声明一个名称仅)
- [NL.25:不要使用void作为参数类型](#nl25不要使用void作为参数类型)
- [NL.26:使用传统的const符号](#nl26使用传统的const符号)
- [NL.27:代码文件使用.cpp后缀，接口文件使用.h后缀](#nl27代码文件使用cpp后缀接口文件使用h后缀)

这些规则大多是审美性的，程序员们持有强烈的意见。集成开发环境也倾向于有默认值和一系列的备选方案。这些规则是建议遵循的默认值，除非你有理由不这样做。

有评论说，命名和布局是非常个人化和/或任意的，我们不应该试图对它们进行"立法"。我们不是在"立法"（见前一段）。然而，我们有许多人要求提供一套命名和布局惯例，以便在没有外部约束的情况下使用。
更具体和详细的规则更容易执行。

这些规则与为支持Stroustrup的《编程》而编写的PPP风格指南中的建议非常相似《使用C++的原则和实践》。

<br/>
### **NL.1:不要在注释中写可以在代码中明确说明的内容**
**原因** 编译器不读注释。注释不如代码精确。注释不像代码那样持续更新。

**Example, bad**
```cpp
auto x = m * v1 + vv;   // multiply m with v1 and add the result to vv
```

<br/>
### **NL.2:在注释中说明意图**
**原因** 代码说的是做了什么，而不是应该做什么。通常情况下，意图可以比实施更清楚、更简洁地表达出来。

**Example**
```cpp
void stable_sort(Sortable& c)
    // sort c in the order determined by <, keep equal elements (as defined by ==) in
    // their original relative order
{
    // ... quite a few lines of non-trivial code ...
}
```

**注意** 如果注释和代码不一致，两者都有可能是错误的。

<br/>
### **NL.3:保持注释的简洁性**

**原因** 冗长会减慢理解速度，并通过将代码分散在源文件中使代码更难阅读。

**注意** 使用可理解的英语。我可能会说流利的丹麦语，但大多数程序员不是；我的代码的维护者可能也不是。避免使用SMS行话，注意你的语法、标点符号和大小写。力求专业性，而不是"酷"。

<br/>
### **NL.4:保持一致的缩进风格**
**原因** 可读性。避免 "愚蠢的错误"。

**Example, bad**
```cpp
int i;
for (i = 0; i < max; ++i); // bug waiting to happen
if (i == j)
    return i;
```

**注意** 在if (...)、for (...)和while (...)后面总是缩进语句通常是个好主意。
```cpp
if (i < 0) error("negative argument");

if (i < 0)
    error("negative argument");
```

**Enforcement** 使用工具。

<br/>
### **NL.5:避免在名称中对类型信息进行编码**
**基本原理** 如果名称反映类型而不是功能，则很难更改用于提供该功能的类型。此外，如果更改了变量的类型，则必须修改使用它的代码。最大限度地减少无意的转换。

**Example, bad**
```cpp
void print_int(int i);
void print_string(const char*);

print_int(1);          // repetitive, manual type matching
print_string("xyzzy"); // repetitive, manual type matching
```

**Example, good**
```cpp
void print(int i);
void print(string_view);    // also works on any string-like sequence

print(1);              // clear, automatic type matching
print("xyzzy");        // clear, automatic type matching
```

**注意** 具有编码类型的名称要么冗长要么含糊。
```cpp
printS  // print a std::string
prints  // print a C-style string
printi  // print an int
```
要求像匈牙利符号这样的技术来编码一个类型，在非类型化语言中已经被使用了，但在像C++这样的强静态类型化语言中一般是不必要的，而且是积极有害的，因为注释会过时（疣子就像注释一样腐烂），而且它们会干扰语言的良好使用（用同名和重载解析代替）。

**注意** 一些样式使用非常通用的（不是特定于类型的）前缀来表示变量的一般用途。
```cpp
auto p = new User();
auto p = make_unique<User>();
// note: "p" is not being used to say "raw pointer to type User,"
//       just generally to say "this is an indirection"

auto cntHits = calc_total_of_hits(/*...*/);
// note: "cnt" is not being used to encode a type,
//       just generally to say "this is a count of something"
```
这并不有害，也不属于本准则的范围，因为它没有对类型信息进行编码。

**注意** 有些风格将成员与局部变量和/或全局变量区分开来。
```cpp
struct S {
    int m_;
    S(int m) : m_{abs(m)} { }
};
```
这并不有害，也不属于本准则的范围，因为它没有对类型信息进行编码。

**注意** 像C++一样，有些风格将类型与非类型区分开来。例如，通过大写类型名称，而不是函数和变量的名称。
```cpp
typename<typename T>
class HashTable {   // maps string to T
    // ...
};

HashTable<int> index;
```
这并不有害，也不属于本准则的范围，因为它没有对类型信息进行编码。

<br/>
### **NL.7:使名字的长度与它的范围的长度大致成正比**

**原理** 范围越大，发生混淆和意外的名称冲突的机会越大。

**Example**
```cpp
double sqrt(double x);   // return the square root of x; x must be non-negative

int length(const char* p);  // return the number of characters in a zero-terminated C-style string

int length_of_string(const char zero_terminated_array_of_char[])    // bad: verbose

int g;      // bad: global variable with a cryptic name

int open;   // bad: global variable with a short, popular name
```
在有限的范围内，用p表示指针，用x表示浮点变量是常规的，不会产生混淆。

<br/>
### **NL.8:使用一致的命名方式**
**原理** 命名和命名方式的一致性增加了可读性。

**注意** 有很多风格，当你使用多个库时，你不能遵循他们所有不同的惯例。选择一种"内部风格"，但让"进口"的库保持其原始风格。

**Example** ISO标准，只使用小写字母和数字，用下划线分隔单词。
- int
- vector
- my\_map
避免使用双下划线\_\_。

**Example** Stroustrup:ISO标准，但在自己的类型和概念上使用大写字母。
- int
- vector
- My\_map

**Example** CamelCase:将多字标识符中的每个字大写。
- int
- vector
- MyMap
- myMap
有些惯例将第一个字母大写，有些则不。

**注意** 在使用缩略语和标识符的长度方面，尽量保持一致。
```cpp
int mtbf {12};
int mean_time_between_failures {12}; // make up your mind
```

<br/>
### **NL.9:只对宏名称使用ALL_CAPS**
**原因** 为了避免混淆那些遵守范围和类型规则的名称的宏。

**Example**
```cpp
void f()
{
    const int SIZE{1000};  // Bad, use 'size' instead
    int v[SIZE];
}
```

**注意** 这条规则适用于非宏的符号常数。
```cpp
enum bad { BAD, WORSE, HORRIBLE }; // BAD
```

**Enforcement**
- 小写字母的宏要标记出
- 非宏ALL\_CAPS名称要标记出

<br/>
### **NL.10:倾向于下划线风格的名称**
**原因** 使用下划线来分隔名字的各个部分是最初的C和C++风格，在C++标准库中使用。

**注意** 这个规则是默认的，只有在你有选择的情况下才使用。通常情况下，你没有选择，必须遵循既定的风格以保持一致性。对一致性的需要胜过个人品味。
这是在你没有约束或更好的想法时的建议。这条规则是在许多人要求提供指导后添加的。

**Example** Stroustrup:ISO标准，但在自己的类型和概念上使用大写字母。
- int
- vector
- My\_map

<br/>
### **NL.11:让文字可读**
**原因** 可读性。

**Example** 使用数字分隔符以避免长串的数字
```cpp
auto c = 299'792'458; // m/s2
auto q2 = 0b0000'1111'0000'0000;
auto ss_number = 123'456'7890;
```

**Example** 在需要说明的地方使用文字后缀
```cpp
auto hello = "Hello!"s; // a std::string
auto world = "world";   // a C-style string
auto interval = 100ms;  // using <chrono>
```

**注意** 文字不应该像“魔法常量”一样散布在整个代码中，但让它们在定义的地方可读仍然是一个好主意。在一长串整数中很容易打错字。

**Enforcement** 标记长数字序列。问题是如何定义 "长"；也许是7。

<br/>
### **NL.15:少用空格**
**原因** 太多的空格会使文本变大并分散注意力。

**Example, bad**
```cpp
#include < map >

int main(int argc, char * argv [ ])
{
    // ...
}
```

**Example**
```cpp
#include <map>

int main(int argc, char* argv[])
{
    // ...
}
```

**注意** 一些IDE有自己的观点，并增加了分散注意力的空格。
这是在你没有约束或更好的想法时的建议。这条规则是在许多人要求提供指导后添加的。

**注意** 我们重视合理的留白，认为它对可读性有很大帮助。只是不要过度。

<br/>
### **NL.16:使用常规的类成员声明顺序**
**原因** 成员的常规顺序可以提高可读性。
在声明一个类时，请使用以下顺序
- types: classes, enums, and aliases (using)
- constructors, assignments, destructor
- functions
- data

Use the public before protected before private order.

这是在你没有约束或更好的想法时的建议。这条规则是在许多人要求提供指导后添加的。

**Example**
```cpp
class X {
public:
    // interface
protected:
    // unchecked function for use by derived class implementations
private:
    // implementation details
};
```

**Example** 有时，成员的默认顺序与将公共接口与实现细节分开的愿望相冲突。在这种情况下，私有类型和函数可以与私有数据放在一起。
```cpp
class X {
public:
    // interface
protected:
    // unchecked function for use by derived class implementations
private:
    // implementation details (types, functions, and data)
};
```

**Example, bad** 避免多个具有一种访问权限的声明块（如public）分散在具有不同访问权限的声明块（如private）中。
```cpp
class X {   // bad
public:
    void f();
public:
    int g();
    // ...
};
```
使用宏来声明成员组，往往会导致违反任何排序规则。然而，无论如何，使用宏会掩盖正在表达的内容。

**Enforcement** 偏离建议顺序的地方要标记出。会有很多不遵循这一规则的旧代码。

<br/>
### **NL.17:使用K&R衍生的布局**
**原因** 这是最初的C和C++布局。它很好地保留了垂直空间。它很好地区分了不同的语言结构（如函数和类）。

**注意** 在C++的背景下，这种风格通常被称为 "Stroustrup"。
这是在你没有约束条件或更好的想法时的一种建议。这条规则是在许多人要求提供指导后添加的。

**Example**
```cpp
struct Cable {
    int x;
    // ...
};

double foo(int x)
{
    if (0 < x) {
        // ...
    }

    switch (x) {
    case 0:
        // ...
        break;
    case amazing:
        // ...
        break;
    default:
        // ...
        break;
    }

    if (0 < x)
        ++x;

    if (x < 0)
        something();
    else
        something_else();

    return some_value;
}
```
注意if和( )之间的空格。

**注意** 每条语句、if的分支和for的主体都要用单独的行。

**注意** 类和结构的{不在一个单独的行上，但函数的{在一个单独的行上。

**注意** 将用户定义的类型的名称大写，以区别于标准库中的类型。

**注意** 不要将函数名称大写。

**Enforcement** 如果你想强制执行，请使用IDE来重新格式化。

<br/>
### **NL.18:使用C++风格的声明布局**
**原因** C风格的布局强调在表达式和语法中的使用，而C++风格则强调类型。在表达式参数中的使用不适用于引用。

**Example**
```cpp
T& operator[](size_t);   // OK
T &operator[](size_t);   // just strange
T & operator[](size_t);   // undecided
```

**注意** 这是在你没有约束或更好的想法时的建议。这条规则是在许多人要求提供指导后添加的。

<br/>
### **NL.19:避免使用容易被误读的名称**
**原因** 可读性。不是每个人的屏幕和打印机都能很容易地分辨出所有的字符。我们很容易混淆相似的拼写和略微错误的单词。

**Example**
```cpp
int oO01lL = 6; // bad

int splunk = 7;
int splonk = 8; // bad: splunk and splonk are easily confused
```

<br/>
### **NL.20:不要将两个语句放在同一行中**
**原因** 可读性。当一行有更多的语句时，真的很容易被忽略。

**Example**
```cpp
int x = 7; char* p = 29;    // don't
int x = 7; f(x);  ++x;      // don't
```

<br/>
### **NL.21:每个声明声明一个名称（仅）**
**原因** 可读性。尽量减少对声明语法的混淆。

见ES.10

<br/>
### **NL.25:不要使用void作为参数类型**
**原因** 它很冗长，只有在 C 兼容性很重要的情况下才需要。

**Example**
```cpp
void f(void);   // bad

void g();       // better
```

**注意** 甚至Dennis Ritchie也认为void f(void)是一种可憎的行为。你可以为C语言中的这种可恶行为提出论据，当时函数原型很少，所以禁止了。
```cpp
int f();
f(1, 2, "weird but valid C89");   // hope that f() is defined int f(a, b, c) char* c; { /* ... */ }
```
会造成重大问题，但在21世纪和C++中不会。

<br/>
### **NL.26:使用传统的const符号**
**原因** 更多程序员更熟悉传统符号。大型代码库的一致性。

**Example**
```cpp
const int x = 7;    // OK
int const y = 9;    // bad

const int *const p = nullptr;   // OK, constant pointer to constant int
int const *const p = nullptr;   // bad, constant pointer to constant int
```

**注意** 我们很清楚，你可以说"坏"的例子比标有"OK"的例子更有逻辑性，但它们也让更多的人感到困惑，尤其是依赖使用更常见的、传统的OK风格的教材的新手。
与以往一样，请记住，这些命名和布局规则的目的是一致性，而美学上的差异是巨大的。
这是在你没有约束或更好的想法时的建议。这条规则是在许多人要求提供指导后添加的。

**Enforcement** 作为类型的后缀使用的const要标记出。

<br/>
### **NL.27:代码文件使用.cpp后缀，接口文件使用.h后缀**
**原因** 这是一个长期的惯例。但一致性更重要，所以如果你的项目使用其他东西，否则就遵循这个。

**注意** 这种惯例反映了一种常见的使用模式。头文件更经常与C共享，以编译成C++和C，C通常使用.h，将所有头文件命名为.h，而不是为那些打算与C共享的头文件设置不同的扩展名。另一方面，实现文件很少与C共享，所以通常应该与.c文件区分开来，所以通常最好将所有C++的实现文件命名为其他名称（如.cpp）。

.h和.cpp这些特定的名字不是必须的（只是作为默认推荐），其他的名字也被广泛使用。例如：.hh、.C和.cxx。使用这样的名字是等同的。在本文档中，我们把.h和.cpp作为头文件和实现文件的缩写，尽管实际的扩展名可能不同。

你的IDE（如果你使用的话）可能对后缀有强烈的意见。

**Example**
```cpp
// foo.h:
extern int a;   // a declaration
extern void foo();

// foo.cpp:
int a;   // a definition
void foo() { ++a; }
```
foo.h提供了foo.cpp的接口。全局变量最好避免使用。

**Example, bad**
```cpp
// foo.h:
int a;   // a definition
void foo() { ++a; }
```
\#include \<foo.h\>在一个程序中出现两次，你就会因为违反两个单一定义规则而得到一个链接错误。

**Enforcement**
- 非传统的文件名标记出
- 检查.h和.cpp（和等价物）是否遵循以下规则

<br/>
# **Appendix C:Discussion**
本节包含关于规则和规则集的后续材料。特别是，我们在这里提出了进一步的理由，更长的例子，以及对替代方案的讨论。

<br/>
### **讨论：按照成员声明的顺序定义和初始化成员变量**
成员变量总是按照它们在类定义中声明的顺序进行初始化，所以在构造函数初始化列表中要按照这个顺序来写它们。以不同的顺序写它们只会使代码变得混乱，因为它不会按照你看到的顺序运行，这可能会使你很难看到与顺序相关的错误。

```cpp
class Employee {
    string email, first, last;
public:
    Employee(const char* firstName, const char* lastName);
    // ...
};

Employee::Employee(const char* firstName, const char* lastName)
  : first(firstName),
    last(lastName),
    // BAD: first and last not yet constructed
    email(first + "." + last + "@acme.com")
{}
```
在这个例子中，email将在first和last之前被构造，因为它被首先声明。这意味着它的构造函数会过早地尝试使用first和last--不仅仅是在它们被设置为所需的值之前，而是在它们被构造之前。

如果类定义和构造函数主体在不同的文件中，成员变量声明的顺序对构造函数正确性的远距离影响将更难发现。

<br/>
### **讨论：使用=、{}和（）作为初始化**

<br/>
### **讨论：如果你在初始化过程中需要"虚拟行为"，请使用工厂函数**
如果你的设计希望从基类的构造函数或析构函数中虚拟派发到派生类中，比如f和g，你需要其他技术，比如后构造函数--一个单独的成员函数，调用者必须调用它来完成初始化，它可以安全地调用f和g，因为在成员函数中虚拟调用行为正常。这方面的一些技术在参考文献中显示。下面是一个非详尽的选项列表。
- Pass the buck: 只要记录下用户代码必须在构造对象后立即调用后初始化函数。
- Post-initialize lazily: 在成员函数的第一次调用时进行。基类中的一个布尔标志告诉我们是否已经进行了后建。
- Use virtual base class semantics: 语言规则规定，最外派生类的构造函数决定了哪个基类的构造函数将被调用；你可以利用这一点为你带来好处。
- Use a factory function: 这样，你可以很容易地强制调用一个后构造函数。

下面是最后一个选项的例子：
```cpp
class B {
public:
    B()
    {
        /* ... */
        f(); // BAD: C.82: Don't call virtual functions in constructors and destructors
        /* ... */
    }

    virtual void f() = 0;
};

class B {
protected:
    class Token {};

public:
    // constructor needs to be public so that make_shared can access it.
    // protected access level is gained by requiring a Token.
    explicit B(Token) { /* ... */ }  // create an imperfectly initialized object
    virtual void f() = 0;

    template<class T>
    static shared_ptr<T> create()    // interface for creating shared objects
    {
        auto p = make_shared<T>(typename T::Token{});
        p->post_initialize();
        return p;
    }

protected:
    virtual void post_initialize()   // called right after construction
        { /* ... */ f(); /* ... */ } // GOOD: virtual dispatch is safe
    }
};


class D : public B {                 // some derived class
protected:
    class Token {};

public:
    // constructor needs to be public so that make_shared can access it.
    // protected access level is gained by requiring a Token.
    explicit D(Token) : B{ B::Token{} } {}
    void f() override { /* ...  */ };

protected:
    template<class T>
    friend shared_ptr<T> B::create();
};

shared_ptr<D> p = D::create<D>();    // creating a D object
```
这种设计需要以下准则：
- 像D这样的派生类决不能暴露出一个可公开调用的构造函数。否则，D的用户可以创建不调用post\_initialize的D对象。
- 分配被限制在operator new。然而，B可以重写new。
- D必须定义一个与B选择的参数相同的构造函数。然而，定义create的几个重载可以缓解这个问题；而且重载甚至可以在参数类型上进行模板化。

如果满足了上述要求，设计就能保证post\_initialize对于任何完全构造的B派生对象都被调用了。post\_initialize不需要是虚拟的；但是，它可以自由调用虚拟函数。

总而言之，没有一种后构造技术是完美的。最差的技术通过简单地要求调用者手动调用后构造器来回避整个问题。即使是最好的，也需要用不同的语法来构造对象（在编译时容易检查）和/或来自派生类作者的合作（在编译时无法检查）。

<br/>
### **讨论：使基类的析构成为公共的和虚拟的，或protected和非虚拟的**
销毁应该是虚拟的吗？也就是说，是否应该允许通过一个指向基类的指针进行销毁？如果是的话，那么基类的析构必须是公共的，以便可以调用，并且是虚拟的，否则调用它将导致未定义的行为。否则，它应该是受保护的，这样只有派生类可以在它们自己的析构函数中调用它，而且是非虚拟的，因为它不需要有虚拟行为。

**Example** 基类的常见情况是它打算拥有公共派生类，因此调用代码几乎肯定会使用类似shared\_ptr\<base\>的东西：
```cpp
class Base {
public:
    ~Base();                   // BAD, not virtual
    virtual ~Base();           // GOOD
    // ...
};

class Derived : public Base { /* ... */ };

{
    unique_ptr<Base> pb = make_unique<Derived>();
    // ...
} // ~pb invokes correct destructor only when ~Base is virtual
```
在比较少见的情况下，比如策略类，该类被用作基类是为了方便，而不是为了多态行为。我们建议将这些析构变成受保护的非虚拟的。
```cpp
class My_policy {
public:
    virtual ~My_policy();      // BAD, public and virtual
protected:
    ~My_policy();              // GOOD
    // ...
};

template<class Policy>
class customizable : Policy { /* ... */ }; // note: private inheritance
```

**注意** 这条简单的准则说明了一个微妙的问题，并反映了继承和面向对象设计原则的现代用法。

对于基类Base，调用代码可能会尝试通过指向Base的指针销毁派生对象，例如在使用unique\_ptr\<Base\>时。如果Base的析构函数是公共的和非虚拟的（默认），它可能会在实际指向派生对象的指针上意外调用，在这种情况下，尝试删除的行为是未定义的。这种情况导致旧的编码标准强制要求所有基类析构函数必须是虚拟的。这太过分了（即使这是常见的情况）；相反，规则应该是当且仅当它们是公共的时才使基类析构函数成为虚拟的。

编写基类就是定义一个抽象（见第35至37项）。回顾一下，对于参与该抽象的每个成员函数，你需要决定：
- 它是否应该有虚拟的行为。
- 它是否应该对所有使用指向Base的指针的调用者公开，否则就是一个隐藏的内部实现细节。

正如第39项中所描述的，对于一个普通的成员函数，可以选择允许它通过指向Base的指针被非虚拟地调用（但是如果它调用了虚拟函数，例如在NVI或模板方法模式中，可能会有虚拟行为），虚拟地调用，或者根本不调用。NVI模式是一种避免公共虚拟函数的技术。

销毁可以被视为另一种操作，尽管具有使非虚拟调用危险或错误的特殊语义。因此，对于基类析构函数，选择是允许通过指向Base的指针虚拟调用它还是根本不允许调用它；“非虚拟”不是一种选择。因此，如果基类析构函数可以被调用（即公共的），则它是虚拟的，否则就是非虚拟的。

请注意，NVI模式不能应用于析构函数，因为构造函数和析构函数不能进行深度虚拟调用。参见第39和55项。

推论：在编写基类时，始终显式编写析构函数，因为隐式生成的析构函数是公共的和非虚拟的。如果默认主体很好并且您只是编写函数以赋予它适当的可见性和虚拟性，那么您总是可以=default实现。

**例外** 一些组件体系结构（例如 COM 和 CORBA）不使用标准的删除机制，并促进不同的对象处理协议。遵循他自己的模式和习语，并酌情调整本指南。
还要考虑这种罕见的情况：
- B既是一个基类，也是一个可以被自己实例化的具体类，因此B对象的析构必须是公共的，才能被创建和销毁。
- 然而，B也没有虚函数，也不是为了多态地使用，所以尽管析构是公共的，但它不需要是虚的。

然后，尽管析构函数必须是公共的，但不把它变成虚拟的压力可能很大，因为作为第一个虚拟函数，它将产生所有的运行时类型开销，而增加的功能应该永远不需要。
在这种罕见的情况下，你可以让析构成为公共的和非虚拟的，但明确规定进一步派生的对象不得作为B的多态使用。这就是std::unary\_function的做法。

然而，一般来说，要避免具体的基类（见第35项）。例如，unary\_function是一个类型定义的集合，它从未被打算独立地实例化。给它一个公共的析构是没有意义的；更好的设计是遵循本项目的建议，给它一个受保护的非虚拟的析构。

<br/>
### **讨论：Usage of noexcept**

<br/>
### **讨论：析构函数、释放和swap绝不能失败**

决不允许从析构函数、资源释放函数（例如，operator delete）或使用throw的swap函数报告错误。如果这些操作可能失败，则几乎不可能编写出有用的代码，即使出现问题，重试也几乎没有任何意义。具体来说，其析构函数可能引发异常的类型被明确禁止与 C++ 标准库一起使用。大多数析构函数现在默认是隐式的 noexcept 。

**Example**
```cpp
class Nefarious {
public:
    Nefarious() { /* code that could throw */ }    // ok
    ~Nefarious() { /* code that could throw */ }   // BAD, should not throw
    // ...
};
```
- Nefarious对象即使作为局部变量也很难安全使用。

```cpp
 void test(string& s)
 {
     Nefarious n;          // trouble brewing
     string copy = s;      // copy the string
 } // destroy copy and then n
```
在这里，复制s可能会抛出异常，如果抛出异常，并且如果n的析构随后也抛出，程序将通过std::terminate退出，因为两个异常不能同时传播。

- 带有Nefarious成员或基的类也很难安全使用，因为它们的析构必须调用Nefarious的析构，并且同样受到它的不良行为的影响。

```cpp
 class Innocent_bystander {
     Nefarious member;     // oops, poisons the enclosing class's destructor
     // ...
 };

 void test(string& s)
 {
     Innocent_bystander i;  // more trouble brewing
     string copy2 = s;      // copy the string
 } // destroy copy and then i
```
在这里，如果构造copy2抛出，我们也有同样的问题，因为i的析构现在也可以抛出，如果是这样，我们会调用std::terminate。

- 你也不能可靠地创建全局或静态的Nefarious对象。

```cpp
static Nefarious n;       // oops, any destructor exception can't be caught
```

- 你无法可靠地创建Nefarious数组。

```cpp
 void test()
 {
     std::array<Nefarious, 10> arr; // this line can std::terminate()
 }
```
在存在抛出的析构的情况下，数组的行为是无法定义的，因为没有任何合理的回滚行为可以被设计出来。试想一下：如果第四个对象的构造函数被抛出异常，编译器可以生成什么样的代码来构造一个数组，而代码不得不放弃，并在其清理模式下尝试调用已经构造好的对象的析构......而这些析构中的一个或多个被抛出？没有令人满意的答案。

- 你不能在标准容器中使用Nefarious对象。

```cpp
 std::vector<Nefarious> vec(10);   // this line can std::terminate()
```
标准库禁止所有与它一起使用的析构抛出。你不能将Nefarious对象存储在标准容器中，也不能将其与标准库的任何其他部分一起使用。

**注意** 这些是不能失败的关键函数，因为它们对于事务编程中的两个关键操作是必需的：如果在处理过程中遇到问题，则退出工作，如果没有问题发生，则提交工作。如果没有办法使用no-fail操作安全地退出，那么no-fail回滚就不可能实现。如果没有办法使用no-fail操作（尤其是但不限于swap）安全地提交状态更改，那么就不可能实现无失败提交。

请考虑以下C++标准中的建议和要求：

> If a destructor called during stack unwinding exits with an exception, terminate is called (15.5.1). So destructors should generally catch exceptions and not let them propagate out of the destructor. –[C++03] §15.2(3)

> No destructor operation defined in the C++ Standard Library (including the destructor of any type that is used to instantiate a standard-library template) will throw an exception. –[C++03] §17.4.4.8(3)

请考虑以下C++标准中的建议和要求：
释放函数，包括特别重载的operator delete和operator delete[]，属于同一类，因为它们通常也在清理期间使用，特别是在异常处理期间，以退出需要撤消的部分工作。除了析构函数和释放函数，常见的错误安全技术还依赖于永不失败的交换操作——在这种情况下，不是因为它们用于实现有保证的回滚，而是因为它们用于实现有保证的提交。例如，这里是类型T的operator=的惯用实现，它执行复制构造，然后调用无失败交换：这些都是不能失败的关键函数，因为它们是事务性编程中两个关键操作所必需的：如果在处理过程中遇到问题，就把工作退回去，如果没有问题发生，就提交工作。如果没有办法使用不失败的操作安全地回退，那么不失败的回滚就不可能实现。如果没有办法使用无故障操作安全地提交状态变化（特别是，但不限于swap），那么无故障提交就不可能实现。
```cpp
T& T::operator=(const T& other)
{
    auto temp = other;
    swap(temp);
    return *this;
}
```
幸运的是，在释放资源时，失败的范围肯定更小。如果使用异常作为错误报告机制，请确保此类函数处理其内部处理可能产生的所有异常和其他错误。（对于例外情况，只需将析构函数所做的所有敏感内容包装在try/catch(...) 块中。）这尤其重要，因为析构函数可能会在危机情况下调用，例如未能分配系统资源（例如、内存、文件、锁、端口、窗口或其他系统对象）。
当使用异常作为你的错误处理机制时，总是通过声明这些函数noexcept来记录这种行为。(参见第75项）。

<br/>
## **一致地定义复制、移动和析构**

**注意** 如果你定义了一个复制构造函数，你也必须定义一个复制赋值操作符。

**注意** 如果你定义了一个移动构造函数，你也必须定义一个移动赋值操作符。

**Example**
```cpp
class X {
public:
    X(const X&) { /* stuff */ }

    // BAD: failed to also define a copy assignment operator

    X(x&&) noexcept { /* stuff */ }

    // BAD: failed to also define a move assignment operator

    // ...
};

X x1;
X x2 = x1; // ok
x2 = x1;   // pitfall: either fails to compile, or does something suspicious
```
如果你定义了一个析构，你不应该使用编译器生成的复制或移动操作；你可能需要定义或抑制复制和/或移动。
```cpp
class X {
    HANDLE hnd;
    // ...
public:
    ~X() { /* custom stuff, such as closing hnd */ }
    // suspicious: no mention of copying or moving -- what happens to hnd?
};

X x1;
X x2 = x1; // pitfall: either fails to compile, or does something suspicious
x2 = x1;   // pitfall: either fails to compile, or does something suspicious
```
如果你定义了复制，并且任何base或成员有一个定义了移动操作的类型，你也应该定义一个移动操作。
```cpp
class X {
    string s; // defines more efficient move operations
    // ... other data members ...
public:
    X(const X&) { /* stuff */ }
    X& operator=(const X&) { /* stuff */ }

    // BAD: failed to also define a move construction and move assignment
    // (why wasn't the custom "stuff" repeated here?)
};

X test()
{
    X local;
    // ...
    return local;  // pitfall: will be inefficient and/or do the wrong thing
}
```
如果你定义了任何一个拷贝构造、拷贝赋值运算符或析构，你可能应该定义其他的。

**注意** 如果你需要定义这五个功能中的任何一个，就意味着你需要它做比其默认行为更多的事情--而且这五个功能是不对称地相互关联的。情况是这样的:
- 如果你写/禁用了复制构造函数或复制赋值运算符中的任何一个，你可能需要对另一个做同样的事情。如果一个做了"特殊"的工作，另一个可能也应该如此，因为这两个函数应该有类似的效果。(参见第53项，它对这一点进行了单独的阐述）。
- 如果你明确地编写了复制函数，你可能需要编写析构。如果在复制构造函数中的"特殊"工作是分配或复制一些资源（例如，内存、文件、套接字），你需要在析构函数中去回收。
- 如果你明确地写了析构，你可能需要明确地写或禁止复制。如果你不得不写一个非琐碎的析构，往往是因为你需要手动释放对象所持有的资源。如果是这样，很可能那些资源需要仔细复制，那么你需要注意对象的复制和分配方式，或者完全禁用复制。
在许多情况下，使用RAII"拥有"对象持有适当封装的资源可以消除自己编写这些操作的需要。(见第13项)。
倾向于使用编译器生成的（包括=default）特殊成员；只有这些成员可以被归类为"trivial"，而且至少有一个主要的标准库供应商对拥有trivial的特殊成员的类进行了大量优化。这可能会成为普遍的做法。

**例外** 当任何一个特殊函数的声明只是为了使其成为非公共的或虚拟的，但没有特殊的语义，这并不意味着其他函数是需要的。在极少数情况下，拥有奇怪类型成员的类（如引用成员）是一个例外，因为它们有特殊的拷贝语义。在一个拥有引用的类中，你可能需要编写拷贝构造函数和赋值运算符，但默认的析构函数已经做了正确的事情。(注意，使用一个引用成员几乎总是错误的）。


资源管理规则摘要：
- [提供强大的资源安全；也就是说，永远不要泄露任何你认为是资源的东西](#讨论提供强大的资源安全也就是说永远不要泄露任何你认为是资源的东西)
- [持有不属于句柄的资源时永远不要返回或抛出异常](#讨论持有不属于句柄的资源时永远不要返回或抛出异常)
- [一个裸指针或引用绝不是一个资源句柄](#讨论一个裸指针或引用绝不是一个资源句柄)
- [不要让一个指针的寿命超过它所指向的对象](#讨论不要让一个指针的寿命超过它所指向的对象)
- [使用模板来表达容器（和其他资源处理）](#讨论使用模板来表达容器和其他资源处理)
- [按值返回容器（依靠移动或复制优化来提高效率）](#讨论按值返回容器依靠移动或复制优化来提高效率)
- [如果一个类是一个资源句柄，它需要一个构造函数，一个析构函数，以及复制和/或移动操作](#讨论如果一个类是一个资源句柄它需要一个构造函数一个析构函数以及复制和或移动操作)
- [如果一个类是一个容器，给它一个初始化列表构造函数](#讨论如果一个类是一个容器给它一个初始化列表构造函数)

<br/>
### **讨论：提供强大的资源安全；也就是说，永远不要泄露任何你认为是资源的东西**
**原因** 防止泄漏。泄漏会导致性能下降、神秘的错误、系统崩溃和安全违规。

**替代** 让每一个资源作为某个类的对象来代表，管理其生命周期。

**Example**
```cpp
template<class T>
class Vector {
private:
    T* elem;   // sz elements on the free store, owned by the class object
    int sz;
    // ...
};
```
这个类是一个资源句柄。它管理着T的生命周期。为此，Vector必须定义或删除一组特殊的操作（构造函数，一个析构函数，等等）。

**Enforcement** 
防止泄漏的基本技术是让每个资源都由一个资源句柄拥有，并有一个合适的析构。检查器可以找到"naked new"。给定一个C风格的分配函数列表（例如，fopen()），检查器也可以找到没有被资源句柄管理的使用。一般来说，"裸指针"可以被怀疑地看待，被标记，和/或被分析。如果没有人的输入，一个完整的资源列表是无法生成的（"资源"的定义必然过于笼统），但一个工具可以通过资源列表来"参数化"。

<br/>
### **讨论：持有不属于句柄的资源时永远不要返回或抛出异常**
**原因** 这将是一个泄漏。

**Example**
```cpp
void f(int i)
{
    FILE* f = fopen("a file", "r");
    ifstream is { "another file" };
    // ...
    if (i == 0) return;
    // ...
    fclose(f);
}
```
如果i == 0，一个文件的句柄就会被泄露。另一方面，另一个文件的ifstream将正确地关闭其文件（在销毁时）。如果你必须使用一个显式指针，而不是一个具有特定语义的资源句柄，请使用unique\_ptr或带有自定义删除器的shared\_ptr。
```cpp
void f(int i)
{
    unique_ptr<FILE, int(*)(FILE*)> f(fopen("a file", "r"), fclose);
    // ...
    if (i == 0) return;
    // ...
}
```
或者更好是：
```cpp
void f(int i)
{
    ifstream input {"a file"};
    // ...
    if (i == 0) return;
    // ...
}
```

**Enforcement** 检查器必须认为所有的 "裸指针"是可疑的。检查器可能必须依赖人类提供的资源清单。对于初学者，我们知道标准库的容器、字符串和智能指针。span和string\_view的使用应该有很大的帮助（它们不是资源句柄）。

<br/>
### **讨论：一个裸指针或引用绝不是一个资源句柄**
**原因** 区分是view还是所有者。

**注意** 这与你如何"拼写"指针无关。T\*, T&, Ptr\<T\> 和Range\<T\>不是所有者。

<br/>
### **讨论：不要让一个指针的寿命超过它所指向的对象**
**原因** 为了避免极难发现的错误。解引用这样的指针是未定义的行为，可能导致对类型系统的违反。

**Example**
```cpp
string* bad()   // really bad
{
    vector<string> v = { "This", "will", "cause", "trouble", "!" };
    // leaking a pointer into a destroyed member of a destroyed object (v)
    return &v[0];
}

void use()
{
    string* p = bad();
    vector<int> xx = {7, 8, 9};
    // undefined behavior: x might not be the string "This"
    string x = *p;
    // undefined behavior: we don't know what (if anything) is allocated a location p
    *p = "Evil!";
}
```
v的字符串在退出bad()时被销毁，v本身也是如此。返回的指针指向free store上未分配的内存。这个内存（由p指向）在\*p被执行时可能已经被重新分配了。可能没有字符串可读，通过p写的东西很容易破坏不相关类型的对象。

**Enforcement** 大多数编译器已经对简单的情况提出了警告，并且有信息可以做得更多。考虑从一个函数返回的任何指针都是可疑的。使用容器、资源句柄和view（例如，已知不是资源句柄的span）来降低需要检查的案例数量。对于初学者来说，把每一个带有析构的类都视为资源句柄。

<br/>
### **讨论：使用模板来表达容器（和其他资源处理）**
**原因** 提供静态类型安全的元素操作。

**Example**
```cpp
template<typename T> class Vector {
    // ...
    T* elem;   // point to sz elements of type T
    int sz;
};
```

<br/>
### **讨论：按值返回容器（依靠移动或复制优化来提高效率）**
**原因** 简化代码并消除对显式内存管理的需求。将一个对象带入周围的范围，从而延长其寿命。

**Example**
```cpp
vector<int> get_large_vector()
{
    return ...;
}

auto v = get_large_vector(); //  return by value is ok, most modern compilers will do copy elision
```
另见F.20

**Enforcement** 检查从函数返回的指针和引用，看它们是否被分配给资源句柄（例如，分配给unique\_ptr）。

<br/>
### **讨论：如果一个类是一个资源句柄，它需要一个构造函数，一个析构函数，以及复制和/或移动操作**
**原因** 提供对资源寿命的完全控制。为资源提供一套连贯的操作。

**注意** 如果所有成员都是资源句柄，尽可能依靠默认的特殊操作。
```cpp
template<typename T> struct Named {
    string name;
    T value;
};
```
现在Named有一个默认的构造函数，一个析构函数，以及高效的复制和移动操作，只要T有。

**Enforcement** 一般来说，一个工具不能知道一个类是否是一个资源句柄。然而，如果一个类有一些默认的操作，它应该有所有的操作，如果一个类有一个成员是资源句柄，它应该被认为是资源句柄。

<br/>
### **讨论：如果一个类是一个容器，给它一个初始化列表构造函数**
**原因** 需要一个初始的元素集是很常见的。

**Example**
```cpp
template<typename T> class Vector {
public:
    Vector(std::initializer_list<T>);
    // ...
};

Vector<string> vs { "Nygaard", "Ritchie" };
```
