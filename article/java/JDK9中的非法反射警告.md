反射是一项相当强大的特性，不仅在各类框架中被广泛应用，即使是在日常开发中我们也隔三差五得要和它打交道。然而在JDK9中JDK对反射加上了一些限制，需要注意。

考虑有如下的代码：

```java
import java.lang.reflect.Field;
import java.util.ArrayList;

public class TestReflect {
    public static int getCapacity(ArrayList<?> l) throws Exception {
        Field dataField = l.getClass().getDeclaredField("elementData");
        dataField.setAccessible(true); // 即使设置了可访问也会触发警告
        return ((Object[]) dataField.get(l)).length; // 注意这行
    }

    public static void main(String[] args) {
        var arr = new ArrayList<Integer>(4);
        try {
            System.out.println("capacity:" + TestReflect.getCapacity(arr));        
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

这段代码的作用是读取`ArrayList`的实际容量，由于JDK并没有为我们提供类似`cap()`这样的公开接口，所以我们不得不使用反射来绕过限制。

在JDK8上这段代码运行良好，然而当我们升级成JDK9后却会是这样的一幅画面：

```text
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by TestReflect (file:/tmp/TestReflect.java) to field java.util.ArrayList.elementData
WARNING: Please consider reporting this to the maintainers of TestReflect
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
capacity:4
```

别紧张，代码还是正常运行了。其实这是JDK9中添加的新特性，即reflect不再可以访问non-public成员以及不可公开访问的class，原先这些访问控制虽然存在但是可以通过reflect绕过，从JDK9开始反射也将遵循访问控制的规则。JDK9中对于第一次访问非公开成员的操作会显示警告信息，我们可以通过``选项进一步显示出有用的warning提示：

```bash
$ java --illegal-access=warn TestReflect.java

WARNING: Illegal reflective access by TestReflect (file:/tmp/TestReflect.java) to field java.util.ArrayList.elementData
capacity:4
```

将warn替换为debug可以指出非法访问发生在哪一行（通过观察打印的调用堆栈），而替换为deny则将会直接抛出`java.lang.reflect.InaccessibleObjectException`异常：

```bash
$ java --illegal-access=deny TestReflect.java

java.lang.reflect.InaccessibleObjectException: Unable to make field transient java.lang.Object[] java.util.ArrayList.elementData accessible: module java.base does not "opens java.util" to unnamed module @72967906
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:349)
        at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:289)
        at java.base/java.lang.reflect.Field.checkCanSetAccessible(Field.java:174)
        at java.base/java.lang.reflect.Field.setAccessible(Field.java:168)
        at TestReflect.getCapacity(TestReflect.java:7)
        at TestReflect.main(TestReflect.java:14)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:564)
        at jdk.compiler/com.sun.tools.javac.launcher.Main.execute(Main.java:415)
        at jdk.compiler/com.sun.tools.javac.launcher.Main.run(Main.java:192)
        at jdk.compiler/com.sun.tools.javac.launcher.Main.main(Main.java:132)
```

对于后续的更新版本，JDK会将deny作为默认的行为，不过目前(14.0.2)默认行为依旧为permit（显示那些默认的warning信息）。

显而易见，新特性会导致如下的结果：

1. 无法再通过反射修改/访问其他类型的私有成员;
2. 无法再通过反射使用internal APIs

然而健康的代码并不应该依赖于非法的访问：

1. 标记为非公开的成员本身就不希望被外部直接访问，访问应该通过public APIs进行;
2. 无视访问控制会导致数据的修改变得不可控，从而产生意想不到的缺陷;
3. 私有成员可能会随着开发/更新/修复/重构等活动而改变，过度依赖于访问其他class的私有成员会产生高耦合性的代码，使维护的负担成倍增加

当然，想要消除警告也很容易，虽然官方文档中不建议无视或消除这个警告：

```bash
$ java --add-opens java.base/java.util=ALL-UNNAMED TestReflect.java

capacity:4
```

`--add-opens`选项将特定的module下的package公开给制定的package，或者使用特殊值`ALL-UNAMED`代指所有匿名包（比如本例中的类），公开后我们就可以通过reflect来访问被公开包中class的成员而不抛出异常了（需要使用`setAccessible(true)`）

这种trick实际上严重破坏了代码的封装，个人是极不推荐的;此外还可以选择将JDK版本锁定在8，当然这也是不得以才为之的办法，那么对于即想尝试新版本又不想被警告信息烦扰的话该怎么办呢，只能尝试下面几种办法了：

1. 如果是第三方依赖导致的警告，升级对应的依赖或是联系该依赖的维护人员，请他们尽快修复问题。通常来说这类向后兼容的问题修复无需花费很多的时间。
2. 如果是自己的代码，则考虑是否要对私有成员提供可访问的public API，是否将私有成员公有化。
3. 如果使用了某些internal apis，那么你的代码应该重构以避免过度依赖这些api。

##### 参考资料

<https://blog.codefx.org/java/java-9-migration-guide/#Illegal-Access-To-Internal-APIs>
<http://mail.openjdk.java.net/pipermail/jigsaw-dev/2017-June/012841.html>
