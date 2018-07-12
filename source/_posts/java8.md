---
title: Java8的几个重要特性介绍
---

`Java8`的新特性有很多，只对代表性的四个重要特性做一次总结，方便以后回顾。

`Java8`的四个重要新特性:
+ Lambda；
+ 方法引用；
+ 默认方法；
+ Stream。
<!--more-->
# 1. Lambda

`Lambda`表达式：Lambda可以让函数当做一个方法的参数进行传递，并且让代码变得更加简洁。`Lambda`表达式省去了匿名类的麻烦，但Lambda只能给包含一个方法的接口定义，且Lambda的入参和返回值必须和接口中的方法一致。
语法：
(type param1, type param2…) -> {Statements}

Lambda几个特性：
+ 参数类型可选：编译器可以自动识别参数的类型；
+ 参数圆括号可选：只有一个参数时，可省略圆括号；
+ 语句块大括号可选：只有一个语句时，可省略大括号；
+ 返回值可选：只有一个语句时，可省略返回值。

这四个特性，两个针对参数，两个对应语句块，很好理解，代码分析如下：
```java
public class LambdaTest3 {
    public static void main(String[] args) {
		
		/**
		 * 类型声明，圆括号，大括号的省略情况
		 */
		
		// 先看参数部分，主要看类型声明和括号的省略情况
		// 1.参数类型声明可选
		// 2.一个参数时，括号可选
		
		// 两个参数，没有省略时
		MathOperate1 mathOperate1_1 = (int a, int b) -> {
			System.out.println("two parameter");
			return a + b;
		};
		mathOperate1_1.operate(12, 45);

		// 两个参数，省略参数类型声明
		MathOperate1 mathOperate1_2 = (a, b) -> {
			System.out.println("two parameter");
			return a + b;
		};
		mathOperate1_2.operate(12, 45);

		// 一个参数，带类型声明，必须有括号
		MathOperate2 mathOperate2_1 = (int a) -> {
			System.out.println("one parameter");
			return a;
		};
		mathOperate2_1.operate(8);

		// 一个参数，省略类型声明
		MathOperate2 mathOperate2_2 = (a) -> {
			System.out.println("one parameter");
			return a;
		};
		mathOperate2_2.operate(8);

		// 一个参数，省略括号，必须同时省略类型声明
		MathOperate2 mathOperate2_3 = a -> {
			System.out.println("one parameter");
			return a;
		};
		mathOperate2_3.operate(8);

		// 无参数，不可省略括号
		MathOperate3 mathOperate3_1 = () -> {
			System.out.println("no parameter");
			return -1;
		};
		mathOperate3_1.operate();

		// 语句块部分
		// 1.一条语句，可省略大括号
		// 2.一条语句，未省略大括号，有返回值，则必须要return关键字
		
		// 多条语句，有返回值，不能省略大括号，不能省略return，若无返回值则不需要return
		MathOperate1 operate1 = (a, b) -> {
			int c;
			return c = a + b;
		};
		operate1.operate(8, 8);

		// 一条语句，有返回值，有大括号，则必须有return
		MathOperate1 operate2 = (a, b) -> {
			return a + b;
		};
		operate2.operate(8, 8);
		
		// 一条语句，没有返回值，有大括号时，不需要return
		MathOperate4 operate3 = (a, b) -> {
			int c = a + b;
		};
		operate3.operate(8, 8);
		
		// 一条语句，省略大括号和return关键字，如果有返回值，表达式结果默认为返回值
		MathOperate1 operate4 = (a, b) -> a + b;
		operate4.operate(8, 8);

	}

	// 两个参数
	@FunctionalInterface
	interface MathOperate1 {
		int operate(int a, int b);
	}

	// 一个参数
	@FunctionalInterface
	interface MathOperate2 {
		int operate(int a);
	}

	// 无参数
	@FunctionalInterface
	interface MathOperate3 {
		int operate();
	}

	// 无返回值
	@FunctionalInterface
	interface MathOperate4 {
		void operate(int a, int b);
	}

}
```

另外，Lambda主体部分可能使用外部的变量，当Lambda要使用外部的变量时，其内部不能修改外部变量的值。
```java
public class LambdaTest2 {

	public static void main(String[] args) {
		
		/**
		 * 作用域问题
		 */
		
		// 引用一个块外部的变量，在内部不能修改它的值
		int c = 12; 
		test2(56, 45, (a, b) -> {
			a += 2;
			b += 2;
			int d = a + b + c;
			System.out.println(d);
		});
	}

	private static void test2(int a, int b, LambdaTestInterface2 lam2) {
		lam2.init(a, b);
	}

	public interface LambdaTestInterface2 {
		void init(int a, int b);
	}
}
```

如何理解java 8中引入的Lambda表达式？匿名类，从这里着手看看，在此之前，接口对象，可用匿名类实例化。而Lambda可以理解为接口的另一种实例化的方法，但前提是这个接口只能有一个抽象方法。
```java
public class LambdaTest {

	public static void main(String[] args) {
		// TODO Auto-generated method stub

		/**
		 * lambda和匿名类的对比
		 */

		// 匿名类的写法，当参数使用
		int result1 = operation(8, 8, new MathOperate() {
			@Override
			public int operate(int a, int b) {
				return a + b;
			}
		});

		// lambda，同样当参数使用
		int result2 = operation(8, 8, (a, b) -> a + b);

		System.out.println(result1);
		System.out.println(result2);
	}

	private static int operation(int a, int b, MathOperate mathOperate) {
		return mathOperate.operate(a, b);
	}

	@FunctionalInterface
	interface MathOperate {
		int operate(int a, int b);
	}
}
```

# 2. 方法引用

使用“::”进行方法引用，按引用的方法分三类：
+ 静态方法；
+ 构造方法；
+ 成员方法。

引用静态方法，类名::方法名。方法参数和返回值需要和接口中的一致；
引用构造函数，类名::new。构造函数需要和接口中参数一致；
引用对象方法，对象.方法名。方法参数和返回值需要和接口中的一致，不能通过对象引用静态方法。
```java
public class MethodJava8 {
	public static void main(String[] args) {

		/**
		 * 方法引用
		 */
		
		// ::可以引用静态方法，对象的方法或构造函数
		// 1.静态方法引用，引用方法的形参和返回值，需和接口中的一致
		Convert<String, Integer> convert = Integer::valueOf;
		System.out.println(convert.convert("888"));
		// 匿名类的方法实现
		Convert<String, Integer> convert2 = new Convert<String, Integer>() {

			@Override
			public Integer convert(String t1) {
				// TODO Auto-generated method stub
				return Integer.valueOf(t1);
			}

		};
		System.out.println(convert2.convert("999"));

		// 引用自定义类的静态方法
		Convert<String, Integer> convert3 = MyConvert::convert;
		System.out.println(convert3.convert("101010"));

		// 2.引用对象中的方法，静态方法不能通过对象引用
		MyConvert myConvert = new MethodJava8.MyConvert();
		//引用对象的成员方法
		Convert<String, Integer> convert4 = myConvert::convert2;
		// Convert<String, Integer> convert4 = myConvert::convert; //错误，不能引用静态方法
		System.out.println(convert4.convert("111111"));

		// 3.引用User的构造函数
		UserFactory<User> userfactory = User::new;
		User user = userfactory.create("Test2", 22, "male");
		System.out.println(user.getName());

		// 匿名类实现UserFactory
		UserFactory<User> userfactory2 = new UserFactory<User>() {
			@Override
			public User create(String name, int age, String gendle) {
				return new User(name, age, gendle);
			}
		};
		System.out.println(userfactory2.create("Test1", 11, "female").getName());
	}

	@FunctionalInterface
	interface Convert<T1, T2> {
		T2 convert(T1 t1);
	}

	interface UserFactory<U extends User> {
		U create(String name, int age, String gendle);
	}
	
	public static class MyConvert {

		public static Integer convert(String t1) {
			// TODO Auto-generated method stub
			return Integer.valueOf(t1);
		}
		
		public Integer convert2(String t1){

			return Integer.valueOf(t1);
		}
	}
}
```

# 3. 默认方法

在接口的方法前加default关键字，表示一个默认方法，加static表示默认静态方法。
有了默认方法，在需求变更的时候，不必因增加接口中的方法，而需要重新实现已经实现该接口的类。

```java
public class DefaultMethod {

	public static void main(String[] args) {
		// TODO Auto-generated method stub

		/**
		 * 默认方法
		 */
		
		MyDefaultMethodTest.operate();
		
		MyDefaultMethodTest myDefaultMethodTest = new MyDefaultMethodTest() {
		};
		
		myDefaultMethodTest.init();
	}

	public interface MyDefaultMethodTest {
		// 默认方法
		default void init() {
			System.out.println("It is init() in MyDefaultMethodTest");

		}
		
		//默认静态方法，不能通过对象访问
		static void operate() {
			System.out.println("It is operate() in MyDefaultMethodTest");
		}
	}

}
```

# 4. Stream

Stream，类似把操作的对象看成一个数据流，对它操作后会返回被处理后的流，可以继续操作，比如排序，过滤，变换等等。
+ 数据流的生成
集合，数组，IO，产生器等。
+ 数据流操作
foreach，map，filter，limit，sorted，parallel等。
```java
public class StreamJava8 {

	public static void main(String[] args) {
		// TODO Auto-generated method stub

		/**
		 * 创建Stream流
		 */
		List<String> list = Arrays.asList("a", "b", "c", "d", "e", "f");
		List<String> filtered1 = list.stream()
				.filter(string -> !string.isEmpty())
				.collect(Collectors.toList());
		/**
		 * 聚合操作
		 */
		// foreach
		filtered1.forEach(s -> System.out.println(s));
		System.out.print("\n");

		// map
		List<Integer> mapNumbers = Arrays.asList(11, 2, 2, 6, 4, 3, 9);
		List<Integer> mapResult = mapNumbers.stream().map(a -> a * a)
				.distinct().collect(Collectors.toList());
		mapResult.forEach(System.out::println);
		System.out.print("\n");

		// filter
		List<Integer> filterNumbers = Arrays.asList(11, 2, 2, 6, 4, 3, 9);
		List<Integer> filterResult = filterNumbers.stream().filter(a -> a > 5)
				.collect(Collectors.toList());
		filterResult.forEach(System.out::println);
		System.out.print("\n");

		// limit
		List<String> limitStr = Arrays.asList("a", "b", "c", "d", "e", "f");
		List<String> limitResult = limitStr.stream().limit(2)
				.collect(Collectors.toList());
		limitResult.forEach(System.out::println);
		System.out.print("\n");

		// sorted
		List<Integer> sortedNumbers = Arrays.asList(11, 2, 2, 6, 4, 3, 9);
		List<Integer> sortedResult = sortedNumbers.stream()
				.sorted((a, b) -> a < b ? 1 : a == b ? 0 : -1)
				.collect(Collectors.toList());
		sortedResult.forEach(System.out::println);
		System.out.print("\n");

		// parallel
		List<String> parallelNumbers = Arrays.asList("a", "d", "a", "b", "c",
				"d", "e", "f");
		List<String> parallelResult = parallelNumbers.parallelStream()
				.filter(str -> str.equals("a")).collect(Collectors.toList());
		parallelResult.forEach(System.out::println);
		System.out.print("\n");

		// Collectors
		List<String> collectorsStr = Arrays.asList("a", "d", "a", "b", "c",
				"d", "e", "f");
		// 返回List
		List<String> collectorResultList = collectorsStr.stream()
				.filter(str -> !str.isEmpty()).collect(Collectors.toList());
		System.out.println("列表: " + collectorResultList);
		// 返回String
		String collectorResultString = collectorsStr.parallelStream()
				.filter(str -> !str.isEmpty()).collect(Collectors.joining("+"));
		System.out.println("合并: " + collectorResultString);
	}

}
```
能力有限，先写到这，部分要点来自网络，仅作学习之用。
