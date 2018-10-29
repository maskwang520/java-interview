这里声明下，Set里面对应的实现分别是HashSet,LinkedHashSet,TreeSet,都是基于对应的HashMap，HashSet,TreeMap来实现的，少许的变化，底层的支撑Map都是对应的Map.
### 1. HashSet则是利用HashMap的实现
```java
// Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

      public HashSet() {
        map = new HashMap<>();
    }

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

### 2. LinkedHashSet则继承于HashSet.
```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {


}
```

### 3. TreeSet则是利用了TreeMap的实现
```java
private static final Object PRESENT = new Object();
 public TreeSet() {
        this(new TreeMap<E,Object>());
    }
```-