# 常用的工具类



## 数学运算BLAS

**Java Blas库：**

- **[jblas](http://jblas.org/)** is based on BLAS and LAPACK；
- [Apache Commons Math](http://commons.apache.org/proper/commons-math/) for the most popular mathematics library in Java (not using netlib-java）；
- **[Breeze](https://github.com/scalanlp/breeze)** for high performance linear algebra in Scala and Spark (builds on top of luhenry/netlib)；
- [**luhenry/netlib**: An high-performance, hardware-accelerated implementation of Netlib in Java](https://github.com/luhenry/netlib)

免费版不再提供：

- [Matrix Toolkits for Java](https://github.com/fommil/matrix-toolkits-java/) for high performance linear algebra in Java (builds on top of netlib-java)；
- [**netlib-java**](https://github.com/fommil/netlib-java) is a wrapper for low-level BLAS, LAPACK and ARPACK（不再开发免费版）；

## [FasterXML Jackson](https://github.com/FasterXML/jackson)

> Versions 3.x require JDK 17
>
> - 2.x (`com.fasterxml.jackson`) is the older actively developed and maintained version
> - 3.x (`tools.jackson`) is the newer actively developed and maintained version

[text 格式](https://github.com/FasterXML/jackson-dataformats-text)： CSV, Java Properties, YAML

[json 格式](https://github.com/FasterXML/jackson-core)：Json 

[binary 格式](https://github.com/FasterXML/jackson-dataformats-binary)：Avro, BSON(binary json), CBOR, Smile, **Protobuf**, 

### JSON

一般会直接使用 [data-bind](https://github.com/FasterXML/jackson-databind)，将JSON与Java对象进行转换，

Maven地址

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>${jackson.version.core}</version>
</dependency>
```

#### 使用 

```java
// YAML 格式解析示例，使用不同的 Factory
ObjectMapper mapper = new ObjectMapper(new YAMLFactory());  
// 示例1：对象映射
User user = mapper.readValue(yamlString, User.class);
// 带泛型的反序列化
Result<List<AfsStatus>>> tmp = mapper.readValue(yamlSource, new TypeReference<Result<List<AfsStatus>>>() {});

// 示例2：Tree Model 形式
JsonNode rootNode = objectMapper.readTree(yamlString);  
JsonNode nameNode = rootNode.get("name");  

// 示例3 流式解析
YAMLFactory factory = new YAMLFactory();
YAMLParser parser = factory.createParser(yamlString);
while (parser.nextToken() != null) {
  // do something!
}
```

#### 枚举

```java
// 使用 ENUM 的 toString 进行序列化和反序列化
objectMapper.configure(SerializationFeature.WRITE_ENUMS_USING_TO_STRING, true);
objectMapper.configure(DeserializationFeature.READ_ENUMS_USING_TO_STRING, true);
```

#### 自定义类解析

```java
ObjectMapper objectMapper = new ObjectMapper()
SimpleModule module = new SimpleModule();
module.addDeserializer(Schema.class, new SchemaDeserializer());
objectMapper.registerModule(module);
```

#### 多态序列化

```java
// 通过 name 属性字段，决定使用哪个子类（同时会正常赋值到子类的 name 属性）
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY,property = "name")
@JsonSubTypes(value = {
     @JsonSubTypes.Type(value = Daedalus.class, name = "Daedalus"),
     @JsonSubTypes.Type(value = HugeDrink.class, name = "Huge drink"),
     @JsonSubTypes.Type(value = StarWand.class, name = "Star wand"),
})
public interface Equipment {
}

@Data
public class StarWand implements Equipment{
    private String name;
    private int length;
    private int price;
    private List<String> effect;
}

// @JsonSubTypes的注解,可以通过代码操作
mapper.registerSubtypes(new NamedType(HugeDrink.class, "Huge drink"));
mapper.registerSubtypes(new NamedType(Daedalus.class, "Daedalus"));
mapper.registerSubtypes(new NamedType(StarWand.class, "Star wand"));

// 处理 name字段不存在或者值不在配置的值集合中时，如何处理
objectMapper.addHandler(new DeserializationProblemHandler() {
    @Override
    public JavaType handleMissingTypeId(DeserializationContext ctxt, JavaType baseType, TypeIdResolver idResolver, String failureMsg) throws IOException {
        return baseType;
    }

    @Override
    public JavaType handleUnknownTypeId(DeserializationContext ctxt, JavaType baseType, String subTypeId, TypeIdResolver idResolver, String failureMsg) throws IOException {
        return baseType;
    }
});
```

新增子类的时候都要去加一下JsonSubTypes，通过 `@JsonTypeIdResolver` 解决

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.CUSTOM, property = "name")
@JsonTypeIdResolver(SchemaIdResolver.class)
public interface Equipment {
    String getName();
}

public class SchemaIdResolver extends TypeIdResolverBase {
    private JavaType baseType; // 基础类型（Equipment接口）

    // 初始化基础类型
    @Override
    public void init(JavaType baseType) {
        this.baseType = baseType;
    }

    // 序列化时：从对象获取type值（比如ServerEquipment→"server"）
    @Override
    public String idFromValue(Object value) {
        return ((Equipment) value).getName();
    }
    
    // 反序列化时：根据type值映射到具体实现类
    @Override
    public JavaType typeFromId(DeserializationContext ctxt, String id) {
        // 灵活映射：可从配置文件、数据库、枚举等读取映射关系
        switch (id) {
            case "Star wand":
                return TypeFactory.defaultInstance().constructSpecializedType(baseType, StarWand.class);
            // 其他 case 情形
            default:
                // 未知type返回null，触发handleUnknownTypeId
                return null;
        }
    }
}
```

### [JsonPath](https://github.com/json-path/JsonPath) 

根据路径解析所需的数据，**通过 SPI 机制，支持Gson，FastXML jackson 等 json 解析工具**。

- ` Double[] prices = JsonPath.read(json, "$.store.book[*].price");`

```xml
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
    <version>2.4.0</version>
</dependency>
```

## Hutools

一个Java基础工具类，对文件、流、加密解密、转码、正则、线程、XML等JDK方法进行封装，组成各种Util工具类，同时提供以下组件：

| 模块               | 介绍                                                         |
| ------------------ | ------------------------------------------------------------ |
| hutool-aop         | **JDK动态代理封装，提供非 IOC下的切面支持**                  |
| hutool-bloomFilter | 布隆过滤，提供一些Hash算法的布隆过滤                         |
| hutool-cache       | 简单缓存实现                                                 |
| hutool-core        | 核心，包括Bean操作、日期、各种Util等                         |
| hutool-cron        | 定时任务模块，提供类Crontab表达式的定时任务                  |
| hutool-crypto      | 加密解密模块，提供对称、非对称和摘要算法封装                 |
| hutool-db          | JDBC封装后的数据操作，基于ActiveRecord思想                   |
| hutool-dfa         | 基于DFA模型的多关键字查找                                    |
| hutool-extra       | 扩展模块，对第三方封装（模板引擎、邮件、Servlet、二维码、Emoji、FTP、分词等） |
| hutool-http        | 基于HttpUrlConnection的Http客户端封装                        |
| hutool-log         | 自动识别日志实现的日志门面                                   |
| hutool-script      | 脚本执行封装，例如Javascript                                 |
| hutool-setting     | 功能更强大的Setting配置文件和Properties封装                  |
| hutool-system      | 系统参数调用封装（JVM信息等）                                |
| hutool-json        | JSON实现                                                     |
| hutool-captcha     | 图片验证码实现                                               |
| hutool-poi         | 针对POI中Excel和Word的封装                                   |
| hutool-socket      | 基于Java的NIO和AIO的Socket封装                               |
| hutool-jwt         | JSON Web Token (JWT)封装实现                                 |

可以根据需求对每个模块单独引入，也可以通过引入`hutool-all`方式引入所有模块。

### 使用

```xml
<dependency>
	<groupId>cn.hutool</groupId>
	<artifactId>hutool-all</artifactId>
	<version>5.8.16</version>
</dependency>
```

## Image

图像处理：Java用于图像读取和处理的包；

### 库

#### [OpenCV](https://opencv.org/releases/)

由BSD许可证发布，可以免费学习和商业使用，提供了包括 C/C++、Python 和 Java 等主流编程语言在内的接口。OpenCV 专为计算效率而设计，强调实时应用，可以充分发挥多核处理器的优势。

[openpnp](https://github.com/openpnp/opencv) 对 [opencv](https://github.com/opencv/opencv)做简单的封装（将windows/linux等平台的opencv动态库放入jar包中，并且提供OpenCV.loadShared 进行加载）。

```xml
<dependency>
 <groupId>org.openpnp</groupId>
 <artifactId>opencv</artifactId>
 <version>4.5.5</version>
</dependency>
```

#### [Commons Imaging](https://github.com/apache/commons-imaging)

Apache Commons Imaging，一个读取和写入各种图像格式的库，包括快速解析图像信息（如大小，颜色，空间，ICC配置文件等）和元数据。

#### [metadata-extractor](https://github.com/drewnoakes/metadata-extractor)

> **Extracts Exif, IPTC, XMP, ICC and other *metadata* from image, video and audio files**

https://drewnoakes.com/code/exif/

maven依赖

```xml
<dependency>
    <groupId>com.drewnoakes</groupId>
    <artifactId>metadata-extractor</artifactId>
    <version>2.13.0</version>
</dependency>
```

### 使用

#### 读写

BufferedImage ：图像类

ImageIO：图形读写工具类

```java
// 读取图片
File input = new File("ceshi.jpg");
BufferedImage image = ImageIO.read(input);

// 根据图片格式，创建ImageWrite类
Iterator<ImageWriter> writers =  ImageIO.getImageWritersByFormatName("jpg");
ImageWriter writer = (ImageWriter) writers.next();

// 创建对应的文件流
File compressedImageFile = new File("bbcompress.jpg");
OutputStream os =new FileOutputStream(compressedImageFile);
ImageOutputStream ios = ImageIO.createImageOutputStream(os);

// 设置输出流
writer.setOutput(ios);

// 设置参数（压缩模式和压缩质量）
ImageWriteParam param = writer.getDefaultWriteParam();
param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
param.setCompressionQuality(0.01f);

// 将图像写出
writer.write(null, new IIOImage(image, null, null), param);

os.close();
ios.close();
writer.dispose();
```

#### Compress

```java
// 设置参数（压缩模式和压缩质量）
ImageWriteParam param = writer.getDefaultWriteParam();
param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
param.setCompressionQuality(0.01f);
```

压缩模式一共有四种：

- MODE_EXPLICIT 表示 ImageWriter 可以根据后续的 set 的附加信息进行平铺和压缩，比如说接下来的 `setCompressionQuality()` 方法。

`setCompressionQuality()` 方法的参数是一个 0-1 之间的数，0.0 表示尽最大程度压缩，1.0 表示保证图像质量很重要。

- 有损压缩：压缩质量应该控制文件大小和图像质量之间的权衡（例如，通过在写入 JPEG 图像时选择量化表）；
- 无损压缩：压缩质量可用于控制文件大小和执行压缩所需的时间之间的权衡（例如，通过优化行过滤器并在写入 PNG 图像时设置 ZLIB 压缩级别）。

**OpenCV**

```java
OpenCV.loadShared();

Mat src = Imgcodecs.imread(imagePath);

// 第一个参数 IMWRITE_JPEG_QUALITY 表示对图片的质量进行改变，第二个是质量因子，1-100，值越大表示质量越高。
MatOfInt dstImage = new MatOfInt(Imgcodecs.IMWRITE_JPEG_QUALITY, 10);

Imgcodecs.imwrite(imageOutPath, src, dstImage);
```



