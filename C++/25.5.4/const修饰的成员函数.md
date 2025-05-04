常函数：
- 成员函数后加const后被称为**常函数**
- 常函数内不可以修改成员变量
- 成员属性声明时加关键字mutable后，在常函数中依旧可以修改

常对象：
- 声明对象前加const被称为**常对象**
- 常对象只能调用常函数

示例：
```
#include <iostream>
using namespace std;

class Person {
public:
    void showClass() const {
        m_A = 100;  // 错误：不能修改成员变量
    }
    int m_A;
};

int main() {
    return 0;
}
```

报错如下：
![[res4.png]]

这是因为代码被隐式地转换为了`this->m_A`，而this的本质是指针常量，在成员函数后加上const后，修饰的是this，让this指向的值不可以修改。
在成员变量的前面加上mutable则可在常函数或常对象中进行修改。

const成员函数的限制
- 不能修改类的成员变量（包括非静态数据成员）。
- 不能调用非const的成员函数（因为非const函数可能修改对象状态）。
- 不能访问或修改类的静态成员变量（除非显式声明为mutable static）。

mutable关键字的作用
mutable仅适用于非静态数据成员，且需在声明时指定。
示例：
```
class Counter {
public:
    mutable int count;  // 允许在const函数中修改
    void increment() const { ++count; }
};
```

const对象的限制
常对象（const Person obj;）只能调用const成员函数，且不能初始化非const成员变量。
示例：
```
const Person obj;
obj.showClass();  // 正确（若showClass是const）
obj.m_A = 200;    // 错误（m_A未声明为mutable）
```

const成员函数的返回值
若返回指针或引用，需确保其指向的数据不会被修改。例如：
```
const int* getVal() const { return &m_val; }  // 安全，因m_val不可变
```

const与非const函数的调用场景
- 非const函数可修改对象状态，常用于需要改变数据的场景。
- const函数用于保证数据不变，如计算、查询等操作。





