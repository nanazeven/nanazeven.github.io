

#  Weblogic

weblogic支持部署大型分布式应用，同时是管理和配置整个域中所有资源的中心点，包括web server，web应用程序和资源jdbc或者DataSource

weblogic本身就是分布式设计程序，可以部署weblogic集群来协同工作有更好的伸缩性也更可靠，有集群，也就存在集群间的通信和数据同步，大部分weblogic漏洞都发现在这里

WebLogic版本众多，经常见到的只有两个类别：10.x和12.x，这两个大版本也叫WebLogic Server 11g和WebLogic Server 12c。

- Oracle WebLogic Server 10.3.6 支持的最低JDK版本为JDK1.6
- Oracle WebLogic Server 12.1.3 支持的最低JDK版本为JDK1.7
- Oracle WebLogic Server 12.2.1 12.2.1及以上支持的最低JDK版本为JDK1.8
- Oracle WebLogic Server 12.2.1.1
- Oracle WebLogic Server 12.2.1.2
- Oracle WebLogic Server 12.2.1.3

在上面的众多反序列化中，不同的jdk版本他的利用链也不一样，在不同的weblogic版本中引用的依赖jar包版本也不尽相同，就很讨厌

##  Weblogic RMI

与java RMI的通信协议JRMP不同的是,WebLogic RMI使用的是T3协议

weblogic rmi完全兼容Java rmi的对象,也可以动态加载class,同时优化了套接字的使用

RMI客户端可以使用多种的URL协议:rmi://, http:// ,iiop://. 而且这些协议使用的端口可以是同一个,weblogic会判断协议类型然后路由到正确的位置解析

## XMLDecoder 反序列化

### XMLDecder简介

XMLDecoder是Philip Mine 在 JDK 1.4 中开发的一个用于将JavaBean或POJO对象序列化和反序列化的一套API，开发人员可以通过利用XMLDecoder的readObject()方法将任意的XML反序列化，从而使得整个程序更加灵活。

XMLDecoder使用的是SAX解析规范，在调用readObject()后会使用`SAXParserFactory`工厂生成SAXParser实例，在调用`SAXParser.parse()`中初始化DocumentHandler用于解析xml数据流，DocumentHandler继承与DefaultHandler，weblogic中默认的解析xml和创建事件的handle。XMLDecoder在com.sun.beans.decoder实现了DocumentHandler，其中定义各种标签对应的处理器handle，用于构造反序列化poc的元素。

```xml
// weblogic xmldecoder反序列化漏洞poc 最初的poc版本没有任何过滤
<java>
    <object class="java.lang.ProcessBuilder">
        <array class="java.lang.String" length="1">
            <void index="0">
                <string>calc</string>
            </void>
        </array>
        <void method="start"/>
    </object>
</java>

```

##### object标签

处理类是`ObjectElementHandler`类，继承于NewElementHandler类用于加载一个对象，存在object标签的class属性时，会调用父类NewElementHandler的`addAttribute()`寻找class属性值`java.lang.ProcessBuilder`的class对象引用到`ObjectElementHandler`的`type`属性

##### array标签

用于常见一个arraylist数组，作为父标签的参数

##### String标签

定义一个字符串，继承于ElementHandler，在endElementhandle()将值添加到父节点的Argument中

##### void标签

处理器类为VoidElementHandler类，继承于ObjectElementHandler类，且只重写了isArgument方法。这里设置method属性

![path](https://nanazeven.github.io/image/xmldecoder1.png)

结束void标签是调用VoidElementHandler的endElement函数。由于继承关系：voidElementHandler->objectElementHandler->newElementHandler->ElementHandler，调用的ElementHandler#endElement后调用newendElement的getValueObject(无参)函数，最终调用objectElementHandler的getValueObject(有参)函数：


在objectElementHandler的getValueObject函数中调用getContentBean函数获取操作对象

![path](https://nanazeven.github.io/image/xmldecoder2.png)

上面的geContentbean会调用父节点也就是object的getValueObjec函数，这里通过创建Expression对象创建了java.lang.ProcessBuilder的对象并返回


之后再次回到void标签处理器实例的getValueObjec函数，通过Expression执行start函数

![path](https://nanazeven.github.io/image/xmldecoder3.png)



##### xmlDecoder反序列化CVE

- CVE-2017-3506 修复是过滤object标签，利用void和object处理器类的继承关系，void可以替换object标签绕过
- CVE-2017-10271 过滤了object/new/method标签，void标签只允许用index，array的class只能用byte
- CVE-2019-2725 