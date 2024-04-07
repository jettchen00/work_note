# 1. c++关键要点
1. 原则上应该在所有的构造函数前加explicit关键字，防止隐式调用构造函数。
2. copy构造函数与copy赋值函数的调用很容易区分，因为如果存在新对象被定义则一定是调用copy构造函数。
3. c++有着十分固定的“成员初始化次序”，base classes更早于derived classes被初始化，而class的成员变量总是以其声明次序被初始化。
4. 如果你自己没有声明以下函数，则编译器会自动生成一个copy构造函数、一个copy assignment操作符、一个析构函数。另外，如果你没有声明任何构造函数，编译器也会自动生成一个default构造函数。
5. 如果不想使用编译器自动生成的函数，则应该显示将该函数声明成private的（并且只声明而不实现），或者实现1个base class（base class同样只声明private函数），然后派生类继承该base class即可。比如noncopyable类的实现。
6. 当class内至少含有一个virtual函数时（只有此时才代表它会被设计为base class），才将其析构函数声明为virtual的。
7. 确定你的构造函数和析构函数都没有调用virtual函数。
8. 令operator=操作符返回一个reference to *this，并且能够处理自我赋值。
9. 当你编写一个copying函数（copy构造函数或者copy assign operator），请确保会复制所有local成员变量，也会调用所有base classes内的copying函数。不要尝试以某个copying函数实现另一个copying函数，应该将共同机能放进第三个函数中，并由两个copying函数共同调用。
10. 以独立语句将newed对象存储于智能指针内。如果不这样做，一旦中途抛出异常，有可能导致难以察觉的资源泄露。
11. 尽量以pass-by-reference-to-const替换pass-by-value，通常前者更高效，不过对于内置类型、STL迭代器和函数对象，pass-by-value往往更合适。
12. 如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member函数。
13. 如果swap缺省实现版的效率不足，试着做以下事情：1. 提供一个public swap成员函数，让它高效地置换新定义类型的两个对象值。2. 在你的class或者template所在的命名空间内提供一个non-member swap，并令它调用上述swap成员函数。
14. 注意理解模版函数与模版类的全特化与偏特化，但c++中不支持模版函数的偏特化（需要使用技巧才能实现）。