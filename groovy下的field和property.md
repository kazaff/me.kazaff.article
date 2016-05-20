今天要聊的这个话题，是基于一个很有意思的jvm语言：groovy！有好奇心的童靴不妨去了解一下它的风骚~

在java bean的定义中，一般都会为这个bean设置一些属性，然后按照javabean规范会创建getter和setter方法~（你是否也早已经烦透了这种样板代码？）

groovy的哲学，就是在表达清晰的前提下让你尽可能少和样板代码打交道！而我们要讨论的，就是与java bean看起来极其相似的groovy bean！

本文一下内容，翻译自[groovy bean](http://groovy.jmiguel.eu/groovy.codehaus.org/Groovy+Beans.html)

---

groovy bean 就是 java bean，但是提供了大量的简化语法！

```groovy
	class Customer {
		// properties
		Integer id
		String name
		Date dob

		// sample code
		static void main(args) {
				def customer = new Customer(id:1, name:"Gromit", dob:new Date())
				println("Hello ${customer.name}")
		}
	}
```
执行上面的代码后会看到输出

	Hello Gromit

可以看到property似乎就好像是公开的类field。你甚至可以在构造函数里为这些property初始化。在groovy里，field和property的概念被合并了，它们不仅看起来一样，使用起来也一样。因此，上面的groovy代码和下面的java代码是等价的：

```java
	import java.util.Date;

	public class Customer {
	    // properties
	    private Integer id;
	    private String name;
	    private Date dob;

	    public Integer getId() {
	        return this.id;
	    }

	    public String getName() {
	        return this.name;
	    }

	    public Date getDob() {
	        return this.dob;
	    }

	    public void setId(Integer id) {
	        this.id = id;
	    }

	    public void setName(String name) {
	        this.name = name;
	    }

	    public void setDob(Date dob) {
	        this.dob = dob;
	    }

	    // sample code
	    public static void main(String[] args) {
	        Customer customer = new Customer();
	        customer.setId(1);
	        customer.setName("Gromit");
	        customer.setDob(new Date());

	        System.out.println("Hello " + customer.getName());
	    }
	}
```

我们来看看property和field的一些规则~

当groovy被编译成字节码文件后，将遵守下面的规则：

- 如果声明时为name明确指定了一个操作限定符（public，private或protected）则将会创建一个field
- 若没有指定任何操作限定符，则该name将被创建为private field并且自动创建getter和setter方法（就是一个property）
- 如果property被定义为final，则创建的private field同样是final，且不会自动创建setter方法
- 你可以声明一个property，并且自己实现getter和setter方法（译者注：多半是在学习这个知识点的时候才会这么做吧~）
- 你可以同时定义相同name的property和field，此时property会默认使用同名的field（译者注：这是要疯了啊）
- 如果你想要一个private或protected的property，你就必须自己提供满足你操作限定需求的getter和setter方法
- 如果你在class内部操作property（例如`this.foo`或`foo`），groovy会直接操作对应的field，而不会调用对应的getter或setter方法
- 如果你试图操作一个不存在property，groovy会通过meta class来操作对应的property，这可能会在运行时失败（译者注：动态语言特性）

现在来看一个例子，创建一个包含只读property的类：

```groovy
	class Foo {
		// read only property
		//译者注：使用final，groovy不会为其创建setter方法
		final String name = "John"

		// read only property with public getter and protected setter
		Integer amount
		protected void setAmount(Integer amount) { this.amount = amount }

		// dynamically typed property
		def cheese
	}
```

注意：property声明是需要一个标识符，或者是明确的类型（如`String`），或者是无类型声明（`def`）。

为什么groovy不会为一个被定义为public的field自动创建getter和setter方法呢？（译者注：极好的问题！）如果我们在任何条件下都自动创建getter和setter方法，意味着groovy下你无法摆脱getter和setter，这在某些你就是不想要getter和setter方法的场景下就傻眼了！（译者注：此段为“随译”，就是随便翻译的意思~哈哈）

#### Closures and listeners

本段和主题关系不大，就不翻译了~
