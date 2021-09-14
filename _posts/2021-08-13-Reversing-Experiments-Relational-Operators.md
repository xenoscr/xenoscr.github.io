---
layout: post
title: "Reversing Experiments: Relational Operators"
published: false
---
# Introduction

I have recently decided to perform a few experiments to learn more about how compilers turn various logic, structures and other things into machine code. I want to focus on how different compilers and compiler flags could change the resulting code as well. I have decided to document the experience to share as well as for my own future reference. If this is useful to others, that's awesome. If not, I have some notes to look at and a better understanding of what compilers do to turn my code into a runnable program.

## Relational Operators

The purpose of this reversing experiment is to learn about how relational operators (i.e. ==, !==, >, <, >=, <=) are handled by compilers. To accomplish this, I wrote the following, very simple, C program that makes use of all of the relational operators in combination with static global variables as well as local variables.

{% highlight c linenos %}
#ifdef _WIN32
#include <windows.h>
#endif
#include <stdio.h>
#include <stdbool.h>

static int a = 1;
static int b = 2;
static int c = 3;
static int d = 4;
static int e = 1;
static int f = 2;
static int g = 3;
static int h = 4;

bool isEqual(int intA, int intB)
{
    if (intA == intB)
        return true;
    return false;
}
    
bool isNotEqual(int intA, int intB)
{
    if (intA != intB)
        return true;
    return false;
}

bool isGreater(int intA, int intB)
{
    if (intA > intB)
        return true;
    return false;
}

bool isLess(int intA, int intB)
{
    if (intA < intB)
        return true;
    return false;
}

bool isGreaterOrEqual(int intA, int intB)
{
    if (intA >= intB)
        return true;
    return false;
}

bool isLessOrEqual(int intA, int intB)
{
    if (intA <= intB)
        return true;
    return false;
}

int main(void)
{
    int a1 = 1;
    int b1 = 2;
    int c1 = 3;
    int d1 = 4;
    int e1 = 1;
    int f1 = 2;
    int g1 = 3;
    int h1 = 4;
    
    bool result = false;
    
    // == checks
    // Two equal local integers
    result = isEqual(a1, e1);
    printf("Evaluated %i == %i: %s\n", a1, e1, result ? "true" : "false");
    // Two unequal local integers
    result = isEqual(a1, h1);
    printf("Evaluated %i == %i: %s\n", a1, h1, result ? "true" : "false");
    // Two equal static global integers
    result = isEqual(a, e);
    printf("Evaluated %i == %i: %s\n", a, e, result ? "true" : "false");
    // Two unequal static global integers
    result = isEqual(a, h);
    printf("Evaluated %i == %i: %s\n", a, h, result ? "true" : "false");
    // Equal local and static integers
    result = isEqual(a1, a);
    printf("Evaluated %i == %i: %s\n", a1, a, result ? "true" : "false");
    // Unequal local and static global integers
    result = isEqual(a1, h);
    printf("Evaluated %i == %i: %s\n", a1, h, result ? "true" : "false");
    
    // != Checks
    // Two unequal local integers
    result = isNotEqual(a1, c1);
    printf("Evaluated %i != %i: %s\n", a1, c1, result ? "true" : "false");
    // Two equal local integers
    result = isNotEqual(d1, h1);
    printf("Evaluated %i != %i: %s\n", d1, h1, result ? "true" : "false");
    // Two unequal static global integers
    result = isNotEqual(b, g);
    printf("Evaluated %i != %i: %s\n", b, g, result ? "true" : "false");
    // Two equal static global integers
    result = isNotEqual(c, g);
    printf("Evaluated %i != %i: %s\n", c, g, result ? "true" : "false");
    // Local and static global unequal integers
    result = isNotEqual(a, c1);
    printf("Evaluated %i != %i: %s\n", a, c1, result ? "true" : "false");
    // Local and static global equal integer
    result = isNotEqual(g1, g);
    printf("Evaluated %i != %i: %s\n", g1, g, result ? "true" : "false");
       
    // > checks
    // Two equal local integers
    result = isGreaterOrEqual(a1, e1);
    printf("Evaluated %i >= %i: %s\n", a1, c1, result ? "true" : "false");
    // Two local integers a > b
    result = isGreaterOrEqual(f1, a1);
    printf("Evaluated %i >= %i: %s\n", f1, a1, result ? "true" : "false");
    // Two local integers a < b
    result = isGreaterOrEqual(h1, b1);
    printf("Evaluated %i >= %i: %s\n", h1, b1, result ? "true" : "false");
    // Two equal static global integers
    result = isGreaterOrEqual(a, e);
    printf("Evaluated %i >= %i: %s\n", a, e, result ? "true" : "false");
    // Two static global integers a > b
    result = isGreaterOrEqual(g, b);
    printf("Evaluated %i >= %i: %s\n", g, b, result ? "true" : "false");
    // Two static global integers a < b
    result = isGreaterOrEqual(b, g);
    printf("Evaluated %i >= %i: %s\n", b, g, result ? "true" : "false");
    // Local and static integers
    result = isGreaterOrEqual(g1, g);
    printf("Evaluated %i >= %i: %s\n", g1, g, result ? "true" : "false");
    // Local and static global integers a > b
    result = isGreaterOrEqual(d1, e);
    printf("Evaluated %i >= %i: %s\n", d1, e, result ? "true" : "false");
    // Local and static global integers a < b
    result = isGreaterOrEqual(e1, d);
    printf("Evaluated %i >= %i: %s\n", e1, d, result ? "true" : "false");
    
    // <= checks
    // Two equal local integers
    result = isLess(a1, e1);
    printf("Evaluated %i <= %i: %s\n", a1, c1, result ? "true" : "false");
    // Two local integers a > b
    result = isLess(f1, a1);
    printf("Evaluated %i <= %i: %s\n", f1, a1, result ? "true" : "false");
    // Two local integers a < b
    result = isLess(h1, b1);
    printf("Evaluated %i <= %i: %s\n", h1, b1, result ? "true" : "false");
    // Two equal static global integers
    result = isLess(a, e);
    printf("Evaluated %i <= %i: %s\n", a, e, result ? "true" : "false");
    // Two static global integers a > b
    result = isLess(g, b);
    printf("Evaluated %i <= %i: %s\n", g, b, result ? "true" : "false");
    // Two static global integers a < b
    result = isLess(b, g);
    printf("Evaluated %i <= %i: %s\n", b, g, result ? "true" : "false");
    // Local and static integers
    result = isLess(g1, g);
    printf("Evaluated %i <= %i: %s\n", g1, g, result ? "true" : "false");
    // Local and static global integers a > b
    result = isLess(d1, e);
    printf("Evaluated %i <= %i: %s\n", d1, e, result ? "true" : "false");
    // Local and static global integers a < b
    result = isLess(e1, d);
    printf("Evaluated %i <= %i: %s\n", e1, d, result ? "true" : "false");
    
    // >= checks
    // Two equal local integers
    result = isGreater(a1, e1);
    printf("Evaluated %i >= %i: %s\n", a1, c1, result ? "true" : "false");
    // Two local integers a > b
    result = isGreater(f1, a1);
    printf("Evaluated %i >= %i: %s\n", f1, a1, result ? "true" : "false");
    // Two local integers a < b
    result = isGreater(h1, b1);
    printf("Evaluated %i >= %i: %s\n", h1, b1, result ? "true" : "false");
    // Two equal static global integers
    result = isGreater(a, e);
    printf("Evaluated %i >= %i: %s\n", a, e, result ? "true" : "false");
    // Two static global integers a > b
    result = isGreater(g, b);
    printf("Evaluated %i >= %i: %s\n", g, b, result ? "true" : "false");
    // Two static global integers a < b
    result = isGreater(b, g);
    printf("Evaluated %i >= %i: %s\n", b, g, result ? "true" : "false");
    // Local and static integers
    result = isGreater(g1, g);
    printf("Evaluated %i >= %i: %s\n", g1, g, result ? "true" : "false");
    // Local and static global integers a > b
    result = isGreater(d1, e);
    printf("Evaluated %i >= %i: %s\n", d1, e, result ? "true" : "false");
    // Local and static global integers a < b
    result = isGreater(e1, d);
    printf("Evaluated %i >= %i: %s\n", e1, d, result ? "true" : "false");
    
    // <= checks
    // Two equal local integers
    result = isLessOrEqual(a1, e1);
    printf("Evaluated %i <= %i: %s\n", a1, c1, result ? "true" : "false");
    // Two local integers a > b
    result = isLessOrEqual(f1, a1);
    printf("Evaluated %i <= %i: %s\n", f1, a1, result ? "true" : "false");
    // Two local integers a < b
    result = isLessOrEqual(h1, b1);
    printf("Evaluated %i <= %i: %s\n", h1, b1, result ? "true" : "false");
    // Two equal static global integers
    result = isLessOrEqual(a, e);
    printf("Evaluated %i <= %i: %s\n", a, e, result ? "true" : "false");
    // Two static global integers a > b
    result = isLessOrEqual(g, b);
    printf("Evaluated %i <= %i: %s\n", g, b, result ? "true" : "false");
    // Two static global integers a < b
    result = isLessOrEqual(b, g);
    printf("Evaluated %i <= %i: %s\n", b, g, result ? "true" : "false");
    // Local and static integers
    result = isLessOrEqual(g1, g);
    printf("Evaluated %i <= %i: %s\n", g1, g, result ? "true" : "false");
    // Local and static global integers a > b
    result = isLessOrEqual(d1, e);
    printf("Evaluated %i <= %i: %s\n", d1, e, result ? "true" : "false");
    // Local and static global integers a < b
    result = isLessOrEqual(e1, d);
    printf("Evaluated %i <= %i: %s\n", e1, d, result ? "true" : "false");
    return 0;
}
{% endhighlight %}

```{css, echo=FALSE}
.watch-out {
  background-color: lightpink;
  border: 3px solid red;
  font-weight: bold;
}
```

```{c class.source="watch-out"}
#import <windows.h>

int main(void)
{
    return 0;
}
```



## Compiling the Code

The above code was compiled for x86 and x64 CPU architectures using the following compilers:

- [MinGW - Minimalist GNU for Windows](https://osdn.net/projects/mingw/)
- [GCC, the GNU Compiler Collection](https://gcc.gnu.org/)
- [Microsoft's Visual Studio 2019 C++ Toolset](https://docs.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-160)





