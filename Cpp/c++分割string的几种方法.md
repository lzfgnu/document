### c++分割string的几种方法

1. 使用stl的stream_iterator

   优点:

   * 只依赖标准组件
   * 适用于任何流

   缺点:

   * 只能分割空白符
   * 性能略差
   * 代码量多(operator>>)

   代码:

   ```c++
   std::vector<std::string> split_string(const std::string& s) {
     std::istringstream iss(s);
     return std::vector<std::string>
       (std::istream_iterator<std::string>{iss},
        std::istream_iterator<std::string>{});
   }
   ```

2. 客制化operaotr>>(方法一的改进)

   优点:

   * 适用于任何分隔符
   * 使用于任何流
   * 比方法一更快(因为自定义的operator>>代码量会少些)

   缺点:

   * 分隔符必须在编译期确定
   * 非标准做法
   * 代码量还是有点多

   代码:

   ```c++
   template<char delimiter>
   class WordDelimitedBy : public std::string {};
   
   template<char delimiter>
   std::istream& operator>>(std::istream& is, WordDelimitedBy<delimiter>& output) {
     std::getline(is, output, delimiter);
     return is;
   }
   
   template <char delimiter>
   std::vector<std::string> split_string(const std::string& s) {
     std::istringstream iss(s);
     return std::vector<std::string>
       (std::istream_iterator<WordDelimitedBy<delimiter>>{iss},
        std::istream_iterator<WordDelimitedBy<delimiter>>());
   }
   ```

3. 普通的循环迭代

   优点:

   * 接口清晰
   * 适用于任何分隔符
   * 分隔符在运行时指定

   缺点:

   * 非标准(c++ standard)

   代码:

   ```c++
   std::vector<std::string> split_string(const std::string& s, char delimiter) {
     std::vector<std::string> tokens;
     std::string token;
     std::istringstream tokenStream(s);
     while (std::getline(tokenStream, token, delimiter)) {
       tokens.push_back(token);
     }
     return tokens;
   }
   ```

4. 使用boost::split

   优点:

   * 接口清晰
   * 允许任何分隔符
   * 速度更快

   缺点:

   * 引入boost
   * 输出结果不由函数接口返回

   代码:

   ```c++
   #include <boost/algorithm/string.hpp>
    
   std::string text = "Let me split this into words";
   std::vector<std::string> results;
    
   boost::split(results, text, [](char c){return c == ' ';});
   ```

5. 使用c++20的ranges

   代码:

   ```c++
   std::string text = "Let me split this into words";
   auto splitText = text | view::split(' ') | ranges::to<std::vector<std::<wbr />string>>();
   ```

   