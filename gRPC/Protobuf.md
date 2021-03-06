# Protobuf

## Protobuf 消息

---

消息是`Protobuf`中的主要数据传输对象。它们在概念上类似于类。

proto文件用于定义服务与数据结构，是一种高效数据交换格式，相对于Json，Protobuf有更高的转化效率，时间效率和空间效率都是JSON的3-5倍。

```csharp
// proto文件
syntax = "proto3";

option csharp_namespace = "Contoso.Messages";

message Person {
    int32 id = 1;
    string first_name = 2;
    string last_name = 3;
}
```

文件描述：  
- `syntax` 标识Protobuf版本为v3
- `option csharp_namespace` 标识生成C#类的命名空间
- `package` 标识proto文件的命名空间
- `service` 定义服务
- `rpc FuncName (Input) returns (Output)` 定义一个远程过程
- `message` 声明数据结构

前面的消息定义（`Person`）将三个字段指定为名称/值对。与.NET 类型上的属性类似，每个字段都有名称和类型。 

### 类型

`Protobuf`标量值类型与c#对应关系如下：

| Protobuf 类型 | C# 类型   |
|:-----------:|:----------:|
| double      | double     |
| float       | float      |
| int32       | int        |
| int64       | long       |
| uint32      | uint       |
| uint64      | ulong      |
| sint32      | int        |
| sint64      | long       |
| fixed32     | uint       |
| fixed64     | ulong      |
| sfixed32    | int        |
| sfixed64    | long       |
| bool        | bool       |
| string      | string     |
| bytes       | ByteString |

**Tips:** 标量值始终具有默认值，并且该默认值不能设置为`null`。 此约束包括`string`和`ByteString`，它们都属于 C# 类。`string`默认为空字符串值`ByteString`默认为空字节值。尝试将它们设置为`null`会引发错误。

#### 可为null类型

可为 `null` 的包装器类型可用于支持 `null` 值。  
对于需要显式 `null` 的值（例如在 C# 代码中使用 `int?`），`Protobuf` 的“已知类型”包括编译为可以为 `null` 的 C# 类型的包装器。 若要使用它们，请将 `wrappers.proto` 导入到 `.proto` 文件中，如以下代码所示：  

```csharp
syntax = "proto3"

import "google/protobuf/wrappers.proto"

message Person {
    // ...
    google.protobuf.Int32Value age = 5;
}
```

`wrappers.proto` 类型不会在生成的属性中公开。 `Protobuf`会自动将它们映射到 C# 消息中相应的可为 `null` 的 .NET 类型。 例如，`google.protobuf.Int32Value` 字段生成 `int?` 属性。 引用类型属性（如 `string` 和 `ByteString` ）保持不变，但可以向它们分配 null，这不会引发错误。  

| C# 类型      | 已知类型包装器               |
|:----------:|:---------------------------:|
| bool?      | google.protobuf.BoolValue   |
| double?    | google.protobuf.DoubleValue |
| float?     | google.protobuf.FloatValue  |
| int?       | google.protobuf.Int32Value  |
| long?      | google.protobuf.Int64Value  |
| uint?      | google.protobuf.UInt32Value |
| ulong?     | google.protobuf.UInt64Value |
| string     | google.protobuf.StringValue |
| ByteString | google.protobuf.BytesValue  |

#### 时间类型

本机标量类型不提供与 .NET 的 DateTimeOffset、DateTime 和 TimeSpan 等效的日期和时间值。 可使用 Protobuf 的一些“已知类型”扩展来指定这些类型。 这些扩展为受支持平台中的复杂字段类型提供代码生成和运行时支持。

| C# 类型      | 已知类型包装器               |
|:----------:|:---------------------------:|
| DateTimeOffset     | google.protobuf.Timestamp   |
| DateTime    | google.protobuf.Timestamp |
| TimeSpan    | google.protobuf.Duration |

代码如下：

```csharp
syntax = "proto3"

import "google/protobuf/duration.proto";  
import "google/protobuf/timestamp.proto";

message Meeting {
    string subject = 1;
    google.protobuf.Timestamp start = 2;
    google.protobuf.Duration duration = 3;
}
```

#### 字节

Protobuf 支持标量值类型为 `bytes` 的二进制有效负载。 C# 中生成的属性使用 `ByteString` 作为属性类型。  
使用 `ByteString.CopyFrom(byte[] data)` 从字节数组创建新实例：

```csharp
var data = await File.ReadAllBytesAsync(path);

var payload = new PayloadResponse();
payload.Data = ByteString.CopyFrom(data);
```

使用 `ByteString.Span` 或 `ByteString.Memory` 直接访问 `ByteString` 数据。 或调用 `ByteString.ToByteArray()` 将实例转换回字节数组：  

```csharp
var payload = await client.GetPayload(new PayloadRequest());

await File.WriteAllBytesAsync(path, payload.Data.ToByteArray());
```

#### 小数

Protobuf 本身不支持 .NET `decimal` 类型，只支持 `double` 和 `float`。 
可以创建消息定义来表示 `decimal` 类型，以便在 .NET 客户端和服务器之间实现安全序列化。 但其他平台上的开发人员必须了解所使用的格式，并能够实现自己对其的处理。

```csharp
package CustomTypes;

// Example: 12345.6789 -> { units = 12345, nanos = 678900000 }
message DecimalValue {

    // Whole units part of the amount
    int64 units = 1;

    // Nano units of the amount (10^-9)
    // Must be same sign as units
    sfixed32 nanos = 2;
}
```

`nanos` 字段表示从 `0.999_999_999` 到 `-0.999_999_999` 的值。 例如，`decimal` 值 `1.5m` 将表示为 `{ units = 1, nanos = 500_000_000 }`。 这就是此示例中的 `nanos` 字段使用 `sfixed32` 类型的原因：对于较大的值，其编码效率比 `int32` 更高。 如果 `units` 字段为负，则 `nanos` 字段也应为负。

此类型与 BCL decimal 类型之间的转换可能会在 C# 中实现，如下所示：  

```csharp
namespace CustomTypes
{
    public partial class DecimalValue
    {
        private const decimal NanoFactor = 1_000_000_000;
        public DecimalValue(long units, int nanos)
        {
            Units = units;
            Nanos = nanos;
        }

        public long Units { get; }
        public int Nanos { get; }

        public static implicit operator decimal(CustomTypes.DecimalValue grpcDecimal)
        {
            return grpcDecimal.Units + grpcDecimal.Nanos / NanoFactor;
        }

        public static implicit operator CustomTypes.DecimalValue(decimal value)
        {
            var units = decimal.ToInt64(value);
            var nanos = decimal.ToInt32((value - units) * NanoFactor);
            return new CustomTypes.DecimalValue(units, nanos);
        }
    }
}
```

#### 集合

Protobuf 中，在字段上使用 `repeated` 前缀关键字指定列表。 以下示例演示如何创建列表：

```csharp
message Person {
    // ...
    repeated string roles = 8;
}
```

在生成的代码中，`repeated` 字段由 `Google.Protobuf.Collections.RepeatedField<T>` 泛型类型表示。

```csharp
public class Person
{
    // ...
    public RepeatedField<string> Roles { get; }
}
```

`RepeatedField<T>` 可实现 `IList<T>`。 因此你可使用 LINQ 查询，或者将其转换为数组或列表。 `RepeatedField<T>` 属性没有公共 `setter`。 项应添加到现有集合中。

```csharp
var person = new Person();

// Add one item.
person.Roles.Add("user");

// Add all items from another collection.
var roles = new [] { "admin", "manager" };
person.Roles.Add(roles);
```

#### 字典

.NET `IDictionary<TKey,TValue>` 类型在 Protobuf 中使用 `map<key_type, value_type>` 表示。

```csharp
message Person {
    // ...
    map<string, string> attributes = 9;
}
```

在生成的 .NET 代码中，`map` 字段由 `Google.Protobuf.Collections.MapField<TKey, TValue>` 泛型类型表示。 `MapField<TKey, TValue>` 可实现 `IDictionary<TKey,TValue>`。 与 `repeated` 属性一样，`map` 属性没有公共 `setter`。 项应添加到现有集合中。

```csharp
var person = new Person();

// Add one item.
person.Attributes["created_by"] = "James";

// Add all items from another collection.
var attributes = new Dictionary<string, string>
{
    ["last_modified"] = DateTime.UtcNow.ToString()
};
person.Attributes.Add(attributes);
```

#### Oneof

`oneof` 字段是一种语言特性。 编译器在生成消息类时处理 `oneof` 关键字。 使用 `oneof` 指定可能返回 `Person` 或 `Error` 的响应消息可能如下所示：

```csharp
message Person {
    // ...
}

message Error {
    // ...
}

message ResponseMessage {
  oneof result {
    Error error = 1;
    Person person = 2;
  }
}
```

在整个消息声明中，`oneof` 集内的字段必须具有唯一的字段编号。  
使用 `oneof` 时，生成的 C# 代码包括一个枚举，用于指定哪些字段已设置。 可以测试枚举来查找已设置的字段。 未设置的字段将返回 `null` 或默认值，而不是引发异常。

```csharp
var response = await client.GetPersonAsync(new RequestMessage());

switch (response.ResultCase)
{
    case ResponseMessage.ResultOneofCase.Person:
        HandlePerson(response.Person);
        break;
    case ResponseMessage.ResultOneofCase.Error:
        HandleError(response.Error);
        break;
    default:
        throw new ArgumentException("Unexpected result.");
}
```

#### Value

`Value` 类型表示动态类型的值。 它可以是 `null`、数字、字符串、布尔值、值字典 (`Struct`) 或值列表 (`ValueList`)。 `Value` 是一个 Protobuf 已知类型，它使用前面讨论的 `oneof` 功能。 若要使用 `Value` 类型，请导入 `struct.proto`。

```csharp
import "google/protobuf/struct.proto";

message Status {
    // ...
    google.protobuf.Value data = 3;
}
```

```csharp
// Create dynamic values.
var status = new Status();
status.Data = Value.ForStruct(new Struct
{
    Fields =
    {
        ["enabled"] = Value.ForBool(true),
        ["metadata"] = Value.ForList(
            Value.ForString("value1"),
            Value.ForString("value2"))
    }
});

// Read dynamic values.
switch (status.Data.KindCase)
{
    case Value.KindOneofCase.StructValue:
        foreach (var field in status.Data.StructValue.Fields)
        {
            // Read struct fields...
        }
        break;
    // ...
}
```

直接使用 `Value` 可能很冗长。 使用 `Value` 的替代方法是通过 Protobuf 的内置支持，将消息映射到 `JSON`。 Protobuf 的 `JsonFormatter` 和 `JsonWriter` 类型可用于任何 Protobuf 消息。 `Value` 特别适用于与 `JSON` 进行转换。

以下是与之前的代码等效的 `JSON`：

```csharp
// Create dynamic values from JSON.
var status = new Status();
status.Data = Value.Parser.ParseJson(@"{
    ""enabled"": true,
    ""metadata"": [ ""value1"", ""value2"" ]
}");

// Convert dynamic values to JSON.
// JSON can be read with a library like System.Text.Json or Newtonsoft.Json
var json = JsonFormatter.Default.Format(status.Data);
var document = JsonDocument.Parse(json);
```

### 样式

建议使用 `underscore_separated_names` 作为字段名称。 为 .NET 应用创建的新 Protobuf 消息应遵循 Protobuf 样式准则。 .NET 工具会自动生成使用 .NET 命名标准的 .NET 类型。 例如，`first_name` Protobuf 字段生成 `FirstName` .NET 属性。



生成应用时，Protobuf 工具将从 .proto 文件生成 .NET 类型。 Person 消息生成 .NET 类：

```csharp
public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```