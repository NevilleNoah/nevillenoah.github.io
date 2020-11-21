# Java常用命令




Java有四个访问控制符用于控制变量、方法、类被允许访问的范围：

* default
* public
* protected
* private



你真的彻底理解他们的使用方式吗？

我们经常会看到这样一张表，但你真的理解这张表的实际内涵吗？

| 修饰符      | 当前类 | 同一包内 | 子孙类(同一包) | 子孙类(不同包) | 其他包 |
| :---------- | :----- | :------- | :------------- | :------------- | :----- |
| `public`    | Y      | Y        | Y              | Y              | Y      |
| `protected` | Y      | Y        | Y              | Y/N            | N      |
| `default`   | Y      | Y        | Y              | N              | N      |
| `private`   | Y      | N        | N              | N              | N      |

很多人凭借着死记硬背的本事来记这些东西，令人唏嘘不已，让大家在理解的基础上记忆是我写这篇文章的初衷。



## default&public&private

在解释之前，我们要理解什么是“跨包”、“同包”。



### 跨包的误区

首先，建立我们的目录：

```
- Outpack
	- Out.java
	- Pack
		- Target.java
		- Equal.java
		- Inpack
			- In.java
```

其中有三级目录，Outpack、Pack、Inpack。Pack是我们的主要的目录，包含`Target.java`、`Equal.java`；Outpack是Pack的父包，包含`Out.java`；Inpack是Pack的子包，包含`In.java`。

请问，在以上的所有java文件中，与`Target.java`同包（即不存在“跨包”关系）的文件是什么？

我猜你可能会有两种答案：

* `Equal.java`（正确）
* `Equal.java`和`In.java`（错误）

注意这是一个常见的误区，子包也算是“跨包”而不是“同包”。

**“同包”即在同一个目录下，“跨包”即先祖包、子孙包。**



### 三大基础修饰符

接下来来看我们的`Target.java`文件：

```java
public class Target {
  	public int publicVar;
    protected int protectedVar;
    private int privateVar;
    int defaultVar;
}
```

default：默认的访问范围是“同包”，即在同一个目录下能够访问，用default修饰（因为默认通常可忽略）。`Target.java`、`Equal.java`能够访问`defaultVar`。

public：允许跨包访问，“同包”+“跨包”，无论是同包、先祖包还是子孙包都能够访问，用public修饰。上述所有java文件都能访问`publicVar`。

private：不想被其他java文件“访问”，用private修饰。仅允许`Target.java`自身能访问`privateVar`。



## protected

protected的意义是为了应对继承中的访问关系，他其实是在default的基础上加上一条：允许子类访问父类中的protected。




