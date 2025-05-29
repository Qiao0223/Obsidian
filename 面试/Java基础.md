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

## 2.4. 反射的常见操作

### 1. 创建对象实例

通过反射，可以在运行时创建类的实例：
```
Class<?> clazz = Class.forName("com.example.MyClass");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

### 2. 访问和修改字段

可以通过反射访问和修改对象的属性，包括私有属性
```
Field field = clazz.getDeclaredField("fieldName");
field.setAccessible(true); // 允许访问私有字段
field.set(instance, value); // 设置字段值
Object value = field.get(instance); // 获取字段值
```

### 3. 调用方法

通过反射调用对象的方法
```
Method method = clazz.getDeclaredMethod("methodName", parameterTypes);
method.setAccessible(true); // 允许访问私有方法
Object result = method.invoke(instance, args);
```

### 4. 获取构造函数并创建实例

可以获取特定的构造函数，并使用它创建对象
```
Constructor<?> constructor = clazz.getDeclaredConstructor(parameterTypes);
constructor.setAccessible(true); // 允许访问私有构造函数
Object instance = constructor.newInstance(args);
```

## 2.5. 优缺点

**优点：**

- **灵活性高**：可以在运行时动态地操作对象，提高了程序的灵活性。
- **解耦合**：通过反射，可以减少代码之间的依赖，提高模块的独立性。
- **框架支持**：许多框架（如 Spring、Hibernate）都依赖于反射机制来实现其核心功能。

**缺点：**

- **性能开销**：反射操作通常比直接代码调用慢，可能影响性能。
- **安全性问题**：反射可以访问私有成员，可能破坏封装性，带来安全隐患。
- **复杂性增加**：使用反射的代码通常更复杂，难以阅读和维护。

# 3. 泛型

Java 泛型（Generics）是 Java SE 5 引入的一项重要特性，允许在类、接口和方法中使用类型参数，从而实现代码的类型安全和重用性。

## 3.1. 概念

泛型的本质是“参数化类型”，即在定义类、接口或方法时，将类型参数化，使其在使用时指定具体的类型。

常见的类型参数命名约定包括：

- `T`：Type（类型）
- `E`：Element（元素）
- `K`：Key（键）
- `V`：Value（值）
- `N`：Number（数字）

## 3.2. 主要形式

### 1. 泛型类

定义泛型类时，在类名后使用尖括号 `<>` 指定类型参数。例如
```
public class Box<T> {
    private T t;
    public void set(T t) { this.t = t; }
    public T get() { return t; }
}
```
使用时指定具体类型：
```
Box<Integer> integerBox = new Box<>();
integerBox.set(10);
Integer value = integerBox.get();
```

### 2. 泛型接口

定义泛型接口时，也在接口名后使用尖括号指定类型参数。例如
```
public interface Pair<K, V> {
    K getKey();
    V getValue();
}
```
实现泛型接口时，可以指定具体类型或继续使用泛型：
```
public class OrderedPair<K, V> implements Pair<K, V> {
    private K key;
    private V value;
    public OrderedPair(K key, V value) {
        this.key = key;
        this.value = value;
    }
    public K getKey() { return key; }
    public V getValue() { return value; }
}
```

### 3. 泛型方法

泛型方法在返回类型前声明类型参数。例如：
```
public class Util {
    public static <T> void printArray(T[] array) {
        for (T element : array) {
            System.out.println(element);
        }
    }
}
```
调用泛型方法时，编译器会根据传入参数的类型推断类型参数：
```
Integer[] intArray = {1, 2, 3};
Util.printArray(intArray);
```

## 3.3. 通配符与边界

Java 泛型支持通配符 `?`，用于表示未知类型。通配符可以指定上界或下界：

- `? extends T`：表示类型是 T 或其子类，适用于读取数据的场景。
- `? super T`：表示类型是 T 或其父类，适用于写入数据的场景。

```
List<? extends Number> list = new ArrayList<Integer>();
```
在这个例子中，`list` 可以引用任何 `Number` 的子类的列表，如 `Integer`、`Double` 等。

## 3.4. 类型擦除与限制

Java 的泛型在编译时会进行类型擦除，即将类型参数替换为其限定类型（如未指定则为 `Object`），因此在运行时无法获取泛型的具体类型信息。

这带来了一些限制：

- 不能使用基本类型作为类型参数，如 `List<int>` 是非法的，需使用包装类型 `Integer`。
- 不能创建泛型类型的数组，如 `new T[]` 是非法的。
- 不能在静态上下文中引用泛型类型参数。

这些限制是由于类型擦除机制导致的。

## 3.5. 优势

- **类型安全**：在编译时检查类型，减少运行时错误。
- **代码重用**：编写一次代码，可适用于多种类型。
- **可读性和可维护性**：代码更清晰，易于理解和维护。

