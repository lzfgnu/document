### back_inserter与copy

对于容器间数据的拷贝，dest容器一定要先分配好空间。如果使用back_inserter，会导致性能很低，因为涉及到vector内部内存的再分配。如果copy之前先resize的话，那么空间只用动态分配一次，性能高效的多。

```c++
vector<int> v1{1, 2, 3, 4, 5};
vector<int> v2;
copy(v1.begin(), v1.end(), back_inserter(v2));

// 更好的做法
v2.resize(size);
copy(v1.begin(), v1.end(), v2.begin(), v2.end());
```

