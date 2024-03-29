#Android JNI技术总结

首先介绍`JNI`函数注册框架，之后再描述各技术细节

在Java中调`System.loadLibrary("hello")`时会调用native层的`JNI_OnLoad`，注册函数可以放在这里进行
```C
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved)
{
	JNIEnv* env = NULL;
	jint result = JNI_VERSION_1_4;

	if ((*vm)->GetEnv(vm, (void**) &env, JNI_VERSION_1_4) != JNI_OK) {
		return -1;
	}

	if (!registerNatives(env)) {//注册
		return -1;
	}

	return result;
}

// Java中Jni类完整类名
char *gClassName = "com/example/jnihello/JniHello";

// 要注册的native函数
JNINativeMethod gMethods[] = {
	{ "native_add", "(II)I", native_add },
	{ "native_string", "()Ljava/lang/String;", native_string },
	{ "native_pointer", "()J", native_pointer },
	{ "native_get", "(J)Ljava/lang/String;", native_get }
};

int registerNatives(JNIEnv* env) {
	if (!registerNativeMethods(env, gClassName, gMethods, NELEMS(gMethods))) {
		return JNI_FALSE;
	}

	return JNI_TRUE;
}

int registerNativeMethods(JNIEnv* env, const char* className,
        JNINativeMethod* gMethods, int numMethods)
{
	jclass clazz;
	clazz = (*env)->FindClass(env, className);
	if (clazz == NULL) {
		return JNI_FALSE;
	}
	if ((*env)->RegisterNatives(env, clazz, gMethods, numMethods) < 0) {
		return JNI_FALSE;
	}

	return JNI_TRUE;
}
```

###一. JNI调用Java对象
JNI提供的功能之一是在本地代码中使用Java对象。包括：创建一个java类对象和通过函数传递一个java对象。

创建一个java类对象，首先需要得到得到使用`FindClass/GetObjectClass`函数得到该类，然后使用`GetMethodID`方法得到该类的方法id,然后调用该函数。

Java 和 Native 代码之间函数调用时，如果是简单类型，也就是内置类型，比如 int, char 等是值传递（pass by value），而其它 Java 对象都是引用传递（pass by reference），这些对象引用由 JVM 传给 Native 代码。

在本地方法中调用Java对象的方法的步骤：

1）获取你需要访问的Java对象的类

`FindClass`通过传java中完整的类名来查找java的class

`GetObjectClass`通过传入jni中的一个java的引用来获取该引用的类型。

他们之间的区别是，前者要求你必须知道完整的类名，后者要求在Jni有一个类的引用。

2）获取MethodID,调用方法

`GetMethodID` 得到一个实例的方法的ID

`GetStaticMethodID` 得到一个静态方法的ID

3)获取对象的属性

`GetFieldID` 得到一个实例的域的ID

`GetStaticFieldID` 得到一个静态的域的ID

###二. JNI中引用的Java对象的生命周期
Java对象作为引用被传递到native层，这些对象引用都有其生命周期。生命周期分为:全局引用，局部引用、弱全局引用。

1、`Local Reference` 局部引用

函数调用时传入jobject或者jni函数创建的jobejct，都是局部引用。

其特点就是一旦JNI层函数返回，jobject就被垃圾回收掉，所以需要注意其生命周期。可以强制调用DeleteLocalRef进行立即回收。
```C
jstring pathStr = env->NewStringUTF(path)
....
env->DeleteLocalRef(pathStr);
```

2、`Global Reference` 全局引用

这种对象如不主动释放，它永远都不会被垃圾回收

创建： `env->NewGlobalRef(obj)`

释放： `env->DeleteGlobalRef(obj)`

若要在某个 Native 代码返回后，还希望能继续使用 JVM 提供的参数, 或者是过程中调用 JNI 函数的返回值（比如 g_mid）， 则将该对象设为 global reference，以后只能使用这个 global reference；若不是一个 jobject，则无需这么做。

3、`Weak Global Reference` 弱全局引用

一种特殊的 Global Reference ,在运行过程中可能被垃圾回收掉，所以使用时请务必注意其生命周期及随时可能被垃圾回收掉,比如内存不足时。使用前可以利用JNIEnv的 IsSameObject 进行判定它是否被回收
`(*env)->IsSameObject(env, wobj, NULL)`

###三. 本地线程中调用java对象
问题1：

JNIEnv是一个线程相关的变量

JNIEnv 对于每个 thread 而言是唯一的

JNIEnv *env指针不可以为多个线程共用

解决办法：

但是java虚拟机的JavaVM指针是整个jvm公用的，我们可以通过JavaVM来得到当前线程的JNIEnv指针.

可以使用javaAttachThread保证取得当前线程的Jni环境变量
```C
static JavaVM *gs_jvm=NULL;
gs_jvm->AttachCurrentThread((void **)&env, NULL);//附加当前线程到一个Java虚拟机
jclass cls = env->GetObjectClass(gs_object);
jfieldID fieldPtr = env->GetFieldID(cls,"value","I");
```

问题2：

不能直接保存一个线程中的jobject指针到全局变量中,然后在另外一个线程中使用它。

解决办法：

用env->NewGlobalRef创建一个全局变量，将传入的obj(局部变量)保存到全局变量中,其他线程可以使用这个全局变量来操纵这个java对象

注意：若不是一个 jobject，则不需要这么做。如：

jclass 是由 jobject public 继承而来的子类，所以它当然是一个 jobject，需要创建一个 global reference 以便日后使用。

而 jmethodID/jfieldID 与 jobject 没有继承关系，它不是一个jobject，只是个整数，所以不存在被释放与否的问题，可保存后直接使用。
```C
static jobject gs_object=NULL;
JNIEXPORT void JNICALL Java_Test_setEnev(JNIEnv *env, jobject obj)
{
	env->GetJavaVM(&gs_jvm); //保存到全局变量中JVM
	//直接赋值obj到全局变量是不行的,应该调用以下函数:
	gs_object=env->NewGlobalRef(obj);
}
```

###三. Java层向native层传数组
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

###四. native层向Java层抛异常

```C
jclass exClass = (*env)->FindClass(env, "com/dingjikerbo/inuker/ui/view/gif/GifIOException");
jmethodID mid = (*env)->GetMethodID(env, exClass, "<init>", "(I)V");
jobject exception = (*env)->NewObject(env, exClass, mid, errorCode);
(*env)->Throw(env, exception);
```

这里要抛出`GifIOException`，所以在Java中要定义好该异常，并注意混淆时要排除该类，并及时catch住。
这里逻辑很简单，首先找到Java层异常类，然后调用构造函数新建一个异常对象，最后抛出异常。

注意，构造函数名为`<init>`，签名中返回值为V，表示void。NewObject中传入构造函数id及对应参数。

另外，native函数要带上抛异常声明
```Java
static native long openFile(int[] metaData, String filePath, boolean justDecodeMetaData) throws GifIOException;
```

###五. native指针和Java层互传
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

