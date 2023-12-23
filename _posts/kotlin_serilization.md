### 官方文档

https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md

https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/serialization-guide.md

`kotlin serialization`支持`Json`、`HOCON`、`ProtoBuf`、`CBOR`、`Properties`等格式的序列化/反序列化。

目前只有`Json`是`Stable`状态。

通过了解，此库只支持静态序列化/反序列化，使用插件生成对应类的序列化器。对于基础类型提供序列化器实现，对`Pair`、`Triple`、`List`、`Set`、`Map`以及`Unit`、`Noting`等内置`object`也提供序列化器。

**对于第三方类，以及`Java`一些常用类并不提供序列化器比如`Date`，使用这些类需要手动编写序列化器。**

### 引入依赖

```groovy
// project build.gradle

//方式1
// Top-level build file where you can add configuration options common to all sub-projects/modules.
plugins {
    id 'com.android.application' version '8.2.0' apply false
    id 'org.jetbrains.kotlin.android' version '1.9.10' apply false
    id 'org.jetbrains.kotlin.plugin.serialization' version '1.9.21'
}

//方式2
buildscript {
    dependencies {
        classpath 'org.jetbrains.kotlin.plugin.serialization:1.9.21'
    }
}
```

```groovy
// app build.gradle

dependencies {
    implementation "org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2"
}
```



### 使用

```kotlin
@Serializable
class Project(val name: String, val language: String)
 
fun main() {
    var project = Project("kotlinx.serialization", "Kotlin")
    val json = Json.encodeToString(project)
    println(json)
    project = Json.decodeFromString(json)
}
```

### 注意事项

* 使用`@Serializable`注解的类才支持序列化与反序列化，kotlin序列化插件会自动生成并连接此类的序列化器

* 具有幕后字段的属性才支持序列化

* 被反序列化的类会执行`init { }`

* ```kotlin
  //仅当输入中存在存在所有属性才支持反序列化
  
  @Serializable
  data class Project(val name: String, val language: String)
  
  fun main() {
      val data = Json.decodeFromString<Project>("""
          {"name":"kotlinx.serialization"}
      """)
      println(data)
  }
  
  //Exception in thread "main" kotlinx.serialization.MissingFieldException: Field 'language' is required for type with serial name 'example.exampleClasses04.Project', but it was missing at path: $
  
  //解决方法是为属性添加默认值
  data class Project(val name: String, val language: String = "kotlin")
  ```

* 使用[@Transient](https://kotlinlang.org/api/kotlinx.serialization/kotlinx-serialization-core/kotlinx.serialization/-transient/index.html)排除不参与序列化/反序列化的类

* 默认情况下，默认值不参与序列化，可以通过 `@EncodeDefault`注解改变此行为

* `kotlin-serialization`支持空安全

* 使用`@Serializable`注解的类所引用的类也必须使用`@Serializable`注解才支持序列化

### 通用序列化类

```kotlin
@Serializable
class Box<T>(val contents: T)
```

```kotlin
@Serializable
class Data(
    val a: Box<Int>,
    val b: Box<Project>
)

fun main() {
    val data = Data(Box(42), Box(Project("kotlinx.serialization", "Kotlin")))
    println(Json.encodeToString(data))
}
```

```kotlin
{"a":{"contents":42},"b":{"contents":{"name":"kotlinx.serialization","language":"Kotlin"}}}
```

### 序列化字段名称

```kotlin
@Serializable
class Project(val name: String, @SerialName("lang") val language: String)

fun main() {
    val data = Project("kotlinx.serialization", "Kotlin")
    println(Json.encodeToString(data))
}
```

### 内置序列化类

Kotlin 序列化具有以下 10 个原语： `Boolean`、`Byte`、`Short`、`Int`、`Long`、`Float`、`Double`、`Char`、`String`和 `enum`。Kotlin 序列化中的其他类型是*复合类型*——由这些原始值组成。

`Pair`、`Triple`、`List`、`Map`、`Set`也支持序列化。

```kotlin
@Serializable
class Project(val name: String)

fun main() {
    val pair = 1 to Project("kotlinx.serialization")
    println(Json.encodeToString(pair))
}

// 输出
{"first":1,"second":{"name":"kotlinx.serialization"}}
```



### 长数字

```kotlin
@Serializable
class Data(val signature: Long)

fun main() {
    val data = Data(0x1CAFE2FEED0BABE0)
    println(Json.encodeToString(data))
}
//序列化输出
{"signature":2067120338512882656}
```

上面的序列化输出在`javascript`中反序列化会发生截断

```javascript
JSON.parse("{\"signature\":2067120338512882656}")
//console.log
▶ {signature: 2067120338512882700} 
```

可以使用`LongAsStringSerializer`序列化程序将`长数字`序列化为字符串

```kotlin
@Serializable
class Data(
    @Serializable(with=LongAsStringSerializer::class)
    val signature: Long
)

fun main() {
    val data = Data(0x1CAFE2FEED0BABE0)
    println(Json.encodeToString(data))
}

// 输出
{"signature":"2067120338512882656"}
```

### 为枚举类指定序列化名称

```kotlin
@Serializable // required because of @SerialName
enum class Status { @SerialName("maintained") SUPPORTED }
        
@Serializable
class Project(val name: String, val status: Status) 

fun main() {
    val data = Project("kotlinx.serialization", Status.SUPPORTED)
    println(Json.encodeToString(data))
}
// 输出
{"name":"kotlinx.serialization","status":"maintained"}
```

### Unit 与 object

`Unit`也是一个`object`，从概念上来讲`object`是只有一个实例的类。

在`kotlin-serialization`中，对象被序列化成空结构。

```kotlin
@Serializable
object SerializationVersion {
    val libraryVersion: String = "1.0.0"
}

fun main() {
    println(Json.encodeToString(SerializationVersion))
    println(Json.encodeToString(Unit))
}  
// 输出
{}
{}
```

### Duration

[Duration](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.time/-duration/)从`kotline 1.7.20`已变得可序列化。

```kotlin
fun main() {
    val duration = 1000.toDuration(DurationUnit.SECONDS)
    println(Json.encodeToString(duration))
}
// 输出
"PT16M40S"
```

### Nothing

默认情况下，[Nothing](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-nothing.html)是可序列化的类。但是，由于没有此类的实例，因此无法对其值进行编码或解码 - 任何尝试都会导致异常。

当语法上需要某种类型时使用此序列化器，但实际上并不在序列化中使用它。例如，当使用参数化多态基类时：

```kotlin
@Serializable
sealed class ParametrizedParent<out R> {
    @Serializable
    data class ChildWithoutParameter(val value: Int) : ParametrizedParent<Nothing>()
}

fun main() {
    println(Json.encodeToString(ParametrizedParent.ChildWithoutParameter(42)))
}
```



### 序列化器

https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/serializers.md

* `kotlin`序列化插件会为内置序列化类生成序列化器。

* `kotlin`序列化插件会为`@Serializable`注解的类生成序列化器。

* 获取序列化器：`Class.serializer()`

### 自定义序列化器：

```kotlin
class Color(val rgb: Int)
//{color : "#00ff00"}
object ColorAsStringSerializer : KSerializer<Color> {
    override val descriptor: SerialDescriptor = PrimitiveSerialDescriptor("Color", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: Color) {
        val string = value.rgb.toString(16).padStart(6, '0')
        encoder.encodeString(string)
    }

    override fun deserialize(decoder: Decoder): Color {
        val string = decoder.decodeString()
        return Color(string.toInt(16))
    }
}
```

### 绑定到类：

```kotlin
@Serializable(with = ColorAsStringSerializer::class)
class Color(val rgb: Int)
```



### 序列化第三方类

必须为第三方类编写序列化器，如为`Date`编写序列化器：

```kotlin
object DateAsLongSerializer : KSerializer<Date> {
    override val descriptor: SerialDescriptor = PrimitiveSerialDescriptor("Date", PrimitiveKind.LONG)
    override fun serialize(encoder: Encoder, value: Date) = encoder.encodeLong(value.time)
    override fun deserialize(decoder: Decoder): Date = Date(decoder.decodeLong())
}

fun main() {                                              
    val kotlin10ReleaseDate = SimpleDateFormat("yyyy-MM-ddX").parse("2016-02-15+00") 
    println(Json.encodeToString(DateAsLongSerializer, kotlin10ReleaseDate))    
}
```

