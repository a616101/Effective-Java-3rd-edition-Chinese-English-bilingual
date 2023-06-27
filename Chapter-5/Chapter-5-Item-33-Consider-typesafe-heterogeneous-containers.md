## Chapter 5. Generics（泛型）

### Item 33: Consider typesafe heterogeneous containers（考虑类型安全的异构容器）

Common uses of generics include collections, such as `Set<E>` and `Map<K,V>`, and single-element containers, such as `ThreadLocal<T>` and `AtomicReference<T>`. In all of these uses, it is the container that is parameterized. This limits you to a fixed number of type parameters per container. Normally that is exactly what you want. A Set has a single type parameter, representing its element type; a Map has two, representing its key and value types; and so forth.

泛型的常見用途包括集合，例如 `Set<E>` 和 `Map<K,V>`，以及單元素容器，例如 `ThreadLocal<T>` 和 `AtomicReference<T>`。在所有這些用法中，都是參數化容器。這限制了每個容器的固定類型參數數量。通常這正是您想要的。Set 有一個單一類型參數，表示其元素類型；Map 有兩個類型參數，表示其鍵和值類型；等等。

Sometimes, however, you need more flexibility. For example, a database row can have arbitrarily many columns, and it would be nice to be able to access all of them in a typesafe manner. Luckily, there is an easy way to achieve this effect. The idea is to parameterize the key instead of the container. Then present the parameterized key to the container to insert or retrieve a value. The generic type system is used to guarantee that the type of the value agrees with its key.

然而，有時您需要更多的靈活性。例如，數據庫行可以具有任意多列，希望能夠以類型安全的方式訪問所有這些列。幸運的是，有一種簡單的方法可以實現此效果。其思想是將`鍵`參數化而不是容器。然後將參數化`鍵`呈現給容器以插入或檢索`值`。使用通用類型系統可保證`值`的類型與其`鍵`相符。


As a simple example of this approach, consider a Favorites class that allows its clients to store and retrieve a favorite instance of arbitrarily many types. The Class object for the type will play the part of the parameterized key. The reason this works is that class Class is generic. The type of a class literal is not simply Class, but `Class<T>`. For example, String.class is of type `Class<String>`, and Integer.class is of type `Class<Integer>`. When a class literal is passed among methods to communicate both compiletime and runtime type information, it is called a type token [Bracha04].

以一個簡單的例子來說明這種方法，考慮一個Favorites類別，它允許客戶端存儲和檢索任意多種類型的實例。該類型的Class對象將扮演參數化鍵的角色。這能夠運作是因為Class類是泛型的。 一個class literal 的類型不僅僅是 Class，而是 `Class<T>` 。例如，String.class 是 `Class<String>` 類型，Integer.class 是 `Class<Integer>` 類型。當在方法之間傳遞 class literal 去通信時，在編譯時和運行時都需要知道其所代表的具體类型信息，這被稱為 type token [Bracha04]。

The API for the Favorites class is simple. It looks just like a simple map, except that the key is parameterized instead of the map. The client presents a Class object when setting and getting favorites. Here is the API:

Favorites 類別的API非常簡單。它看起來就像一個簡單的映射，只是`鍵`被參數化而不是映射。客戶端在設置和獲取收藏時提供了一個Class物件。以下是API：

```
// Typesafe heterogeneous container pattern - API
public class Favorites {
    public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
}
```

Here is a sample program that exercises the Favorites class, storing, retrieving, and printing a favorite String, Integer, and Class instance:

這裡有一個範例程式，可以使用 Favorites 類別來存儲、檢索和打印喜愛的字串、整數和類實例：

```
// Typesafe heterogeneous container pattern - client
public static void main(String[] args) {
    Favorites f = new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 0xcafebabe);
    f.putFavorite(Class.class, Favorites.class);
    String favoriteString = f.getFavorite(String.class);
    int favoriteInteger = f.getFavorite(Integer.class);
    Class<?> favoriteClass = f.getFavorite(Class.class);
    System.out.printf("%s %x %s%n", favoriteString,favoriteInteger, favoriteClass.getName());
}
```

As you would expect, this program prints Java cafebabe Favorites. Note, incidentally, that Java’s printf method differs from C’s in that you should use %n where you’d use \n in C. The %n generates the applicable platform-specific line separator, which is \n on many but not all platforms.

正如您所期望的那樣，此程序會打印Java cafebabe Favorites。順便提一下，Java的printf方法與C的不同之處在於，在C中使用\n時，應該使用%n。%n生成適用於特定平台的換行符號，例如在許多但不是所有平台上都是\n。

**译注：`favoriteClass.getName()` 的打印结果与 Favorites 类所在包名有关，结果应为：包名.Favorites**

A Favorites instance is typesafe: it will never return an Integer when you ask it for a String. It is also heterogeneous: unlike an ordinary map, all the keys are of different types. Therefore, we call Favorites a typesafe heterogeneous container.

一個 Favorites 實例是類型安全的：當您要求它返回字符串時，它永遠不會返回整數。 它還是異構的：與普通映射不同，所有`鍵`都是不同類型的。 因此，我們稱Favorites為類型安全的異構容器。

The implementation of Favorites is surprisingly tiny. Here it is, in its entirety:

Favorites 的實現非常簡單。以下是全部內容：

```
// Typesafe heterogeneous container pattern - implementation
public class Favorites {
  private Map<Class<?>, Object> favorites = new HashMap<>();

  public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(Objects.requireNonNull(type), instance);
  }

  public <T> T getFavorite(Class<T> type) {
    return type.cast(favorites.get(type));
  }
}
```

There are a few subtle things going on here. Each Favorites instance is backed by a private `Map<Class<?>, Object>` called favorites. You might think that you couldn’t put anything into this Map because of the unbounded wildcard type, but the truth is quite the opposite. The thing to notice is that the wildcard type is nested: it’s not the type of the map that’s a wildcard type but the type of its key. This means that every key can have a different parameterized type: one can be `Class<String>`, the next `Class<Integer>`, and so on. That’s where the heterogeneity comes from.

這裡有一些微妙的事情正在發生。每個收藏夾實例都由一個名為favorites的私有`Map<Class <?>，Object>`支持。您可能會認為由於無邊界通配符類型，您無法將任何內容放入此映射中，但事實恰恰相反。要注意的是萬用字元類型是嵌套的：它不是地圖類型的萬用字元類型，而是其`鍵`值的類型。這意味著每個`鍵`可以具有不同的參數化類型：一個可以是`Class <String>`，下一個可以是`Class <Integer>`等等。這就是異質性來源所在。

The next thing to notice is that the value type of the favorites Map is simply Object. In other words, the Map does not guarantee the type relationship between keys and values, which is that every value is of the type represented by its key. In fact, Java’s type system is not powerful enough to express this. But we know that it’s true, and we take advantage of it when the time comes to retrieve a favorite.

接下來要注意的是，favorites Map 的`值`類型僅為 Object。換句話說，Map 不保證`鍵`和`值`之間的類型關係，即每個`值`都是由其`鍵`所表示的類型。事實上，Java 的類型系統不足以表達這一點。但我們知道這是真實存在的，在檢索收藏時利用了它。

The putFavorite implementation is trivial: it simply puts into favorites a mapping from the given Class object to the given favorite instance. As noted, this discards the “type linkage” between the key and the value; it loses the knowledge that the value is an instance of the key. But that’s OK, because the getFavorites method can and does reestablish this linkage.

putFavorite的實現非常簡單：它只是將給定Class對象到給定Favorites實例的映射放入Favorites中。如前所述，這會丟失`鍵`和`值`之間的「類型聯繫」，它會丟失`值`是`鍵`的實例這一知識。但沒關係，因為getFavorites方法可以重新建立此聯繫。

The implementation of getFavorite is trickier than that of putFavorite. First, it gets from the favorites map the value corresponding to the given Class object. This is the correct object reference to return, but it has the wrong compile-time type: it is Object (the value type of the favorites map) and we need to return a T. So, the getFavorite implementation dynamically casts the object reference to the type represented by the Class object, using Class’s cast method.

getFavorite的實現比putFavorite更棘手。首先，它從收藏夾映射中獲取與給定Class對象相對應的`值`。這是正確的物件引用以返回，但它具有錯誤的編譯時類型：它是Object（收藏夾映射的`值`類型），我們需要返回一個T。因此，getFavorite實現動態地將物件引用轉換為由Class對象表示的類型，使用Class 的cast方法。

The cast method is the dynamic analogue of Java’s cast operator. It simply checks that its argument is an instance of the type represented by the Class object. If so, it returns the argument; otherwise it throws a ClassCastException. We know that the cast invocation in getFavorite won’t throw ClassCastException, assuming the client code compiled cleanly. That is to say, we know that the values in the favorites map always match the types of their keys.

cast方法是Java的強制轉換運算符號的動態類比。它只是檢查其參數是否為Class對象所表示的類型的實例。如果是，則返回該引數；否則，它會拋出ClassCastException異常。我們知道，在getFavorite中進行強制轉換調用不會拋出ClassCastException異常，假設客戶端代碼已編譯成功。也就是說，我們知道favorites映射中的`值`始終與其`鍵`的類型匹配。

So what does the cast method do for us, given that it simply returns its argument? The signature of the cast method takes full advantage of the fact that class Class is generic. Its return type is the type parameter of the Class object:

那麼，考慮到 cast 方法只是返回其參數，它對我們有什麼作用呢？cast 方法的簽名充分利用了類別 Class 是泛型的這一事實。它的返回類型是 Class 物件的類型參數：

```
public class Class<T> {
    T cast(Object obj);
}
```

This is precisely what’s needed by the getFavorite method. It is what allows us to make Favorites typesafe without resorting to an unchecked cast to T.

這正是 getFavorite 方法所需的。它使我們能夠使 Favorites 類型安全，而不必使用未檢查的 T 強制轉換。

There are two limitations to the Favorites class that are worth noting. First, a malicious client could easily corrupt the type safety of a Favorites instance, by using a Class object in its raw form. But the resulting client code would generate an unchecked warning when it was compiled. This is no different from a normal collection implementations such as HashSet and HashMap. You can easily put a String into a `HashSet<Integer>` by using the raw type HashSet (Item 26). That said, you can have runtime type safety if you’re willing to pay for it. The way to ensure that Favorites never violates its type invariant is to have the putFavorite method check that instance is actually an instance of the type represented by type, and we already know how to do this. Just use a dynamic cast:

值得注意的是，Favorites類別有兩個限制。首先，惡意客戶端可以輕易地破壞Favorites實例的類型安全性，方法是使用原始形式的Class對象。但當編譯生成客戶端代碼時，會產生未檢查的警告。這與普通集合實現（如HashSet和HashMap）沒有區別。您可以輕鬆地使用原始類型HashSet（[第26項](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)）將String放入`HashSet<Integer>`中。也就是說，如果您愿意付出代價，就可以具有運行時類型安全性。確保Favorites永遠不會違反其類型不變量的方法是使putFavorite方法檢查instance是否真正是由type表示的類型的實例，我們已經知道如何做到這一點了：只需使用動態轉換：

```
// Achieving runtime type safety with a dynamic cast
public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(type, type.cast(instance));
}
```

There are collection wrappers in java.util.Collections that play the same trick. They are called checkedSet, checkedList, checkedMap, and so forth. Their static factories take a Class object (or two) in addition to a collection (or map). The static factories are generic methods, ensuring that the compile-time types of the Class object and the collection match. The wrappers add reification to the collections they wrap. For example, the wrapper throws a ClassCastException at runtime if someone tries to put a Coin into your `Collection<Stamp>`. These wrappers are useful for tracking down client code that adds an incorrectly typed element to a collection, in an application that mixes generic and raw types.

在java.util.Collections中有一些集合包装器也使用了相同的技巧。它们被称为checkedSet、checkedList、checkedMap等等。除了集合（或映射）之外，它们的静态工厂还需要一个Class对象（或两个）。这些静态工厂是通用方法，确保Class对象和集合的编译时类型匹配。这些包装器将具体化添加到它们所包装的集合中。例如，如果有人试图将Coin放入您的`Collection <Stamp>`中，则该包装器会在运行时抛出ClassCastException异常。这些包装器对于跟踪客户端代码向混合泛型和原始类型应用程序中错误地键入元素的情况非常有用。

The second limitation of the Favorites class is that it cannot be used on a non-reifiable type (Item 28). In other words, you can store your favorite String or String[], but not your favorite `List<String>`. If you try to store your favorite `List<String>`, your program won’t compile. The reason is that you can’t get a Class object for `List<String>`. The class literal `List<String>.class` is a syntax error, and it’s a good thing, too. `List<String>` and `List<Integer>` share a single Class object, which is List.class. It would wreak havoc with the internals of a Favorites object if the “type literals” `List<String>.class` and `List<Integer>.class` were legal and returned the same object reference. There is no entirely satisfactory workaround for this limitation.

Favorites 類別的第二個限制是它不能用於非可具現化型別（[Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md)）。換句話說，您可以儲存您最喜愛的 String 或 String[]，但無法儲存您最喜愛的 `List<String>`。如果您試圖儲存您最喜愛的 `List<String>`，則程式將無法編譯。原因是你無法取得一個 `List<String>` 的 Class 物件。`List<String>.class` 是一個語法錯誤，這也是好事情。`List<String>` 和 `List<Integer>` 共享同一個 Class 物件 List.class。如果「型態字面常數」` List <String> .class` 和 `List <Integer> .class` 合法且返回相同物件引用，那麼它會對 Favorites 物件內部造成混亂。目前沒有完全令人滿意的解決方法。

The type tokens used by Favorites are unbounded: getFavorite and put-Favorite accept any Class object. Sometimes you may need to limit the types that can be passed to a method. This can be achieved with a bounded type token, which is simply a type token that places a bound on what type can be represented, using a bounded type parameter (Item 30) or a bounded wildcard (Item 31).

Favorites 使用的類型標記是無界限制的：getFavorite 和 put-Favorite 接受任何 Class 物件。有時您可能需要限制可以傳遞給方法的類型。這可以通過有界類型標記來實現，它只是一個將類型綁定到可表示的範圍內的類型標記，使用有界類型參數（[Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md)）或有界通配符（[Item-31](/Chapter-5/Chapter-5-Item-31-Use-bounded-wildcards-to-increase-API-flexibility.md)）。

The annotations API (Item 39) makes extensive use of bounded type tokens. For example, here is the method to read an annotation at runtime. This method comes from the AnnotatedElement interface, which is implemented by the reflective types that represent classes, methods, fields, and other program elements:

Annotation API（[Item-39](/Chapter-6/Chapter-6-Item-39-Prefer-annotations-to-naming-patterns.md)）廣泛使用有界類型令牌。例如，以下是在運行時讀取註釋的方法。此方法來自 AnnotatedElement 接口，該接口由反射類型實現，這些反射類型表示類、方法、字段和其他程序元素：


```
public <T extends Annotation>
    T getAnnotation(Class<T> annotationType);
```

The argument, annotationType, is a bounded type token representing an annotation type. The method returns the element’s annotation of that type, if it has one, or null, if it doesn’t. In essence, an annotated element is a typesafe heterogeneous container whose keys are annotation types.

參數annotationType是一個有界的類型標記，代表一個注釋類型。如果該元素具有該類型的注釋，則此方法返回其注釋；否則返回null。實質上，帶註釋的元素是一個安全類型異質容器，其鍵為注釋類型。

Suppose you have an object of type `Class<?>` and you want to pass it to a method that requires a bounded type token, such as getAnnotation. You could cast the object to `Class<? extends Annotation>`, but this cast is unchecked, so it would generate a compile-time warning (Item 27). Luckily, class Class provides an instance method that performs this sort of cast safely (and dynamically). The method is called asSubclass, and it casts the Class object on which it is called to represent a subclass of the class represented by its argument. If the cast succeeds, the method returns its argument; if it fails, it throws a ClassCastException.

假設您有一個 `Class<?>` 類型的對象，並且您想將其傳遞給需要有界類型標記（例如 getAnnotation）的方法。 您可以將該對象轉換為 `Class<? extends Annotation>`，但此轉換未經檢查，因此它會生成編譯時警告（[Item-27](/Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings.md)）。 幸運的是，Class類提供了一種實例方法來安全地執行這種轉換（動態地）。 該方法稱為asSubclass，它將調用它的 Class 對象強制轉換為表示其引數所表示之類別子級別。 如果強制轉換成功，則該方法返回其引數；如果失敗則拋出 ClassCastException。

Here’s how you use the asSubclass method to read an annotation whose type is unknown at compile time. This method compiles without error or warning:

以下是使用asSubclass方法來讀取在編譯時期未知類型的註釋的方法。此方法可以正常編譯，不會出現錯誤或警告：

```
// Use of asSubclass to safely cast to a bounded type token
static Annotation getAnnotation(AnnotatedElement element,String annotationTypeName) {
    Class<?> annotationType = null; // Unbounded type token
    try {
        annotationType = Class.forName(annotationTypeName);
    } catch (Exception ex) {
        throw new IllegalArgumentException(ex);
    }
    return element.getAnnotation(annotationType.asSubclass(Annotation.class));
}
```

In summary, the normal use of generics, exemplified by the collections APIs, restricts you to a fixed number of type parameters per container. You can get around this restriction by placing the type parameter on the key rather than the container. You can use Class objects as keys for such typesafe heterogeneous containers. A Class object used in this fashion is called a type token. You can also use a custom key type. For example, you could have a DatabaseRow type representing a database row (the container), and a generic type `Column<T>` as its key.

總之，泛型的正常使用（例如集合API）限制了您在每個容器中使用固定數量的類型參數。您可以通過將類型參數放在`鍵`而不是容器上來解決此限制。您可以使用Class對象作為這種類型安全異構容器的`鍵`。以這種方式使用的Class對象稱為類型令牌。您還可以使用自定義`鍵`類型。例如，您可以有一個表示數據庫行（即容器）的DatabaseRow類型和一個泛型類`Column<T>`作為其`鍵`。

---
**[Back to contents of the chapter（返回章节目录）](/Chapter-5/Chapter-5-Introduction.md)**
- **Previous Item（上一条目）：[Item 32: Combine generics and varargs judiciously（明智地合用泛型和可变参数）](/Chapter-5/Chapter-5-Item-32-Combine-generics-and-varargs-judiciously.md)**
- **Next Item（下一条目）：[Chapter 6 Introduction（章节介绍）](/Chapter-6/Chapter-6-Introduction.md)**
