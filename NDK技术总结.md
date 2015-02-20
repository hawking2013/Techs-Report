#NDK技术总结

###一. Java层向native层传数组
Java向native传int数组，native中类型为jintArray，拿到数组metaData后，若要向该数组赋值，则需要做如下处理

```C
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

###二. native层向Java层抛异常

```C
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

###三. native指针和Java层互传
native指针和Java层互传，首先Java层是没有指针的概念的，那么native指针只能以long的形式传回Java，当需要时，Java层再将该
long传回native。不过要注意的是这里涉及long和指针的转换，要先通过intptr_t作类型中转，再转换成long或相应类型的指针，如下
```C
typedef struct student {
	char buf[120];
	int age;
} student;

JNIEXPORT jlong JNICALL native_pointer(JNIEnv *env, jclass clazz) {
	student *student = malloc(sizeof(student));
	student->age = 26;
	strcpy(student->buf, "frank");
	return (jlong) (intptr_t) student;
}

JNIEXPORT jstring JNICALL native_get(JNIEnv *env, jclass clazz, jlong ptr) {
	student *stu = (student *)(intptr_t) ptr;
	return (*env)->NewStringUTF(env, stu->buf);
}
```

```Java
public class JniHello {
	
	static {
		System.loadLibrary("hello");
	}
	
	private static volatile long mPtr;
	
	public static native long native_pointer();
	public static native String native_get(long ptr);
	
	public static long start() {
		mPtr = native_pointer();
		return mPtr;
	}
	
	public static String getContent() {
		return native_get(mPtr);
	}
}
```
