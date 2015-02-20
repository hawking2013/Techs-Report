#NDK技术总结

一. Java层向native层传数组
Java向native传int数组，native中类型为jintArray，拿到数组metaData后，若要向该数组赋值，则需要做如下处理

```Java
jint* const ints = (*env)->GetIntArrayElements(env, metaData, 0);
ints[0] = width;
ints[1] = height;
ints[2] = ImageCount;
ints[3] = errorCode;
(*env)->ReleaseIntArrayElements(env, metaData, ints, 0);
```

使用 GetIntArrayElements方法获取指向metaData数组元素的指针，注意该函数的参数，第一个是JNIEnv，第二个是数组，第三个是数组里面开始的元素。之后对数组的使用就和普通的c中的数组使用没有什么不同了。用完后提醒JVM回收metaData数组元素的引用。

这里举的例子是使用int数组的，同样还有boolean、float等对应的数组。
获取数组元素指针的对应关系：

```Java
函数　　　　　　　　　　　　数组类型
GetBooleanArrayElements　　 boolean
GetByteArrayElements　　　　byte
GetCharArrayElements　　 　 char
GetShortArrayElements　　　 short
GetIntArrayElements　　　　 int
GetLongArrayElements　　　　long
GetFloatArrayElements　 　　float
GetDoubleArrayElements　　　double
```
释放数组元素指针的对应关系：
```Java
函数　　　　　　　　　　　　数组类型
ReleaseBooleanArrayElements boolean
ReleaseByteArrayElements　　byte
ReleaseCharArrayElements　　char
ReleaseShortArrayElements　 short
ReleaseIntArrayElements　　 int
ReleaseLongArrayElements　　long
ReleaseFloatArrayElements　 float
ReleaseDoubleArrayElements　double
```

二. native层向Java层抛异常

```Java
jclass exClass = (*env)->FindClass(env, "com/dingjikerbo/inuker/ui/view/gif/GifIOException");
jmethodID mid = (*env)->GetMethodID(env, exClass, "<init>", "(I)V");
jobject exception = (*env)->NewObject(env, exClass, mid, errorCode);
(*env)->Throw(env, exception);
```

这里要抛出GifIOException，所以在Java中要定义好该异常，并注意混淆时要排除该类，并及时catch住。
这里逻辑很简单，首先找到Java层异常类，然后调用构造函数新建一个异常对象，最后抛出异常。

注意，构造函数名为<init>，签名中返回值为V，表示void。NewObject中传入构造函数id及对应参数。

另外，native函数要带上抛异常声明
```Java
static native long openFile(int[] metaData, String filePath, boolean justDecodeMetaData) throws GifIOException;
```
