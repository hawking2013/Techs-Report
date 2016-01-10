Dalvik虚拟机反射调用函数执行流程
----------------------------------

Dalvik虚拟机函数执行有以下几种方式：

1. interpreted 调用 interpreted
2. interpreted 调用 native
3. native 调用 interpreted
4. native 调用 native
5. 反射调用 interpreted
6. 反射调用 native

本章主要分析反射调用。新建一个工程如下：

```
 public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Method method = MethodUtils.getMatchingAccessibleMethod(MainActivity.class, "test", String.class);

        try {
            String result = (String) method.invoke(MainActivity.class, "hello");
            Toast.makeText(this, result, Toast.LENGTH_LONG).show();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }

    public static String test(String name) {
        return "invoked: " + name;
    }
}
```

这个Method的invoke是调到哪里了？

```
public Object invoke(Object receiver, Object... args)
            throws IllegalAccessException, IllegalArgumentException, InvocationTargetException {
        if (args == null) {
            args = EmptyArray.OBJECT;
        }
        return invokeNative(receiver, args, declaringClass, parameterTypes, returnType, slot, flag);
    }

private native Object invokeNative(Object obj, Object[] args, Class<?> declaringClass,
        Class<?>[] parameterTypes, Class<?> returnType, int slot, boolean noAccessCheck)
                throws IllegalAccessException, IllegalArgumentException,
                        InvocationTargetException;
```

可见是调到invokeNative了，这是个native函数，具体实现如下：

```
static void Dalvik_java_lang_reflect_Method_invokeNative(const u4* args, JValue* pResult)
{
    meth = dvmSlotToMethod(declaringClass, slot);
    
    ..................
    
    result = dvmInvokeMethod(methObj, meth, argList, params, returnType, noAccessCheck);

    RETURN_PTR(result);
}
```

可见这里是调用了dvmInvokeMethod来执行反射的函数的。我们再来看dvmInvokeMethod的实现：

```
Object* dvmInvokeMethod(Object* obj, const Method* method,
    ArrayObject* argList, ArrayObject* params, ClassObject* returnType,
    bool noAccessCheck)
{
    clazz = callPrep(self, method, obj, !noAccessCheck);

    ins = ((s4*)self->curFrame) + (method->registersSize - method->insSize);
    verifyCount = 0;

    if (!dvmIsStaticMethod(method)) {
        *ins++ = (s4) obj;
    }

    if (dvmIsNativeMethod(method)) {
        (*method->nativeFunc)(self->curFrame, &retval, method, self);
    } else {
        dvmInterpret(self, method, &retval);
    }

    dvmPopFrame(self);

    return retObj;
}
```

可见这里的实现和dvmCallMethodV非常类似，就不再赘述了。

总结一下，通过反射调用一个函数，首先要获取Dalvik虚拟机中该函数的Method对象，然后要为这个函数开辟一块栈帧，如果该函数不是native的，则将pc指向该函数的指令码数组，然后在循环中依次取指令执行，直到函数返回，再恢复栈帧和pc。如果该函数是native的，则直接跳转到该nativeFunc执行，完后恢复栈帧即可。

