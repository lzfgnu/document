### 使用std::variant和std::visit来实现运行时多态

1. 传统的实现手法(虚函数)的优缺点

   优点:

   * 语法是语言内置的. 实现起来很自然
   * 如果想要拓展, 可以直接添加新的子类而不影响基类
   * 面向对象
   * 可是使用Base指针来存储所有继承链上的类

   缺点:

   * 虚函数带来的议决开销
   * 因为存在虚指针, 所以可能涉及到动态分配

2. 使用std::variant和std::visit

   优点:

   * 值语义, 无需动态分配
   * 便于添加新方法, 只需要实现新的可调用结构体, 不必对实现类做出改变
   * 无需基类, 并且类与类之间是独立关系.
   * 类型推导: 虚函数要求必须有相同的函数签名. 使用std::visit的话, 则没有这个要求

   缺点:

   * 难于添加新的类型
   * 也许会消耗更多的内存, 因为sizeof(variant) = max{classA, classB....}
   * 类型推导: 没有一个统一的签名, 这既是优点也是缺点
   * 每一个操作(虚函数调用)都要求一个visitor, 如何组织这些visitor是个问题

   实例:

   ```c++
   struct SimpleLabel {
     std::string str;  
   };
   struct DateLabel {
     std::string str;  
   };
   struct IconLabel {
     std::string str;
     std::string icon_src;
   };
   
   struct HtmlLabelBuilder {
     [[nodiscard]] std::string operator()(const SimpleLabel& label) {
       return "<p>" + label.str + "</p>";
     }
     [[nodiscard]] std::string operator()(const DateLabel& label) {
       return "<p class=\"date\">Date: " + label.str + "</p>";
     }
     [[nodiscard]] std::string operator()(const IconLabel& label) {
      	return "<p><img src=\"" + label.icon_str + "\"/>" + labe.str + "</p>";  
     }
   }
   
   using LabelVariant = std::variant<SimpleLabel, DateLabel, IconLabel>;
   int main() {
     std::vector<LabelVariant> vecLabels;
     vecLabels.emplace_back(SimpleLabel {"hello world"});
     vecLabels.emplace_back(DateLabel {"10th China 2020"});
     vecLabels.emplace_back(IconLabel{"Error", "error.png"});
     
     std::string final_html;
     for (auto& label : vecLabels) {
       final_html += std::visit(HtmlLabelBuilder{}, label) + "\n";
     }
   }
   ```

   另一种实现:

   ```c++
   struct SimpleLabel {
     [[nodiscard]] std::string build_html() const {
        return "<p>" + label.str + "</p>";
     }
   };
   struct DateLabel {
     [[nodiscard]] std::string build_html() const {
       return "<p class=\"date\">Date: " + label.str + "</p>";
     }
   };
   struct IconLabel {
     [[nodiscard]] std::string build_html() const {
      	return "<p><img src=\"" + label.icon_str + "\"/>" + labe.str + "</p>";
     }
   };
   auto html_builder = [](auto& label) { return label.build_html(); }
   for (auto& label : vecLabels) {
     final_html += std::visit(html_builder, label) + "\n";
   }
   ```

3. 为lambda增加限制(concept(cpp20))以支持同一的函数签名

   ```c++
   template <typename T>
   concept ILabel = requires(const T v) {
       {v.build_html()} -> std::convertible_to(std::string);
   }
   auto html_builder = [ILabel auto& label] -> std::string {
     return label.build_html();
   }
   for (auto& label : vecLabels) {
     final_html += std::visit(html_builder, label) + "\n";
   }
   ```

   

   