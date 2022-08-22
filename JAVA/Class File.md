A class file consists of a stream of 8-bit bytes. 

All 16-bit, 32-bit, and 64-bit quantities are constructed by reading in two, four, and eight consecutive 8-bit bytes, respectively. 

Multibyte data items are always stored in big-endian order, where the high bytes come first.

In the Java SE platform, this format is supported by interfaces java.io.DataInput and java.io.DataOutput and classes such as java.io.DataInputStream and java.io.DataOutputStream.

---

클래스 파일은 byte stream으로 구성되어있다.

모든 16비트, 32비트, 64비트들은  연속적인 2바이트, 4바이트 8바이트로 읽어진다.

멀티바이트로 이루어진 데이터들은 항상 big-endian에 따라 저장된다. 포맷은 다음과 같다. (역어셈블 과정으로 읽기 쉽게 변환된 것이 아닌 class 파일 자체의 바이너리 형태 포맷) 

```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

##### 1. 데이터 형태에 따른 구조  
- u4 -> 4바이트로 이루어진 데이터  
- u2 -> 2바이트로 이루어진 데이터

```
cp_info {
    u1 tag;
    u1 info[];
}

```
jvm의 명령어는 런타임에 클래스, 인터페이스 등의 layout에 의존하지 않고 constant_pool에 존재하는 symbolic information에 의존한다.

tag : constant_pool에 존재하는 데이터가 어떠한 유형인지 나타내는 필드이다. tag 뒤에는 2바이트 이상의 바이트로 해당 상수에 대한 구체적인 정보를 나타내며, 태그값에 따라 뒤에 오는 포맷이 달라지게 된다. 

|Constant Type|Value|
|---|---|
|CONSTANT_Class|7|
|CONSTANT_Fieldref|9|
|CONSTANT_Methodref|10|
|CONSTANT_InterfaceMethodref|11|
|CONSTANT_String|8|
|CONSTANT_Integer|3|
|CONSTANT_Float|4|
|CONSTANT_Long|5|
|CONSTANT_Double|6|
|CONSTANT_NameAndType|12|
|CONSTANT_Utf8|1|
|CONSTANT_MethodHandle|15|
|CONSTANT_MethodType|16|
|CONSTANT_InvokeDynamic|18|

```
CONSTANT_Class_info {
    u1 tag;         
    u2 name_index;
}
```

```
field_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

```
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

```
attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```

---
### Byte Ordering 

Big-Endian : 상위 바이트를 메모리의 하위 번지수에 기록한다.

little-Endian : 상위 바이트를 메모리의 상위 번지 수에 저장한다.

예를 들어 특정 메모리 공간에 임의의 16진수 0x12345678을 저장해야된다면, (이 때 데이터는 연속적인 메모리 공간에 저장되고, 메모리의 시작 주소는 100 이라 가정하자)

빅 엔디안 방식에는  100번지 : 12,   101번지 34,   102번지 56,   103번지 78


리틀 엔디안 방식에는  100번지 : 78,   101번지 56,   102번지 34,   103번지 12의 형태로 데이터를 저장하게 된다. 
