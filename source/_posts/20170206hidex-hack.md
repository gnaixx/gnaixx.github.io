---
title: DEX文件混淆加密
tags: [android, dex, hidex]
toc: true
date: 2017-02-06 23:24:44
categories: android
description: 现在部分 app 出于安全性(比如加密算法)或者用户体验(热补丁修复bug)会考虑将部分模块采用热加载的形式 Load。所以针对这部分的 dex 进行加密是有必要的，如果 dex 是修复的加密算法，你总不想被人一下就反编译出来吧。当然也可以直接用一个加密算法对 dex 进行加密，Load 前进行解密就可以了，但是最好的加密就是让人分不清你是否加密了。一般逆向过程中拿到一个可以直接反编译成 java 源码的 dex 我们很可能就认为这个 dex 文件是没有加密可以分析的。

---

## 0x00 前言
混淆加密主要是为了隐藏 dex 文件中关键的代码，力度从轻到重包括：静态变量的隐藏、函数的重复定义、函数的隐藏、以及整个类的隐藏。混淆后的 dex 文件依旧可以通过 dex2jar jade 等工具的反编译成 Java 源码，但是里面关键的代码已经看不到了。    
**java 效果图：**     
![效果图](/blog_images/20170206/compare-total.png)

**smali 效果图：**     
![效果图](/blog_images/20170206/compare-smali.png)

源码地址和使用说明在 github 上 **[hidex-hack](https://github.com/gnaixx/hidex-hack)**

## 0x01 dex格式分析
dex 文件格式在上一篇有进行了比较详细的介绍，具体可看**[dex文件格式分析](http://gnaixx.cc/2016/11/26/20161126dex-file/)**，这里简单的介绍一下整个 dex 文件的布局。  
   
**1.header(dex头部)**    
header 概述了整个 dex 文件的分布情况，包括了：**magic**, **checksum**, **signature**, **file_size**, **header_size**, **endian_tag**, **link**, **map**, **string_ids**, **type_ids**, **proto_ids**, **field_ids**, **method_ids**, **class_defs**, **data**。

- **checksum** 和 **signature** 是校验值，修改后需要对其进行修复
- **string_ids**, **type_ids**, **proto_ids**, **field_ids**, **method_ids** 作为类型数组节区（我瞎起的）保存了不同类型的值
- **class_defs** 存储了类的定义也是我们修改的重点
- **data** 是数据存储区，包括所有的数据

**2.类型数组节区**    
类型数组节区包括了**string_ids**, **type_ids**, **proto_ids**, **field_ids**, **method_ids**。分别表示：字符串，类型，函数签名，属性，函数。每个节区都保存了对应类型数据数组，可以用 010Editor 分析二进制文件数据。    
**属性示例：**     
![field_ids](/blog_images/20170206/field_ids.png)

**3.类定义**    
类定义是修改的重点，这里保存了所有类的结构，也是整个 dex 文件中结构最复杂的部分。其中包括了：静态属性变量、成员数形变量，虚函数，直接函数，静态函数等数据。

## 0x02 实现功能
通过分析 dex 文件格式，现在可以实现的混淆加密主要包括四种：

1. 静态变量隐藏
2. 函数重复定义
3. 函数隐藏
4. 类定义隐藏

四种混淆加密的实现方式都是通过修改 **class_def** 结构体中字段实现的。可以通过 json 格式了解一下 **class_def** 的结构（这里只列出来要用到的字段）：

```js
{
  "class_def": {
  	"class_idx": 01
  	"static_values_off": 000,       
  	"class_data_off": 001,          
  	"class_data": {                 
  		"direct_methods_size": 001,  
  		"virtual_methods_size": 002, 
  		"virtual_methods":[          
  			{
  				"code_off": 003 
  			},
  			{
  				"code_off": 004
  			}
  		]
  	}
  }
}
```
字段含义：

- class_idx: 类名序号，值是type_ids的一个index
- class_def: 类定义结构体
- static_values_off: 静态变量值偏移
- class_data_off: 类定义偏移
- class_data: 类定义结构体
- direct_methods_size: 直接函数个数
- virtual_methods_size: 虚函数个数
- virtual_methods: 虚函数结构体
- code_off: 函数代码偏移

通过上面的字段介绍其实很容易得到四个功能的实现方案，下面一个一个介绍。

### 1.静态变量隐藏
**static_vaules_off** 保存了每个类中静态变量的值的偏移量，指向 data 区里的一个列表，格式为 **encode_array_item**，如果没有此项内容，该值为0。所以要实现静态变量赋值隐藏只需要将 **static_values_off** 值修改为0。    
**实现效果：**    
![静态变量隐藏](/blog_images/20170206/compare-static.png)

_这里的静态数组数据没有成功隐藏，因为我也不知道怎么搞。😶_

### 2.函数重复定义
**class_def -> class_data -> virtual_methods -> code_ff** 表示的是某个类中某个函数的代码偏移地址。这里需要提到一个概念：Java 中所有函数实现都是虚函数，这一点和 C++ 是不一样的，所有这里修改的都是 **virtual_methods** 中 **code_off**。

![virtual_methods](/blog_images/20170206/virtual_methods.png)

实现方式:读取第一个函数的代码偏移地址，将接下来的函数偏移地址都修改为第一的值。

**实现效果:**  
![mutil-methods](/blog_images/20170206/compare-mutil-methods.png)

### 3.函数隐藏
**class_def -> class_data -> virtual_methods_size** 和 **class_def -> class_data -> direct_methods_size** 记录了类定义中函数的个数,如果没有定义函数则该值为0。所以只要将该值改为0，函数定义就会被隐藏。

**实现效果:**   
![hide-methods](/blog_images/20170206/compare-hide-methods.png)

### 4.类定义隐藏
**class_def -> class_data_off** 保存了具体类定义的偏移地址，也就是 **class_def -> class_data** 的地址，如果该值为0则所有实现将被隐藏。隐藏后会把类定义的所有东西都隐藏包括成员变量，成员函数，静态变量，静态函数。

**实现效果:**    
![hide-class](/blog_images/20170206/compare-class.png)

## 0x03 数据读取
上面一个章节主要介绍了功能实现的原理，接下来要介绍具体实现了。要实现修改 **class_def** 中字段，首先要把整个 dex 文件结构解析出来，当然可以只是我们需要的字段。在工具中我定义的 dex 结构如下，因为 **class_def** 结构比较复杂所以独立了一个包定义：    

```bash
→ tree -L 2
.
├── DexFile.java
├── FieldIds.java
├── Header.java
├── MapList.java
├── MethodIds.java
├── ProtoIds.java
├── StringIds.java
├── TypeIds.java
└── cladef
    ├── ClassData.java
    ├── ClassDefs.java
    ├── Code.java
    ├── EncodedField.java
    ├── EncodedMethod.java
    ├── EncodedValue.java
    └── StaticValues.java
```
也许你可能会疑问，我们功能实现时候只需要修改 **class_def** 为什么还需要读取 **string_ids** 这些区段。这是因为像上面提到的 **class_def -> class_idx** 保存的其实是 **type_ids** 中的序号，而 **type_ids** 中保存的是 **string_ids** 的序号。

为了灵活配置，运行工具的时候我们只需要配置好要隐藏的类名，比如需要隐藏某个类的实现 `hack_me_size: cc.gnaixx.samp.core.EntranceImpl`, 配置文件的具体实现下个章节介绍。

**DexFile.java** 定义了整个 dex 文件结构, 实现比较简单只有一个 **read(byte[] dexBuff)** 函数读取整个 dex 文件格式。

**DexFile.java:**

```java
public class DexFile {
    public static final int HEADER_LEN = 0x70;

    public Header header;
    public StringIds stringIds;
    public TypeIds typeIds;
    public ProtoIds protoIds;
    public FieldIds fieldIds;
    public MethodIds methodIds;
    public ClassDefs classDefs;
    public MapList mapList;

    //reader dex
    public void read(byte[] dexBuff){
        //read header
        byte[] headerbs = subdex(dexBuff, 0, HEADER_LEN);
        header = new Header(headerbs);

        //read string_ids
        stringIds = new StringIds(dexBuff, header.stringIdsOff, header.stringIdsSize);

        //read type_ids
        typeIds = new TypeIds(dexBuff, header.typeIdsOff, header.typeIdsSize);

        //read proto_ids
        protoIds = new ProtoIds(dexBuff, header.protoIdsOff, header.protoIdsSize);

        //read field_ids
        fieldIds = new FieldIds(dexBuff, header.fieldIdsOff, header.fieldIdsSize);

        //read method_ids
        methodIds = new MethodIds(dexBuff, header.methodIdsOff, header.methodIdsSize);

        //read class_defs
        classDefs = new ClassDefs(dexBuff, header.classDefsOff, header.classDefsSize);

        //read map_list
        mapList = new MapList(dexBuff, header.mapOff);
    }
}
```
第一步要先读取 **header** 因为它保存了其他节区的偏移地址和个数。

**Header.java:**

```java
public class Header {

    public byte[]  magic           = new byte[MAGIC_LEN];
    public int     checksum;
    public byte[]  signature       = new byte[SIGNATURE_LEN];
    public int     fileSize;
    public int     headerSize;
    public int     endianTag;
    public int     linkSize;
    public int     linkOff;
    public int     mapOff;
    public int     stringIdsSize;
    public int     stringIdsOff;
    public int     typeIdsSize;
    public int     typeIdsOff;
    public int     protoIdsSize;
    public int     protoIdsOff;
    public int     fieldIdsSize;
    public int     fieldIdsOff;
    public int     methodIdsSize;
    public int     methodIdsOff;
    public int     classDefsSize;
    public int     classDefsOff;
    public int     dataSize;
    public int     dataOff;

    public Header(byte[] headerBuff) {
        Reader reader = new Reader(headerBuff, 0);
        this.magic = reader.subdex(MAGIC_LEN);
        this.checksum = reader.readUint();
        this.signature = reader.subdex(SIGNATURE_LEN);
        //......
    }

    public void write(byte[] dexBuff){
        Writer writer = new Writer(dexBuff, 0);
        writer.replace(magic, MAGIC_LEN);
        writer.writeUint(checksum);
        writer.replace(signature, SIGNATURE_LEN);
        //.....
    }
}
```
知道了各个节区的偏移地址和个数接下来的读取就比较简单了，比如 **string_ids** 节区的读取。

**StringIds.java:**

```java
public class StringIds {

    class StringId {
        int dataOff;            //字符串偏移位置
        Uleb128 utf16Size;      //字符串长度
        byte data[];            //字符串数据

        public StringId(int dataOff, Uleb128 uleb128, byte[] data) {
            this.dataOff = dataOff;
            this.utf16Size = uleb128;
            this.data = data;
        }
    }

    StringId stringIds[];

    public StringIds(byte[] dexBuff, int off, int size) {
        this.stringIds = new StringId[size];

        Reader reader = new Reader(dexBuff, off);
        for (int i = 0; i < size; i++) {
            int dataOff = reader.readUint();
            Uleb128 utf16Size = getUleb128(dexBuff, dataOff);
            byte[] data = subdex(dexBuff, dataOff + 1, utf16Size.getVal());
            StringId stringId = new StringId(dataOff, utf16Size, data);
            stringIds[i] = stringId;
        }
    }

    public String getData(int id) {
        //return "(" + id + ")" + new String(stringIds[id].data);
        return new String(stringIds[id].data);
    }
}
```

其他节区的读取和 **string_ids** 类似，但是 **class_def** 节区结构比较复杂，读取起来可能比较麻烦。但是其实我们要用的值并不是很多，只需要关注那几个字段就好了。

**ClassDefs.java:**

```java
public class ClassDefs {

    public class ClassDef {
        public int          classIdx;       //class类型，对应type_ids
        public int          accessFlags;    //访问类型，enum
        public int          superclassIdx;  //supperclass类型，对应type_ids
        public int          interfacesOff;  //接口偏移，对应type_list
        public int          sourceFileIdx;  //源文件名，对应string_ids
        public int          annotationsOff; //class注解，位置位于data区，对应annotation_direcotry_item
        public HackPoint    classDataOff;   //class具体用到的数据，位于data区，格式为class_data_item,描述class的field,method,method执行代码
        public HackPoint    staticValueOff; //位于data区，格式为encoded_array_item

        public StaticValues staticValues;  // classDataOff不为0时存在
        public ClassData    classData;     // staticValueOff不为0存在

        public ClassDef(int classIdx, int accessFlags,
                        int superclassIdx, int interfacesOff,
                        int sourceFileidx, int annotationsOff,
                        HackPoint classDataOff, HackPoint staticValueOff) {
            this.classIdx = classIdx;
            this.accessFlags = accessFlags;
            this.superclassIdx = superclassIdx;
            this.interfacesOff = interfacesOff;
            this.sourceFileIdx = sourceFileidx;
            this.annotationsOff = annotationsOff;
            this.classDataOff = classDataOff;
            this.staticValueOff = staticValueOff;
        }

        public void setClassData(ClassData classData){
            this.classData = classData;
        }

        public void setStaticValue(StaticValues staticValues){
            this.staticValues = staticValues;
        }
    }

    int      offset; //偏移位置
    int      size;   //大小

    public ClassDef classDefs[];

    public ClassDefs(byte[] dexBuff, int off, int size) {
        this.offset = off;
        this.size = size;

        Reader reader = new Reader(dexBuff, off);
        classDefs = new ClassDef[size];
        for (int i = 0; i < size; i++) {
            int classIdx = reader.readUint();
            int accessFlags = reader.readUint();
            int superclassIdx = reader.readUint();
            int interfacesOff = reader.readUint();
            int sourcFileIdx = reader.readUint();
            int annotationOff = reader.readUint();

            HackPoint classDataOff = new HackPoint(HackPoint.UINT, reader.getOff(), reader.readUint());
            HackPoint staticValueOff = new HackPoint(HackPoint.UINT, reader.getOff(), reader.readUint());

            ClassDef classDef = new ClassDef(
                    classIdx, accessFlags,
                    superclassIdx, interfacesOff,
                    sourcFileIdx, annotationOff,
                    classDataOff, staticValueOff);

            if(staticValueOff.value != 0){
                Reader reader1 = new Reader(dexBuff, staticValueOff.value);
                Uleb128 staticSize = reader1.readUleb128();
                StaticValues staticValues = new StaticValues(staticSize);
                classDef.setStaticValue(staticValues);
            }

            if(classDataOff.value != 0){
                classDef.setClassData(new ClassData(dexBuff, classDataOff.value));
            }
            classDefs[i] = classDef;
        }
    }

    public void write(byte[] dexBuff){
        Writer writer = new Writer(dexBuff, offset);
        for(int i=0; i<size; i++){
            ClassDef classDef = classDefs[i];
            writer.writeUint(classDef.classIdx);
            writer.writeUint(classDef.accessFlags);
            writer.writeUint(classDef.superclassIdx);
            writer.writeUint(classDef.interfacesOff);
            writer.writeUint(classDef.sourceFileIdx);
            writer.writeUint(classDef.annotationsOff);

            writer.writeUint(classDef.classDataOff.value);
            if(classDef.classDataOff.value != 0){
                classDef.classData.write(dexBuff, classDef.classDataOff.value);
            }

            writer.writeUint(classDef.staticValueOff.value);
            if(classDef.staticValueOff.value != 0){
                //暂时不做处理
            }
        }
    }
}
```

这里需要介绍一下 dex 特有的一种数据类型 **LEB128** 官方介绍如下：
> LEB128 ("Little-Endian Base 128") is a variable-length encoding for arbitrary signed or unsigned integer quantities. The format was borrowed from the DWARF3 specification. In a .dex file, LEB128 is only ever used to encode 32-bit quantities.

> Each LEB128 encoded value consists of one to five bytes, which together represent a single 32-bit value. Each byte has its most significant bit set except for the final byte in the sequence, which has its most significant bit clear. The remaining seven bits of each byte are payload, with the least significant seven bits of the quantity in the first byte, the next seven in the second byte and so on. In the case of a signed LEB128 (sleb128), the most significant payload bit of the final byte in the sequence is sign-extended to produce the final value. In the unsigned case (uleb128), any bits not explicitly represented are interpreted as 0.

![leb128](/blog_images/20170206/leb128.png)

也就是说 LEB128 是基于 1 个 Byte 的一种不定长度的编码方式 。若第一个 Byte 的最高位为 1 ，则表示还需要下一个 Byte 来描述 ，直至最后一个 Byte 的最高 位为 0 。每个 Byte 的其余 Bit 用来表示数据。

代码中用 ULeb128.java(unsigned 无符号) 表示是该结构，通过分析Android源码 [Leb128.h](https://github.com/android/platform_dalvik/blob/master/libdex/Leb128.h)可以知道 LEB128 虽然表示的是不定长格式，但是在 Android 中只用到了4 个byte，所以只需要用int表示就可以了。

**ULeb128.java：**

```java
public class Uleb128 {
    byte[] realVal; //存储的byte数据
    int val; //表示的整型数据

    public Uleb128(byte[] realVal, int val){
        this.realVal = realVal;
        this.val = val;
    }

    public int getSize(){
        return this.realVal.length;
    }

    public int getVal(){
        return this.val;
    }

    public byte[] getRealVal(){
        return this.realVal;
    }
}
```
**Bytes to ULEB128:**

```java
//Reader.java
public Uleb128 readUleb128() {
        int value = 0;
        int count = 0;
        byte realVal[] = new byte[4];
        boolean flag = false;
        do {
            flag = false;
            byte seg = buffer[offset];
            if ((seg & 0x80) == 0x80) { //高8位为1
                flag = true;
            }
            seg = (byte) (seg & 0x7F);
            value += seg << (7 * count);
            realVal[count] = buffer[offset];
            count++;
            offset++;
        } while (flag);
        return new Uleb128(BufferUtil.subdex(realVal, 0, count), value);
    }
```

**Integer to ULEB128:**

```java
//Trans.java
public static Uleb128 intToUleb128(int val) {
        byte[] realVal = new byte[]{0x00, 0x00, 0x00, 0x00}; //int 最大长度为4
        int bk = val;
        int len = 0;
        for (int i = 0; i < realVal.length; i++) {
            len = i + 1; //最少长度为1
            realVal[i] = (byte) (val & 0x7F); //获取低7位的值
            if (val > (0x7F)) {
                realVal[i] |= 0x80; //高位为1 加上去
            }
            val = val >> 7;
            if (val <= 0) break;
        }
        Uleb128 uleb128 = new Uleb128(BufferUtil.subdex(realVal, 0, len), bk);
        return uleb128;
    }
```

## 0x04 HackPoint格式
**HackPoint** 表示修改后的数据结构,代码中把所有要修改的的字段都用 **HackPoint** 类型表示。**HackPoint** 类型有三个字段 type、offset、value，都是 int 类型分别表示：类型、偏移地址、原始值。类型主要有三种 **uint(unsigned int)、ushort(unsigned short 2byte)、uleb128**。这三种数据用 int 存储都足够了。   
**HackPoint.java:**

```java
public class HackPoint implements Cloneable {

    public static final int UINT = 0x01;
    public static final int USHORT = 0x02;
    public static final int ULEB128 = 0x03;

    public int type;        //数据类型
    public int offset;      //偏移地址
    public int value;       //原始值

    public HackPoint(int type, int offset, int val) {
        this.type = type;
        this.offset = offset;
        this.value = val;
    }

    @Override
    public HackPoint clone() {
        HackPoint hp = null;
        try {
            hp = (HackPoint) super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return hp;
    }
}
```

在修改完后会把所有的 **HackPoint** 数据写在 dex 文件的末尾。本来 dex 文件末尾是 **map_list** 区段，数据格式是 :

```c
struct map_list{
	ushort type;
	ushort unused;
	uint size;
	uint offset;
};
```
刚好是 12 byte, 所以 **HackPoint** 写入 dex 文件的格式为:
 
![hp](/blog_images/20170206/hp.png)

## 0x05 配置文件
配置文件的定义比较简单看一下示例就知道了：

```bash
####################################################
#    hack_class:      隐藏类定义
#    hack_sf_val:     隐藏静态变量
#    hack_me_size:    隐藏methods
#    hack_me_def:     重复函数定义（以第一个为准）
#####################################################


#隐藏静态变量值
hack_sf_val: cc.gnaixx.samp.core.EntranceImpl

#重复函数定义（以第一个为准）
hack_me_def: cc.gnaixx.samp.core.EntranceImpl

#隐藏函数实现
hack_me_size: cc.gnaixx.samp.core.EntranceImpl

#隐藏整个类实现
hack_class: cc.gnaixx.samp.core.EntranceImpl cc.gnaixx.samp.BuildConfig
```
_当多个类需要实现同一个功能的时候只需要用空格分隔就可以了_

配置文件读取代码：

```java
public static Map<String, List<String>> readConfig(String path) {
    try {
        Map<String, List<String>> config = new HashMap<>();
        FileReader fr = new FileReader(path);
        BufferedReader br = new BufferedReader(fr);
        String line;
        while ((line = br.readLine()) != null) {
            if (!line.startsWith("#") && !line.equals("")) {
                String conf[] = line.split(":");
                if (conf.length != 2) {
                    log("warning", "error config at :" + line);
                    System.exit(0);
                }

                String key = conf[0];
                String values[] = conf[1].split(" ");
                List<String> valueList = new ArrayList<>();
                for (int i = 0; i < values.length; i++) {
                    if (values[i] != null && !values[i].equals("")) {
                        valueList.add(values[i]);
                    }
                }
                config.put(key, valueList);
            }
        }
        fr.close();
        br.close();
        return config;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```
## 0x06 dex混淆隐藏
dex 文件混淆隐藏主要包括三个步骤：

1. 修改 **HackPoint** 并保存到 dex 文件末尾
2. 修复Header

### 1.修改 HackPoint

通过取得的配置文件中的配置类遍历 **class_def_item**：

```java
//查找配置文件所在类位置
private void seekHP(ClassDefs.ClassDef[] classDefItem, List<String> conf, String type, SeekCallBack callBack){
    if (conf == null) {
        return;
    }
    for (int i = 0; i < conf.size(); i++) {
        String classname = conf.get(i);
        boolean isDef = false;
        for (int j = 0; j < classDefItem.length; j++) {
            String className = dexFile.typeIds.getString(dexFile, classDefItem[j].classIdx); //查找顺序 class_idx => type_ids => string_ids
            className = pathToPackages(className); //获取类名
            if (className.equals(classname)) {
                callBack.doHack(classDefItem[j], this.hackPoints); //具体操作
                log(type, conf.get(i));
                isDef = true;
            }
        }
        if (isDef == false) {
            log("warning", "con't find class:" + classname);
        }
    }
}

//具体操作回调处理
interface SeekCallBack {
    void doHack(ClassDefs.ClassDef classDefItem, List<HackPoint> hackPoints);
}
```

**隐藏静态变量值:**

```java
//隐藏静态变量初始化
private void hackSfVal(ClassDefs.ClassDef[] classDefItem, List<String> conf) {
    seekHP(classDefItem, conf, Constants.HACK_SF_VAL, new SeekCallBack() {
        @Override
        public void doHack(ClassDefs.ClassDef classDefItem, List<HackPoint> hackPoints) {
            HackPoint point = classDefItem.staticValueOff.clone();  //获取静态变量数据偏移
            hackPoints.add(point);                          //添加修改点
            classDefItem.staticValueOff.value = 0;          //将静态变量的偏移改为0（隐藏赋值）
        }
    });
}
```

**函数重复定义:**

```java
//重复函数定义
private void hackMeDef(ClassDefs.ClassDef[] classDefItem, List<String> conf){
    seekHP(classDefItem, conf, Constants.HACK_ME_DEF, new SeekCallBack() {
        @Override
        public void doHack(ClassDefs.ClassDef classDefItem, List<HackPoint> hackPoints) {
            //以第一个为默认值
            int virtualMeSize = classDefItem.classData.virtualMethodsSize.value;
            int virtualMeCodeOff = 0;
            for (int i = 0; i < virtualMeSize; i++) {
                if (i == 0) {
                    virtualMeCodeOff = classDefItem.classData.virtualMethods[i].codeOff.value;
                }else{
                    HackPoint point = classDefItem.classData.virtualMethods[i].codeOff.clone();
                    hackPoints.add(point);
                    classDefItem.classData.virtualMethods[i].codeOff.value = virtualMeCodeOff;
                }
            }
        }
    });
}
```

**函数隐藏：**

```java
//隐藏函数定义
private void hackMeSize(ClassDefs.ClassDef[] classDefItem, List<String> conf){
    seekHP(classDefItem, conf, Constants.HACK_ME_SIZE, new SeekCallBack() {
        @Override
        public void doHack(ClassDefs.ClassDef classDefItem, List<HackPoint> hackPoints) {
            HackPoint directPoint = classDefItem.classData.directMethodsSize.clone(); //同时需改虚函数和直接函数
            HackPoint virtualPoint = classDefItem.classData.virtualMethodsSize.clone();
            hackPoints.add(directPoint);
            hackPoints.add(virtualPoint);
            classDefItem.classData.directMethodsSize.value = 0;
            classDefItem.classData.virtualMethodsSize.value = 0;
        }
    });
}
```

**隐藏类：**

```java
//隐藏静态变量初始化
private void hackSfVal(ClassDefs.ClassDef[] classDefItem, List<String> conf) {
    seekHP(classDefItem, conf, Constants.HACK_SF_VAL, new SeekCallBack() {
        @Override
        public void doHack(ClassDefs.ClassDef classDefItem, List<HackPoint> hackPoints) {
            HackPoint point = classDefItem.staticValueOff.clone();  //获取静态变量数据偏移
            hackPoints.add(point);                          //添加修改点
            classDefItem.staticValueOff.value = 0;          //将静态变量的偏移改为0（隐藏赋值）
        }
    });
}
```

**添加 HackPoint 数据到 dex 文件：**

```java
 //保留修改信息
private void appendHP() {
    byte[] pointsBuff = new byte[]{};
    for (int i = 0; i < hackPoints.size(); i++) {
        byte[] pointBuff = hackpToBin(hackPoints.get(i));
        pointsBuff = BufferUtil.append(pointsBuff, pointBuff, pointBuff.length);
    }
    dexBuff = BufferUtil.append(dexBuff, pointsBuff, pointsBuff.length);
}

//hackPoint 转 二进制
public static byte[] hackpToBin(HackPoint point) {
    ByteBuffer bb = ByteBuffer.allocate(4 * 3);
    bb.put(intToBin_Lit(point.type));
    bb.put(intToBin_Lit(point.offset));
    bb.put(intToBin_Lit(point.value));
    return bb.array();
}

//小端二进制
public static byte[] intToBin_Lit(int integer){
    byte[] bin = new byte[]{
            (byte) ((integer >> 0) & 0xFF),
            (byte) ((integer >> 8) & 0xFF),
            (byte) ((integer >> 16) & 0xFF),
            (byte) ((integer >> 24) & 0xFF)
    };
    return bin;
}
```
_dex 文件都是以小端数据保存_

### 2.修复Header
**Header** 中修复的数据有三个：

1. 文件长度 
2. checksum
3. signature

**修改代码:**

```java
//修改header
private void hackHeader() {
    //修改文件长度
    Header header = dexFile.header;
    header.fileSize = this.dexBuff.length;
    header.write(dexBuff); //需要先修改文件长度，才能计算signature checksum
    //修复 signature 校验
    log("old_signature", binToHex(dexFile.header.signature));
    byte[] signature = signature(dexBuff, SIGNATURE_LEN + SIGNATURE_OFF);
    header.signature = signature;
    log("new_signature", binToHex(signature));
    header.write(dexBuff); //需要先写sinature,才能计算checksum，凸
    //修复 checksum 校验
    log("old_checksum", intToHex(dexFile.header.checksum));
    int checksum = checksum_Lit(dexBuff, CHECKSUM_LEN + CHECKSUM_OFF);
    header.checksum = checksum;
    log("new_checksum", intToHex(checksum));
    header.write(dexBuff);
}

//计算signature
public static byte[] signature(byte[] data, int off) {
    int len = data.length - off;
    byte[] signature = SHA1(data, off, len);
    return signature;
}
//sha1算法
public static byte[] SHA1(byte[] decript, int off, int len) {
    try {
        MessageDigest digest = MessageDigest.getInstance("SHA-1");
        digest.update(decript, off, len);
        byte messageDigest[] = digest.digest();
        return messageDigest;
    } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
    }
    return null;
}

//计算checksum 值
public static int checksum_Lit(byte[] data, int off) {
    byte[] bin = checksum_bin(data, off);
    int value = 0;
    for (int i = 0; i < UINT_LEN; i++) {
        int seg = bin[i];
        if (seg < 0) {
            seg = 256 + seg;
        }
        value += seg << (8 * i);
    }
    return value;
}
//计算checksum
public static byte[] checksum_bin(byte[] data, int off) {
    int len = data.length - off;
    Adler32 adler32 = new Adler32();
    adler32.reset();
    adler32.update(data, off, len);
    long checksum = adler32.getValue();
    byte[] checksumbs = new byte[]{
            (byte) checksum,
            (byte) (checksum >> 8),
            (byte) (checksum >> 16),
            (byte) (checksum >> 24)};
    return checksumbs;
}
```

该部分代码地址: **[HidexHandle.java](https://github.com/gnaixx/hidex-hack/blob/master/hidex-tool/src/main/java/cc/gnaixx/tools/core/HidexHandle.java)**

## 0x07 dex还原
相对于加密解密过程简单了很多，只要根据 **HackPoint** 数据一一修复就好了。这里简单的说下修复步骤：

1. 读取 Header 中 **map_list** 的偏移地址和个数，因为 **HackPoint** 数据保存在 **map_list** 之后
2. 读取 **HackPoint** 数据并修复 dex 文件
3. 修复 Header 中的 **file_size、checksum、signature**

### java 实现
修复关键源码：    

```java
//修复dex文件
public byte[] redex() {
    int mapOff = getUint(dexBuff, MAP_OFF_OFF); //获取map_off
    int mapSize = getUint(dexBuff, mapOff); //获取map_size
    int hackInfoStart = mapOff + UINT_LEN + (mapSize * MAP_ITEM_LEN); //获取 hackinfo 开始地址
    int hackInfoLen = dexBuff.length - hackInfoStart; //获取hackinfo 长度
    hackInfoBuff = subdex(dexBuff, hackInfoStart, hackInfoLen); //获取hack数据
    int dexLen = dexBuff.length - hackInfoLen;
    dexBuff = subdex(dexBuff, 0, dexLen); //截取原始dex长度
    HackPoint[] hackPoints = Trans.binToHackP(hackInfoBuff);  //修复hack点
    for (int i = 0; i < hackPoints.length; i++) {
        log("hackPoint", JSON.toJSONString(hackPoints[i]));
        recovery(hackPoints[i]);
    }
    byte[] fileSize = intToBin_Lit(dexLen); //修复文件长度
    replace(dexBuff, fileSize, FILE_SIZE_OFF, UINT_LEN);
    byte[] signature = signature(dexBuff, SIGNATURE_LEN + SIGNATURE_OFF); //修复signature校验
    replace(dexBuff, signature, SIGNATURE_OFF, SIGNATURE_LEN);
    byte[] checksum = checksum_bin(dexBuff, CHECKSUM_LEN + CHECKSUM_OFF); //修复checksum校验
    replace(dexBuff, checksum, CHECKSUM_OFF, CHECKSUM_LEN);
    log("fileSize", dexLen);
    log("signature", binToHex(signature));
    log("checksum", binToHex_Lit(checksum));
    return this.dexBuff;
}
//还原原始值
private void recovery(HackPoint hackPoint) {
    Writer writer = new Writer(this.dexBuff, hackPoint.offset);
    if (hackPoint.type == HackPoint.USHORT) {
        writer.writeUshort(hackPoint.value);
    }
    else if (hackPoint.type == HackPoint.UINT) {
        writer.writeUint(hackPoint.value);
    }
    else if (hackPoint.type == HackPoint.ULEB128) {
        Uleb128 uleb128 = Trans.intToUleb128(hackPoint.value);
        writer.writeUleb128(uleb128);
    }
}
```

### c++ 实现
工具本身就是为了实现安全加固，那么用 java 实现意义就小了很多，所以工具包里面的实现我是用 NDK 开发的。

修复关键源码：

```java
//解密dex
void recode(char* source, uint sourceLen, char* target, uint* targetLen){
    uint mapOff = readUint(source, MAP_OFF_OFF); //获取map_off
    uint mapSize = readUint(source, mapOff); //获取map_size
    LOGD("mapInfo: {map_off:%d, map_size:%d}", mapOff, mapSize);

    uint hackInfoOff = mapOff + UINT_LEN + (mapSize * MAP_ITEM_LEN); //定位hackInfo位置
    uint hackInfoLen = sourceLen - hackInfoOff; //hackInfo长度
    char* hackInfo = (char *) calloc(hackInfoLen, sizeof(char));
    memcpy(hackInfo, source + hackInfoOff, hackInfoLen); //复制hackInfo
    LOGD("hackInfo: {hackInfo_off:%d, hackInfo_len}", hackInfoOff, hackInfoLen);

    uint hackPointSize = hackInfoLen / sizeof(HackPoint); //获取hackPoint结构体
    HackPoint* hackPoints = (HackPoint *) calloc(hackPointSize, sizeof(HackPoint));
    initHP(hackPoints, hackInfo, hackPointSize); //将hockInfo 转化为结构体

    *targetLen = hackInfoOff;
    memcpy(target, source, *targetLen); //恢复原始长度

    //恢复数据
    for(int i=0; i<hackPointSize; i++){
        recoverHP(target, hackPoints[i]);
    }
    LOGD("Recover HackPoint success");

    //修复hearder
    recoverHeader(target, *targetLen);

    free(hackInfo);
    free(hackPoints);
}
```

完整源码地址: **[hidex.cpp](https://github.com/gnaixx/hidex-hack/blob/master/hidex-libs/src/main/cpp/hidex.cpp)**

## 0x09 总结
整体功能还是比较简单，实现的代码也不是很复杂，但是这些都需要基于对 dex 文件格式的了解的前提下。     
另外该工具存在一个缺点，dex 的加载问题。Android中加载 dex 的 DexClassLoad 只支持文件路径加载，不像 java 中的 ClassLoad 可以支持二进制流加载，所以在加载 dex 是就存在加密后的 dex 缓存，这是非常危险的。所以下个研究的点也就是自定义 DexClassLoad 实现不落地加载。（很多安全加固厂商老早就实现了🙄）。     
虽然功能不算强大，也有不少缺点，不过也花了自己不少时间研究，对 dex 文件格式也有点了解，也算值得了。