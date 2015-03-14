# Performance Measurements #

The unit-test 'SerializeDeserializePerformanceTest' provides the performance of following serializers:

For .NET:
  * **EneterProtoBufSerializer** - serializer using Protocol Buffers for .NET implemented by Marc Gravell.
  * **BinarySerializer** -  binary serializer using BinaryFormatter.
  * **XmlStringSerializer** - text serializer using XmlSerializer.

For Java:
  * **EneterProtoBufSerializer** - serializer using Protocol Buffers for Java implemented by Google.
  * **JavaBinarySerializer** - binary serializer for Java using ObjectOutputStream.
  * **XmlStringSerializer** - internal Eneter XML serializer compatible with XmlSerializer from .NET.
<br />
The EneterProtoBufSerializer used the following message declared in the proto file:
```
message TestMessage
{
    required string Name = 1;
    required int32 Value = 2;
}
```

Binary and XML serializers used the same message but declared as the class.<br />
In .NET:
```
[Serializable]
public class TestMessage2
{
    public string Name;
    public int Value;
}
```
In Java:
```
public static class TestMessage2 implements Serializable
{
    private static final long serialVersionUID = -7462355711199438895L;
    public String Name;
    public int Value;
}
```

# Results #
Each serializer was used in the loop to serialize/deserialize the message 100 000 times. (_The test was performed on Intel Core2 Duo CPU 2.27 GHz 4GB RAM in 32bit mode. The source code of the unit-test is available in the source code of this project._)

| **.NET serializers** | **Time** | **Message Size** |
|:---------------------|:---------|:-----------------|
|EneterProtoBufSerializer|250 ms| 9 bytes |
|BinarySerializer| 2 294 ms| 221 bytes |
|XmlStringSerializer| 9 231 ms| 164 bytes |

| **Java serializers** | **Time** |
|:---------------------|:---------|
|EneterProtoBufSerializer|404 ms|
|JavaBinarySerializer| 2 466 ms |
|XmlStringSserializer| 3 317 ms |