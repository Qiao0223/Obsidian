# 1. 集合

Java 集合框架（Java Collections Framework）是 Java 提供的一套用于存储和操作数据的标准化架构，主要包含接口、实现类和算法三部分。它支持多种数据结构，如列表（List）、集合（Set）、队列（Queue）和映射（Map），并通过泛型机制增强了类型安全性。

## 1.1. 集合框架的核心结构

Java 集合框架主要由两个顶层接口组成：

- **Collection 接口**：用于存储单个元素的集合，主要包括：
    - **List**：有序且可重复的元素集合。
    - **Set**：无序且不允许重复的元素集合。
    - **Queue**：支持先进先出（FIFO）操作的集合。
- **Map 接口**：用于存储键值对（key-value）的集合，每个键唯一，对应一个值。

## 1.2. 主要集合类型及其实现类

#### 1. List（有序、可重复）

- **ArrayList**：基于动态数组实现，支持快速随机访问，适合频繁读取操作。
- **LinkedList**：基于双向链表实现，适合频繁插入和删除操作。
- **Vector**：线程安全的动态数组，性能相对较低，现已较少使用。
- **Stack**：继承自 Vector，实现了后进先出（LIFO）的栈结构。

#### 2. Set（无序、不可重复）

- **HashSet**：基于哈希表实现，插入和查找效率高。
- **LinkedHashSet**：继承自 HashSet，维护元素的插入顺序。
- **TreeSet**：基于红黑树实现，元素自动排序，适合需要排序的集合。

#### 3. Queue（队列）

- **PriorityQueue**：基于堆实现的优先队列，元素按优先级排序。
- **ArrayDeque**：基于数组实现的双端队列，可用作栈或队列。

#### 4. Map（键值对映射）

- **HashMap**：基于哈希表实现，键值对无序，允许一个 null 键和多个 null 值。
- **LinkedHashMap**：继承自 HashMap，维护键值对的插入顺序。
- **TreeMap**：基于红黑树实现，键值对按键自动排序。
- **Hashtable**：线程安全的哈希表，已被 ConcurrentHashMap 替代。

# 2. 反射

Java 反射机制（Reflection）是 Java 语言的一项强大特性，允许程序在运行时动态地检查和操作类的结构，包括类的属性、方法和构造函数等。这使得 Java 程序具有更高的灵活性和扩展性，广泛应用于框架设计、插件系统、序列化、测试等领域。

## 2.1. 概念

Java 反射机制主要依赖于以下几个核心类，这些类位于 `java.lang.reflect` 包中：

- **`Class`**：表示类的字节码对象，是反射的起点。
- **`Field`**：表示类的成员变量。
- **`Method`**：表示类的方法。
- **`Constructor`**：表示类的构造函数。

通过这些类，程序可以在运行时获取类的详细信息，并进行相应的操作。

## 2.2. 核心类

Java 反射机制主要依赖于以下几个核心类，这些类位于 `java.lang.reflect` 包中：

- **`Class`**：表示类的字节码对象，是反射的起点。
- **`Field`**：表示类的成员变量。
- **`Method`**：表示类的方法。
- **`Constructor`**：表示类的构造函数。

通过这些类，程序可以在运行时获取类的详细信息，并进行相应的操作。

## 2.3. 获取 Class 对象的方式

在使用反射之前，首先需要获取类的 `Class` 对象。Java 提供了以下几种方式来获取

1. **通过类的 `.class` 属性**：

    `Class<?> clazz = MyClass.class;`

2. **通过对象的 `getClass()` 方法**：

    `MyClass obj = new MyClass(); Class<?> clazz = obj.getClass();`

3. **通过 `Class.forName()` 方法**：
    
    `Class<?> clazz = Class.forName("com.example.MyClass");`

获取到 `Class` 对象后，就可以使用反射机制来操作类的成员了。
