title: IntelliJ IDEA 使用 FindBugs 进行代码分析
author: ghthou
tags:
  - Java
categories:
  - Java
date: 2018-01-13 13:17:00
---
### [FindBugs](http://findbugs.sourceforge.net/) 介绍
FindBugs 是一个使用静态分析来 ** 查找 Java 代码中的错误 ** 的程序。它是免费软件
当前版本的 FindBugs 是 3.0.1
**FindBugs 运行需要 1.7 或更高版本的 JRE（或 JDK）**。但是，它可以分析从任何版本的 Java 编译的程序，从 1.0 到 1.8

以上是来自官网的介绍，核心内容为查找 Java 代码中的错误

### IntelliJ IDEA 安装 FindBugs 插件
- `Ctrl+Alt+S` 快捷键打开设置选项
- 选择 `Plugins`
- 点击底部 `Browse repositories` 按钮打开插件中心
- 在输入框搜索 `findbugs`
- 选择 `FindBugs-IDEA`
- 此时右边红框位置会有一个绿色的 `Install` 按钮 (下面这张图是已经安装的情况), 点击安装即可, 安装完成后需要重启 IDEA
![IDEA 安装 FindBugs 插件](/images/IntelliJ-IDEA-使用-FindBugs-进行代码分析/IDEA安装FindBugs插件.png)

### [FindBugs-IDEA](http://andrepdo.github.io/findbugs-idea/) 插件使用介绍
- `Ctrl+Shift+A` 快捷键打开 Find Action 搜索面板
- 在搜索框输入 `findbugs` 进行搜索
- 点击下方红框的 `FindBugs-IDEA` 即可打开插件面板
![打开 FindBugs 插件](/images/IntelliJ-IDEA-使用-FindBugs-进行代码分析/打开FindBugs插件.png)
插件面板按钮说明
![FindBugs 面板](/images/IntelliJ-IDEA-使用-FindBugs-进行代码分析/FindBugs面板.png)
1. 分析选中的 Java 文件
2. 分析在光标所在的类
3. 分析选中的包
4. 分析选中的模块 (点击时会询问是否同时分析 test 包中的类)
5. 分析整个项目 (点击时会询问是否同时分析 test 包中的类)
6. 自定义分析的类
7. 分析被修改的类 (搭配 SVN,Git 使用)
8. 分析 changelist 中的类 (搭配 SVN,Git 使用)
9. 停止分析
10. 根据 BUG 类型分组
11. 根据类分组
12. 根据包分组
13. 根据 BUG 严重级别分组

### FindBugs-IDEA 插件使用示例
创建一个带有一些问题的类
```java
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Map;
import java.util.Random;

public class FindBugsDemo {

	private static final DateFormat yyyyMMdd = new SimpleDateFormat("yyyy-MM-dd");

	public static String yyyyMMddForMat(Date date) {
		return yyyyMMdd.format(date);
	}

	public static int getRanDom() {
		return new Random().nextInt();
	}

	public static int round(int num) {
		return Math.round(num);
	}

	public static void printMap(Map<?, ?> map) {
		if (map != null && map.size() > 0) {
			for (Object key : map.keySet()) {
				System.out.println("key--->" + key);
				System.out.println("value--->" + map.get(key));
			}
		}
	}

	public static String trimString(String str) {
		str.trim();
		return str;
	}

	@Override
	public boolean equals(Object obj) {
		return super.equals(obj);
	}

}
```
选中该类, 点击上面插件面板序号为 1 对应的按钮, 进行文件分析, 结果如下
![测试代码分析结果](/images/IntelliJ-IDEA-使用-FindBugs-进行代码分析/测试代码分析结果.png)
我们是使用 BUG 严重级别进行分组
分组类型对应如下 (红色编号)
1. Of Concren 建议, 如果遵循能更好的完善代码
2. Troubling 不好的, 可能会引发不良后果
3. Scary **严重问题**, 在某种情况下一定会出现问题
4. Scariest **非常严重**, 已经影响到当前程序功能
可以按照严重级别倒序进行修复, 如果时间允许, 可以将 `Of Concren` 中的问题也一并修复

下面对具体提示的 BUG 进行分析 (黄色编号)
1.Random object created and used only once (Random 对象创建后只使用一次)
> 该方法每次运行都会创建一个新的 Random 对象, 执行一次后就会被回收. 但是在多线程情况获取随机数方法也能正常使用, 所以可以定义一个 Random 对象常量, 然后使用该常量对象进行方法调用. 能减少创建对象的性能开销

2.Class defines equals() and uses Object.hashCode() (覆写了 equals 方法但是没有覆写 hashCode 方法)
> 在 Set,Map 中会使用对象的 hashCode 方法, 如果覆写了 equals 方法但是没有覆写 hashCode 方法会导致在 Set,Map 对象中出现问题

3.Inefficient use of keySet iterator instead of entrySet iterator (keySet 迭代器低效, 应该使用 entrySet 进行替换)
> 如果需要获取 Map 中的 key 和 value, 使用 Map.entrySet() 方法返回 Set<Map.Entry<K, V>> 对象, 然后迭代该 Set, 在使用 Entry 对象获取 key 和 value 更为高效

4.Method ignores return value (方法忽略返回值)
> String 对象是不可变的, 当调用 String.trim() 后, 是返回一个新的 String 对象, 不会对调用者的内容进行改动

5.int value cast to float and then passed to Math.round (将 int 值转换为 float，然后传递给 Math.round)
> Math.round() 方法只接收 float 和 double 类型, 然后转换为 int 和 long 类型, 如果传递 int 类型, 会先将其转换为 float 类型, 然后再转换为 int 类型, 所以导致该操作返回值与参数内容一致

6.Call to static DateFormat (调用静态的 DateFormat 对象)
> DateFormat 对象是线程不安全的, 如果多线程调用同一个 DateFormat 对象会导致结果异常

FindBugs 只是一款静态代码分析工具, 虽然分析大多数的问题, 但是如果希望编写更为健壮的程序, 还需进行更多的测试操作, 切不可认为 FindBugs 没有分析出问题便认为没有问题了
