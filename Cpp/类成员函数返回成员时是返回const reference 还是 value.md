### 类成员函数返回成员时是返回const reference 还是 value

```c++
class C {
public:
    ? get_string() const;
private:
	std::string s;
};
```

对于get_string函数返回 const string& 好还是 string 好? 看起来似乎 const ref 更好些, 毕竟避免了copy.(当然了, string有定义移动函数). 

考虑如下的情况:

```c++
void func() {
    C* c = new C();
    const string& s = c->get_string();
    delete c;
}
```



boom!! 由于 c 被 delete 导致了引用失效, 程序将会陷入未定义的行为. 也许没有那个程序员会如此显式的去 delete c , 但是当代码量增多的时候, 你还是有可能在程序中不小心的将 c delete掉. 或者考虑多线程共享一个C实例, 当其中一个线程引用到C.s, 而另一个线程把这个实例delete掉的时候同样会出问题. 所以, 类成员函数返回成员时, 还是尽可能返回value. 不要过于担心、吝啬这点性能. 
