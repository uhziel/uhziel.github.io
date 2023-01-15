---
title: C++中段错误的常见情况
date: 2020-07-30 17:34:14
tags:
  - cpp
---

Program terminated with signal 11, Segmentation fault.

## 空指针访问非虚函数

```c++
include <iostream>
class Foo
{
public:
  Foo() : a(0) {}
  void Bar() { std::cout << "Bar:" << a << std::endl; }
private:
  int a;
};
int main() {
  Foo* f = NULL;
  f->Bar();
  std::cout << "hello" << std::endl;
  return 0;
}

```

```
(gdb) bt
#0  0x00000000004008ef in Foo::Bar (this=0x0) at null.cpp:6
#1  0x00000000004008bb in main () at null.cpp:14
(gdb)
```

## 空指针访问虚函数

```c++
#include <iostream>
class Foo
{
public:
  Foo() : a(0) {}
  virtual void Bar() { std::cout << "Bar:" << a << std::endl; }
private:
  int a;
};

int main() {
  Foo* f = NULL;
  f->Bar();
  std::cout << "hello" << std::endl;
  return 0;
}

```

```
(gdb) bt
#0  0x0000000000400866 in main () at null.cpp:14
(gdb) p f
$1 = (Cannot access memory at address 0x0
```

## 野指针访问虚函数

```c++
#include <iostream>
class Foo
{
public:
  Foo() : a(0) {}
  virtual void Bar() { std::cout << "Bar:" << a << std::endl; }
private:
  int a;
};

int main() {
  Foo* f = new Foo();
  delete f;
  f->Bar();
  std::cout << "hello" << std::endl;
  return 0;
}
```

```
(gdb) bt
#0  0x00000000004009c4 in main () at null.cpp:15
(gdb) p f
$1 = (Foo *) 0x602010
```
