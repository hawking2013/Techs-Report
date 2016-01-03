为什么要获取Java层的函数在Dalvik中对应的Method数据结构呢？

因为要Hook Java层的某个函数，首先得获取该函数在Dalvik中对应的Method数据结构，然后修改其函数指针指向另外一个函数，从而达到Hook的效果。

首先提出一个猜想，Java层通过反射可以获取函数的Method，而这个Method是否和Dalvik中的Method是一回事呢？

新建一个工程来验证一下（[工程下载地址](https://github.com/dingjikerbo/blog/blob/master/android%E7%83%AD%E4%BF%AE%E5%A4%8D%E6%8A%80%E6%9C%AF/files/MyApplication.tar.gz)）

分别通过三种方式来获取Method：通过反射、通过slot、通过函数签名。

首先新建一个类Test.class，如下

```
public class Test {

    static {
        System.loadLibrary("test");
    }

    public static void hello() {
        Log.i("bush", "hello called");
    }
    
    public native static void showMethodFromReflect(Method method);
    public native static void showMethodFromSlot(Class<?> clazz, int slot);
    public native static void showMethodFromSig(Class<?> clazz, String methodName, String methodSig);
}
```

Native层的实现如下：

```
#include <jni.h>
#include <stdlib.h>

#define ANDROID_SMP 0

#include "Dalvik.h"

JNIEXPORT jint
JNI_OnLoad(JavaVM *vm, void *reserved) {
	JNIEnv *env = NULL;
	jint result = -1;

	if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
		return result;
	}

	return JNI_VERSION_1_6;
}

void showMethodInfo(Method *method) {
	ALOGI("method: %p", method);
	ClassObject *classObject = method->clazz;
	ALOGD("clazz->descriptor: %s", classObject->descriptor);
	ALOGD("clazz->sourceFile: %s", classObject->sourceFile);
	ALOGD("name: %s", method->name);
	ALOGD("shorty: %s", method->shorty);
	ALOGD("accessFlags: %d", method->accessFlags);
}

extern "C" JNIEXPORT JNICALL void
Java_com_example_dingjikerbo_myapplication_Test_showMethodFromReflect(JNIEnv *env, jclass object, jobject methodObj) {
	Method* methodObject = (Method *) dvmDecodeIndirectRef(dvmThreadSelf(), methodObj);
	showMethodInfo(methodObject);
}

extern "C" JNIEXPORT JNICALL void
Java_com_example_dingjikerbo_myapplication_Test_showMethodFromSlot(JNIEnv *env, jclass clazz, jclass clazzToHook, jint slot) {
	ClassObject *clazzObject = (ClassObject *) dvmDecodeIndirectRef(dvmThreadSelf(), clazzToHook);
	Method *method = dvmSlotToMethod(clazzObject, slot);
	showMethodInfo(method);
}


extern "C" JNIEXPORT JNICALL void
Java_com_example_dingjikerbo_myapplication_Test_showMethodFromSig(JNIEnv *env, jclass object, jclass clazz, jstring methodName, jstring methodSig) {
	const char *_methodName = env->GetStringUTFChars(methodName, NULL);
	const char *_methodSig = env->GetStringUTFChars(methodSig, NULL);

	jmethodID methodId = env->GetMethodID(clazz, _methodName, _methodSig);
	if (methodId == NULL) {
		env->ExceptionClear();
		methodId = env->GetStaticMethodID(clazz, _methodName, _methodSig);
	}

	if (methodId != NULL) {
		Method *method = (Method *) methodId;
		showMethodInfo(method);
	}

	env->ReleaseStringUTFChars(methodName, _methodName);
	env->ReleaseStringUTFChars(methodSig, _methodSig);
}
```

MainActivity中调用如下：

```
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        showMethodFromReflect();
        showMethodFromSlot();
        showMethodFromSig();
    }

    private void showMethodFromReflect() {
        Method method = MethodUtils.getAccessibleMethod(Test.class, "hello");
        Test.showMethodFromReflect(method);
    }

    private void showMethodFromSlot() {
        Method method = MethodUtils.getAccessibleMethod(Test.class, "hello");
        Field field = FieldUtils.getField(Method.class, "slot", true);
        try {
            int slot = field.getInt(method);
            Test.showMethodFromSlot(Test.class, slot);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    private void showMethodFromSig() {
        Test.showMethodFromSig(Test.class, "hello", "()V");
    }
}
```

运行后日志如下：

```
method: 0x42d033d0
clazz->descriptor: Ljava/lang/reflect/Method;
clazz->sourceFile: Method.java
name: (null)
shorty: (null)
accessFlags: 0

method: 0x6d910c00
clazz->descriptor: Lcom/example/dingjikerbo/myapplication/Test;
clazz->sourceFile: Test.java
name: hello
shorty: V
accessFlags: 9

method: 0x6d910c00
clazz->descriptor: Lcom/example/dingjikerbo/myapplication/Test;
clazz->sourceFile: Test.java
name: hello
shorty: V
accessFlags: 9
```

从日志中可以看出，通过反射获取到的Method在Dalvik中对应的对象不是Method，所以强制转换成Method导致打出来的数据不对。
但是Class数据是没问题的，原因是Java中的对象在Dalvik中都是继承自Object的，而Object的第一个成员就是ClassObject，所以
任意两个Object相互转换，就算其余的成员都不一样，但是ClassObject总是对的上的。这个ClassObject就是描述该Object对应的类。

通过上面这个实验，有这么几个问题：
1. jmethodID和Method有什么关系？
2. 这个反射到的Method和Dalvik中的Method有什么联系？
3. slot是个什么东西？

先看第一个问题，在代码里找了一圈jmethodID，只找到如下两行

```C
struct _jmethodID;                      /* opaque structure */
typedef struct _jmethodID* jmethodID;   /* method IDs */
```

这个_jmethodID是个黑箱类型，里面怎么定义的外面是不知道的。那它和Method有什么关系呢？在dalvik/vm/jni.c中找到这么个函数

```Java
static jobject ToReflectedMethod(JNIEnv* env, jclass jcls, jmethodID methodID,
    jboolean isStatic)
{
    JNI_ENTER();
    ClassObject* clazz = (ClassObject*) dvmDecodeIndirectRef(env, jcls);
    Object* obj = dvmCreateReflectObjForMethod(clazz, (Method*) methodID);
    dvmReleaseTrackedAlloc(obj, NULL);
    jobject jobj = addLocalReference(env, obj);
    JNI_EXIT();
    return jobj;
}
```

可见，这里面可以直接将methodID转换成Method *，虽然这并不能说明两者就是完全相同的，但是我们可以认为_jmethodID的第一个成员就是Method。

再来看第二个问题，这个反射到的Method和Dalvik中的Method有什么联系？同样在dalvik/vm/jni.c中找到这么一个函数

```
/*
 * Given a java.lang.reflect.Method or .Constructor, return a methodID.
 */
static jmethodID FromReflectedMethod(JNIEnv* env, jobject jmethod)
{
    JNI_ENTER();
    jmethodID methodID;
    Object* method = dvmDecodeIndirectRef(env, jmethod);
    methodID = (jmethodID) dvmGetMethodFromReflectObj(method);
    JNI_EXIT();
    return methodID;
}
```


这个函数的作用就是将反射的Method对象转换成真正的Method对象。我们来看看它是怎么做到的

```
Method* dvmGetMethodFromReflectObj(Object* obj)
{
    ClassObject *clazz = (ClassObject*)dvmGetFieldObject(obj,
                                gDvm.offJavaLangReflectMethod_declClass);
    
    int slot = dvmGetFieldInt(obj, gDvm.offJavaLangReflectMethod_slot);

    return dvmSlotToMethod(clazz, slot);
}
```

原来它是通过slot来获取到的！！

那接下来的问题就很清楚了，搞定了slot一切都好说。看看dvmSlotToMethod的实现：

```
Method* dvmSlotToMethod(ClassObject* clazz, int slot)
{
    if (slot < 0) {
        slot = -(slot+1);
        assert(slot < clazz->directMethodCount);
        return &clazz->directMethods[slot];
    } else {
        assert(slot < clazz->virtualMethodCount);
        return &clazz->virtualMethods[slot];
    }
}
```

看起来，如果是virtualMethods，则slot >0，如果是正常函数，则slot < 0。

再来看看slot是在哪里设置的，仍然在dalvik/vm/reflect.c中：

```
static int methodToSlot(const Method* meth)
{
    ClassObject* clazz = meth->clazz;
    int slot;

    if (dvmIsDirectMethod(meth)) {
        slot = meth - clazz->directMethods;
        slot = -(slot+1);
    } else {
        slot = meth - clazz->virtualMethods;
    }

    return slot;
}
```

可见，这个slot就是Method在ClassObject的函数指针数组中的索引。

这样，三者都通了，一个是slot，一个是Java层反射的Method对象，一个是Dalvik中的Method。

总结一下，我们要Hook一个Java层的函数，则需要获取其在Dalvik中对应的Method。获取方式有两种：

1. 传入函数名和函数签名，然后在Native层调用getMethodID获取到jmethodID，但是函数签名得我们自己拼很麻烦。
2. 传入函数所在的Class和对应的slot，然后在Native层调用dvmSlotToMethod即可，slot可以通过在Java层反射来获取



