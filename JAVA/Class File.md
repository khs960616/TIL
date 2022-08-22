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
클래스 또는 인터페이스의 정보를 나타내는 데 쓰인다. (tag는 CONSTANT_Class인 7값을 가진다)

name_index는 constant_pool 테이블에 존재하는 엔트리의 인덱스를 가진다. 

또한 해당 인덱스는 클래스 또는 인터페이스의 이름을 나타내는 CONSTANT_Utf8_info 타입의 엔트리여야 한다.


```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

상수의 String value를 나타내는데 쓰인다. (tag값은 표에 명시된 것처럼 1을 가진다)

length : bytes array의 길이를 나타낸다. (이 때, 해당 length는 string의 길이를 나타내는 것이 아니다.)

bytes[] : 나타내는 string의 바이트를 포함한다. (각 바이트는 0, 0xf0 ~ 0xff의 값을 가질 수 없다는 제약조건이 있다고 함) 

문자열은 utf-8형태로 인코딩되며, 각 문자열은 non-null 아스키코드을 포함하지 않는다.

```
CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
```
클래스의 field 또는 method를 나타내는데 쓰이며 어떤 클래스 (또는 인터페이스)를 나타내는지에 대한 정보를 가지고 있지는 않다. 

name_index : 상수풀의 엔트리에서 참조하고 있는 index를 나타낸다. 해당 index는 반드시 CONSTANT_Utf8_info 타입의 엔트리여야 한다. 해당 엔트리는 필드 또는 메서드명을 나타낸다. 

name_and_type_index: 상수풀의 엔트리에서 참조하고 있는 index를 나타낸다. 해당 index는 반드시 CONSTANT_Utf8_info 타입의 엔트리여야 한다. 

필드 또는 엔트리에 대한 Descriptors를 나타낸다. 

```
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_InterfaceMethodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

class_index : 상수풀의 엔트리에서 참조하고 있는 index를 나타낸다. 해당 index는 반드시 CONSTANT_Class_info 타입의 엔트리여야 한다. 

- CONSTANT_Fieldref_info : 클래스 타입 또는 인터페이스 타입 두 가지 모두 가능하다.
- CONSTANT_Methodref_info : 반드시 클래스 타입에 대한 참조여야한다.
- CONSTANT_InterfaceMethodref_info: 반드시 인터페이스 타입에 대한 참조여야한다. 


name_and_type_index : 상수풀의 엔트리에서 참조하고 있는 index를 나타낸다. 해당 index는 반드시 CONSTANT_NameAndType_info 타입이여야한다.

해당 엔트리가 가르키는 index는 필드 또는 메서드에 이름과 descriptor에 대한 정보를 가지고 있다. 


---

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
