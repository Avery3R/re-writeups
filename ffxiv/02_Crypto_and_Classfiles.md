### Intro

Looking through the strings I found a lot of references to a JVM. Looking further into strings related to class loading such as `class file load count max over!\n`, `class info error`, and `class name too long` led me to the function which loads classes. The `*.clb` files within archives are a container for java classes and have optional crypto.

### Classfile Parsing

The function which parses class files is located at `0x00BEC3C0` and I am calling it `ParseCLBFile`. CLB files are actually very very similar to oracle/sun jvm classfiles. The major differences are that CLB files are little-endian and have a fixed-size header. Please refer to the [official JVM documentation on classfiles](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.1) as I will be referring to the names of the fields in the ClassFile structure heavily.

#### Magic

There are three different code paths relating to the first four bytes of the file.

* `'SSSS'` - Normal CLB file
* `'TTTT'` - Encrypted CLB file
* Anything else - Assumed to be an Oracle/Sun classfile. These are parsed by converting them to CLB files internally.

#### Major and minor version

These directly follow the magic, as in the official jvm classfile, and are never encrypted.

#### Crypto

Everything beyond the major and minor version can optionally be encrypted. The code for crypto is really messy within hex-rays because the executable is 32-bit and the crypto code makes heavy use of 64-bit numbers. For example, take a look at how the `key` variable is always indexed `2*x` and `2*x+1` in hex-rays' decompilation of this function.

![messy crypto code](http://i.imgur.com/ScpLM0U.png)

Forcing the type of the `key` variable to `unsigned __int64*` only causes hex-rays to cast it to a `DWORD*`. I don't really want to get into the internals of how the crypto works, partially because it's messy, and partially because I don't understand it. However, a high level overview of the algorithm is:

* Block cipher
    * 64-bit blocks
    * 64-bit key
        * Expanded to 32 64-bit numbers internally.
* Two-step
    * S-box
    * Whatever it's doing in that screenshot above
* Built-in checksum that's appended to the end of the encrypted data

This same algorithm is also used for the encryption and decryption of savegames. Partially manually decompiled and slightly cleaned-up decryption code is included in the class converter down below. The key used for the crypto of CLB files are the first four bytes of the file, so `'TTTT'` and then the major and minor versions.

After decryption the magic is changed to `'SSSS'` in the internal CLBFile structure.

#### Fixed-size header

Unlike official JVM classfiles, CLB files have a fixed-size header. General structure is a bunch of field counts, and then a bunch of pointers within the file to field data. The fact that the `ParseCLBFile` function has code to convert java classfiles to clb files helped greatly with figuring out this structure.

```
struct CLBFile
{
  int magic;
  __int16 minor_version;
  __int16 major_version;
  __int16 constant_pool_count;
  __int16 access_flags;
  __int16 interfaces_count;
  __int16 fields_count;
  __int16 methods_count;
  __int16 reserved1;
  __int16 this_classname_hash;
  __int16 reserved2;
  constant_pool_info *pConstant_pool_info;
  constant_pool_info *pThis_class;
  void *pThis_classname;
  constant_pool_info *pSuper_class;
  void *pInterface_info;
  field_info *pField_info;
  method_info *pMethod_info;
  void *zero;
};
```

#### Constant Pool

In official jvm classfiles constant pool entries are of a variable size depending on the type(`tag`). They are of a fixed size within CLB files.

```
struct constant_pool_info
{
  int tag;
  int reserved;
  int tagInfoA;
  int tagInfoB;
};
```

The `tag` field matches up with the offical jvm, and `reserved` is always zero. `tagInfoA` and `tagInfoB` depend on the tag type. For the full details please see the parsing code in the class converter below.

#### Attributes

The one remaining major difference between official jvm classfiles and CLB files are how attributes are encoded. CLB files only allows two different attributes, `Code` attributes and `ConstValue` attributes. Methods are only allowed to have `Code` attributes, and fields are only allowed to have `ConstValue` attributes. Nothing else is allowed to have any type of attribute.

### Class converter code

I hacked together some code to convert `*.clb` files to jvm class files so I could throw them in existing java decompilers. As before, the code is mostly terrible and it's all in one source file: http://pastebin.com/XkVh90Bt.

### Interesting classes

The game's main class is `WhiteResident` and is located at `sys/script/WhiteResident.clb`. The main function of the game is called `WhiteMain`. A decompiled version is available here: http://pastebin.com/qwG8aPB2. All classes within the `fake` package are almost 100% jni bindings. However, there are a few helper functions that contain actual java bytecode.

### Next steps

If you looked within the decompiled `WhiteResident` you may have noticed some debug menu options such as `i = Window.askChoice(str1, "FFXIII起動|ロケセレクタ|フィールドテスト|バトルテスト|ロールテスト|イベントテスト|ムービーテスト|キャラテスト|モーションテスト|CrystalToolsと通信|オートキャプチャ|オプション");`. I'm going to patch the binary so that `White.isUserBoot()` returns false and try to access these options. Spoiler: You totally can. I'll write up the post about it in a day or two when I'm feeling less lazy.