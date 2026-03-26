---
title: "JavaSE 学习笔记"
date: 2026-02-26 10:30:00
tags:
  - java
  - javase
  - 学习笔记
categories:
  - Java
thumbnail: "/images/thumbnails/JavaSE.png"
---

---

## 一、我为什么整理这篇笔记

很多时候，Java 学习卡在两个阶段：

1. 知识点都见过，但写代码会忘；
2. 能写功能，但说不清为什么这样写。

这篇文章按“**语法基础 -> 函数/方法 -> 面向对象 -> 常用 API**”梳理，补充了常见坑点和面试表达，作为我的阶段性学习记录。

### 学习目标

- 能独立写出结构清晰、命名规范的 JavaSE 程序；
- 理解 OOP 核心思想，并能在项目里落地；
- 熟悉常用 Java API（集合、时间、异常、IO、字符串工具等）；
- 形成可持续复盘的学习方式。

---

## 二、语法基础（写代码的地基）

### 1) 数据类型与变量

Java 是强类型语言，变量必须先声明再使用。

```java
public class BasicTypesDemo {
    public static void main(String[] args) {
        byte b = 10;
        short s = 1000;
        int i = 100000;
        long l = 10000000000L; // long 字面量建议加 L

        float f = 3.14F; // float 字面量建议加 F
        double d = 3.1415926;

        char c = 'A';
        boolean flag = true;

        String name = "JavaSE";

        System.out.println(name + " -> " + i + ", " + d);
    }
}
```

**易错点：**

| 场景 | 正确写法 | 常见错误 |
| :-- | :-- | :-- |
| long 字面量 | `long x = 1L;` | `long x = 1;`（小值可过，但习惯不统一） |
| float 字面量 | `float y = 1.0F;` | `float y = 1.0;`（默认 double） |
| 字符串比较 | `a.equals(b)` | `a == b` |

### 2) 运算符和流程控制

核心就三块：

- 分支：`if-else`、`switch`
- 循环：`for`、`while`、`do-while`
- 跳转：`break`、`continue`、`return`

```java
public class ControlFlowDemo {
    public static void main(String[] args) {
        int score = 87;
        if (score >= 90) {
            System.out.println("A");
        } else if (score >= 80) {
            System.out.println("B");
        } else {
            System.out.println("C");
        }

        int sum = 0;
        for (int i = 1; i <= 100; i++) {
            if (i % 2 == 0) {
                sum += i;
            }
        }
        System.out.println("1~100 偶数和: " + sum);
    }
}
```

### 3) 数组基础

```java
import java.util.Arrays;

public class ArrayDemo {
    public static void main(String[] args) {
        int[] nums = {3, 1, 5, 2, 4};
        Arrays.sort(nums);
        System.out.println(Arrays.toString(nums));
    }
}
```

**我的理解：**数组适合定长、追求读取性能的场景；当需要频繁增删时，优先考虑集合（如 `ArrayList`、`LinkedList`）。

---

## 三、函数/方法（把重复逻辑收敛起来）

Java 里通常称为“方法（method）”。

### 1) 方法定义与调用

```java
public class MethodDemo {
    public static int add(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        int result = add(3, 5);
        System.out.println(result);
    }
}
```

### 2) 方法重载（Overload）

同名方法，参数列表不同。

```java
public class OverloadDemo {
    public static int sum(int a, int b) {
        return a + b;
    }

    public static double sum(double a, double b) {
        return a + b;
    }

    public static int sum(int a, int b, int c) {
        return a + b + c;
    }
}
```

### 3) 可变参数与实用写法

```java
public class VarargsDemo {
    public static int total(int... nums) {
        int sum = 0;
        for (int n : nums) {
            sum += n;
        }
        return sum;
    }

    public static void main(String[] args) {
        System.out.println(total(1, 2, 3, 4));
    }
}
```

### 4) 方法设计规范（我在练习中执行的规则）

- 单一职责：一个方法只做一件事；
- 方法名动词化：如 `calculatePrice`、`validateInput`；
- 参数不宜过多（> 4 需考虑对象封装）；
- 尽量避免“超长方法”（可维护性差）。

---

## 四、面向对象（OOP）

面向对象是 Java 的核心。我的理解是：

- **封装**：把数据和行为放在一起，并控制访问边界；
- **继承**：复用父类能力，表达 `is-a` 关系；
- **多态**：同一个父类型引用，指向不同子类对象，运行时表现不同。

### 1) 封装

```java
public class Student {
    private String name;
    private int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("年龄不合法");
        }
        this.age = age;
    }
}
```

### 2) 继承与方法重写

```java
class Animal {
    public void speak() {
        System.out.println("动物发声");
    }
}

class Dog extends Animal {
    @Override
    public void speak() {
        System.out.println("汪汪汪");
    }
}
```

### 3) 多态

```java
public class PolymorphismDemo {
    public static void makeSpeak(Animal animal) {
        animal.speak();
    }

    public static void main(String[] args) {
        Animal a1 = new Animal();
        Animal a2 = new Dog();

        makeSpeak(a1); // 动物发声
        makeSpeak(a2); // 汪汪汪
    }
}
```

### 4) 抽象类 vs 接口

| 对比项 | 抽象类 | 接口 |
| :-- | :-- | :-- |
| 关系定位 | 模板、父类能力沉淀 | 行为规范、能力扩展 |
| 成员 | 可有普通方法/成员变量 | 主要是常量 + 抽象方法（JDK8+ 支持 default/static） |
| 继承方式 | 单继承 | 可多实现 |

**实践建议：**

- 优先组合，其次继承；
- 面向接口编程，降低耦合；
- 公共能力可抽到工具类或抽象父类，但避免过度抽象。

---

## 五、Java API 常用模块速记

### 1) String / StringBuilder

```java
public class StringDemo {
    public static void main(String[] args) {
        String s = "java";
        String upper = s.toUpperCase();
        System.out.println(upper); // JAVA

        StringBuilder sb = new StringBuilder();
        sb.append("Hello").append(" ").append("Java");
        System.out.println(sb.toString());
    }
}
```

- `String` 不可变，适合少量拼接；
- 循环拼接推荐 `StringBuilder`；
- 涉及线程安全考虑 `StringBuffer`（较少用）。

### 2) 集合框架（Collection）

```java
import java.util.*;

public class CollectionDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("Java");
        list.add("SE");

        Set<Integer> set = new HashSet<>();
        set.add(1);
        set.add(1); // 去重

        Map<String, Integer> map = new HashMap<>();
        map.put("Alice", 90);
        map.put("Bob", 85);

        System.out.println(list);
        System.out.println(set);
        System.out.println(map.get("Alice"));
    }
}
```

**集合选型速查：**

| 需求 | 推荐 |
| :-- | :-- |
| 按插入顺序遍历 + 快速随机访问 | `ArrayList` |
| 频繁头尾插入删除 | `LinkedList` |
| 去重、无序 | `HashSet` |
| 键值对存储 | `HashMap` |
| 线程安全 Map（高并发） | `ConcurrentHashMap` |

### 3) 时间 API（java.time）

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class TimeDemo {
    public static void main(String[] args) {
        LocalDate today = LocalDate.now();
        LocalDateTime now = LocalDateTime.now();

        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        System.out.println(now.format(fmt));
        System.out.println(today.plusDays(7));
    }
}
```

### 4) 异常处理

```java
public class ExceptionDemo {
    public static void main(String[] args) {
        try {
            int x = 10 / 0;
            System.out.println(x);
        } catch (ArithmeticException e) {
            System.out.println("发生算术异常: " + e.getMessage());
        } finally {
            System.out.println("finally 一般用于资源收尾");
        }
    }
}
```

**我的异常处理习惯：**

- 不吞异常，至少记录上下文；
- 业务层做可读错误提示，底层保留异常链；
- 预期可恢复异常优先处理，不可恢复异常尽早失败。

### 5) IO 基础

```java
import java.io.*;

public class IoDemo {
    public static void main(String[] args) {
        String file = "demo.txt";

        try (BufferedWriter bw = new BufferedWriter(new FileWriter(file))) {
            bw.write("Hello Java IO");
        } catch (IOException e) {
            e.printStackTrace();
        }

        try (BufferedReader br = new BufferedReader(new FileReader(file))) {
            System.out.println(br.readLine());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

---

## 八、后续计划

1. 继续补全 JavaSE 高级部分：反射、泛型进阶、注解、并发；
2. 用 JavaSE + MySQL 写一个完整的小项目（含日志、异常、分层）；
3. 逐步过渡到 Spring/SpringBoot，建立后端工程化能力。

