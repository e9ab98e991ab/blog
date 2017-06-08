# Java泛型介绍

泛型定义：参数化类型，数据类型被指定为一个参数

##定义泛型

1.定义在类声明上，作用域为整个类
public class ArrayList<T>
2.定义在方法上(静态方法也类似)
public <T> void init(T t) 

##实例化泛型
ArrayList<String> list=new ArrayList<>();

init("abc");

在实例化类或调用方法时要指定具体类。

##通配符

在实例化类时，不确定具体类型可以使用通配符‘?’

ArrayList<?> list=new ArrayList<>();
ArrayList<? extends Number> list=new ArrayList<>();
ArrayList<? super Integer> list=new ArrayList<>();

当指定类型为'?'时，除null外无法添加其他类型对象，因为无法确定类型是否匹配。extends表示是Number的子类。super表示是Integer的父类。


