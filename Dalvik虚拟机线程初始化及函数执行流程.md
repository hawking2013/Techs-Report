Dalvik虚拟机线程初始化及函数执行流程
---------------------------------------------------

研究dalvik线程创建，只因为想搞清楚dalvik函数执行流程，那么这个流程的开端在哪里呢？我想到了线程，因为每个线程都有自己的Runnable，而这个入口一定就是Runnable开始执行的地方。带着这个疑问，我展开了一系列研究。

首先，Thread开始执行时，要调用Thread.start()，如下


```Java
public synchronized void start() {
    VMThread.create(this, stackSize);
}
```

这个VMThread什么都没做，只是定义了一堆native函数，如下

```Java
class VMThread {
    Thread thread;
    int vmData;

    VMThread(Thread t) {
        thread = t;
    }

    native static void create(Thread t, long stackSize);

    static native Thread currentThread();
    static native boolean interrupted();
    static native void sleep (long msec, int nsec) throws InterruptedException;
    static native void yield();

    native void interrupt();

    native boolean isInterrupted();

    void start(long stackSize) {
        VMThread.create(thread, stackSize);
    }
}
```

我们再看create的具体实现：

```Java
static void Dalvik_java_lang_VMThread_create(const u4* args, JValue* pResult) {
    dvmCreateInterpThread(threadObj, (int) stackSize);
}
```

dvmCreateInterpThread代码较长，我们只关注重点部分

```Java
bool dvmCreateInterpThread(Object* threadObj, int reqStackSize)
{
    newThread = allocThread(stackSize);
    
    pthread_create(&threadHandle, &threadAttr, interpThreadStart, newThread);
}
```

可见，这个函数主要做两件事，一个是调用allocThread分配Thread空间，一个就是调用系统接口创建线程了，而入口函数是interpThreadStart。

```Java
static Thread* allocThread(int interpStackSize)
{
    Thread* thread;
    u1* stackBottom;

    thread = (Thread*) calloc(1, sizeof(Thread));

    thread->status = THREAD_INITIALIZING;

#ifdef MALLOC_INTERP_STACK
    stackBottom = (u1*) malloc(interpStackSize);
#else
    stackBottom = mmap(NULL, interpStackSize, PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANON, -1, 0);
#endif

    thread->interpStackSize = interpStackSize;
    thread->interpStackStart = stackBottom + interpStackSize;
    thread->interpStackEnd = stackBottom + STACK_OVERFLOW_RESERVE;

    dvmInitInterpStack(thread, interpStackSize);

    return thread;
}
```

可见，这里主要做了两件事，一个是初始化Thread结构体，一个是创建并初始化线程栈。这个线程栈是在堆上分配的，默认是16K，栈底在高地址，栈顶在低地址，并且预留了大小为STACK_OVERFLOW_RESERVE的空间作为栈溢出的“防护地带”。接下来看看dvmInitInterpStack是怎么初始化线程栈的。

```Java
bool dvmInitInterpStack(Thread* thread, int stackSize) {
    return true;
}
```

让人大跌眼镜的是，这个函数什么也没有做，我们再来看线程的入口函数interpThreadStart。

```Java
static void* interpThreadStart(Thread* self) {
    prepareThread(self);

    self->jniEnv = dvmCreateJNIEnv(self);

    Method* run = self->threadObj->clazz->vtable[gDvm.voffJavaLangThread_run];

    dvmCallMethod(self, run, self->threadObj, &unused);
}
```

这里大致看来主要做了三件事，启动线程的准备工作，JNI环境的创建，然后正式调用Thread的Run。先来看看这个prepareThread：

```
static bool prepareThread(Thread* thread) {
#ifdef USE_INDIRECT_REF
    if (!dvmInitIndirectRefTable(&thread->jniLocalRefTable,
            kJniLocalRefMin, kJniLocalRefMax, kIndirectKindLocal))
        return false;
#else
    if (!dvmInitReferenceTable(&thread->jniLocalRefTable,
            kJniLocalRefMax, kJniLocalRefMax))
        return false;
#endif

    if (!dvmInitReferenceTable(&thread->internalLocalRefTable,
            kInternalRefDefault, kInternalRefMax))
        return false;

    return true;
}
```

可见，主要是初始化各种Reference Table。这些Reference是怎么回事我们这里先不表，以后会专门建立一章讨论这个。再来看dvmCreateJNIEnv的实现：

```Java
JNIEnv* dvmCreateJNIEnv(Thread* self)
{
    JavaVMExt* vm = (JavaVMExt*) gDvm.vmList;
    JNIEnvExt* newEnv;

    newEnv = (JNIEnvExt*) calloc(1, sizeof(JNIEnvExt));
    newEnv->funcTable = &gNativeInterface;
    newEnv->vm = vm;

    newEnv->next = vm->envList;
    vm->envList->prev = newEnv;
    vm->envList = newEnv;

    return (JNIEnv*) newEnv;
}
```

这里主要做了两件事：
一、初始化JNIEnv，并将这个newEnv加到gDvm的envList中。值得注意的是，这个newEnv是JNIEnvExt类型的，而返回时转换成了JNIEnv，这是怎么回事呢？

```
typedef struct JNIEnvExt {
    const struct JNINativeInterface* funcTable;     /* must be first */

    const struct JNINativeInterface* baseFuncTable;

    /* pointer to the VM we are a part of */
    struct JavaVMExt* vm;

    Thread* self;

    struct JNIEnvExt* prev;
    struct JNIEnvExt* next;
} JNIEnvExt;

typedef struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;

	.............................
} *JNIEnv;
```

可见这两个结构体的第一个成员都是JNINativeInterface指针，所以如果只是要用到这个JNINativeInterface，那互相转换是没有问题的。

二、将newEnv的funcTable指向全局的gNativeInterface，这个gNativeInterface就是我们平时调用的JNI函数接口，函数很多，只列一些平时常见的，如下：

```
static const struct JNINativeInterface gNativeInterface = {
    DefineClass,
    FindClass,

    ToReflectedField,

    Throw,

    PushLocalFrame,
    PopLocalFrame,

    NewGlobalRef,
    DeleteGlobalRef,
    DeleteLocalRef,
    IsSameObject,
    NewLocalRef,
    EnsureLocalCapacity,

    AllocObject,
    NewObject,

    GetObjectClass,
    IsInstanceOf,

    GetMethodID,

    CallObjectMethod,
    CallIntMethod,

    GetFieldID,

    GetObjectField,
    SetDoubleField,

    GetStaticMethodID,

    CallStaticObjectMethod,
    CallStaticIntMethod,

    GetStaticFieldID,

    GetStaticBooleanField,

    SetStaticObjectField,

    NewString,

    GetStringLength,
    GetStringChars,
    ReleaseStringChars,

    NewStringUTF,
    GetStringUTFLength,
    GetStringUTFChars,
    ReleaseStringUTFChars,

    GetArrayLength,
    NewObjectArray,
    GetObjectArrayElement,
    SetObjectArrayElement,

    NewBooleanArray,

    GetBooleanArrayElements,

    GetByteArrayRegion,

    RegisterNatives,
    UnregisterNatives,

    GetJavaVM,

    NewWeakGlobalRef,
    DeleteWeakGlobalRef,
};
```

接下来，我们终于可以来到线程的执行入口了，如下：

```
Method* run = self->threadObj->clazz->vtable[gDvm.voffJavaLangThread_run];

dvmCallMethod(self, run, self->threadObj, &unused);
```

这里首先从ClassObject的vtable中取出Thread.java中Run函数的索引，得到Method之后，再通过dvmCallMethod调用这个Java函数。激动人心的时刻到来了，原来每个线程的执行入口就是这个dvmCallMethod。我们继续一层层挖下去：

```
void dvmCallMethod(Thread* self, const Method* method, Object* obj,
    JValue* pResult, ...)
{
    va_list args;
    va_start(args, pResult);
    dvmCallMethodV(self, method, obj, false, pResult, args);
    va_end(args);
}
```

可见这里只是调到了dvmCallMethodV，我们再来看看dvmCallMethodV的实现：

```
void dvmCallMethodV(Thread* self, const Method* method, Object* obj,
    bool fromJni, JValue* pResult, va_list args)
{
    const char* desc = &(method->shorty[1]); // [0] is the return type.
    ClassObject* clazz;
    u4* ins;

    clazz = callPrep(self, method, obj, false);

    ins = ((u4*)self->curFrame) + (method->registersSize - method->insSize);

    if (!dvmIsStaticMethod(method)) {
        *ins++ = (u4) obj;
    }

    JNIEnv* env = self->jniEnv;
    while (*desc != '\0') {
        switch (*(desc++)) {
            case 'D': case 'J': {
                u8 val = va_arg(args, u8);
                memcpy(ins, &val, 8);       // EABI prevents direct store
                ins += 2;
                break;
            }
            case 'F': {
                /* floats were normalized to doubles; convert back */
                float f = (float) va_arg(args, double);
                *ins++ = dvmFloatToU4(f);
                break;
            }
            case 'L': {     /* 'shorty' descr uses L for all refs, incl array */
                void* argObj = va_arg(args, void*);
                if (fromJni)
                    *ins++ = (u4) dvmDecodeIndirectRef(env, argObj);
                else
                    *ins++ = (u4) argObj;
                break;
            }
            default: {
                /* Z B C S I -- all passed as 32-bit integers */
                *ins++ = va_arg(args, u4);
                break;
            }
        }
    }

    if (dvmIsNativeMethod(method)) {
        (*method->nativeFunc)(self->curFrame, pResult, method, self);
    } else {
        dvmInterpret(self, method, pResult);
    }

    dvmPopFrame(self);
}
```

这个函数主要做了四件事：首先通过callPrep准备栈帧，然后将参数复制到栈中，调用函数，最后出栈。先来看看callPrep：

```
static ClassObject* callPrep(Thread* self, const Method* method, Object* obj,
    bool checkAccess)
{
    ClassObject* clazz;

    if (obj != NULL)
        clazz = obj->clazz;
    else
        clazz = method->clazz;

    if (dvmIsNativeMethod(method)) {
        dvmPushJNIFrame(self, method);
    } else {
        dvmPushInterpFrame(self, method);
    }

    return clazz;
}
```

这里逻辑很简单，就是根据是否是native函数来采用不同的方式入栈。

```
bool dvmPushJNIFrame(Thread* self, const Method* method)
{
    StackSaveArea* saveBlock;
    StackSaveArea* breakSaveBlock;
    int stackReq;
    u1* stackPtr;

    stackReq = method->registersSize * 4        // params only
                + sizeof(StackSaveArea) * 2;    // break frame + regular frame

    if (self->curFrame != NULL)
        stackPtr = (u1*) SAVEAREA_FROM_FP(self->curFrame);
    else
        stackPtr = self->interpStackStart;

    stackPtr -= sizeof(StackSaveArea);
    breakSaveBlock = (StackSaveArea*)stackPtr;
    stackPtr -= method->registersSize * 4 + sizeof(StackSaveArea);
    saveBlock = (StackSaveArea*) stackPtr;

    breakSaveBlock->prevFrame = self->curFrame;
    breakSaveBlock->savedPc = NULL;             // not required
    breakSaveBlock->xtra.localRefCookie = 0;    // not required
    breakSaveBlock->method = NULL;
    saveBlock->prevFrame = FP_FROM_SAVEAREA(breakSaveBlock);
    saveBlock->savedPc = NULL;                  // not required
    saveBlock->method = method;

    self->curFrame = FP_FROM_SAVEAREA(saveBlock);

    return true;
}

static bool dvmPushInterpFrame(Thread* self, const Method* method)
{
    StackSaveArea* saveBlock;
    StackSaveArea* breakSaveBlock;
    int stackReq;
    u1* stackPtr;

    stackReq = method->registersSize * 4        // params + locals
                + sizeof(StackSaveArea) * 2     // break frame + regular frame
                + method->outsSize * 4;         // args to other methods

    if (self->curFrame != NULL)
        stackPtr = (u1*) SAVEAREA_FROM_FP(self->curFrame);
    else
        stackPtr = self->interpStackStart;

    stackPtr -= sizeof(StackSaveArea);
    breakSaveBlock = (StackSaveArea*)stackPtr;
    stackPtr -= method->registersSize * 4 + sizeof(StackSaveArea);
    saveBlock = (StackSaveArea*) stackPtr;

    breakSaveBlock->prevFrame = self->curFrame;
    breakSaveBlock->savedPc = NULL;             // not required
    breakSaveBlock->xtra.localRefCookie = 0;    // not required
    breakSaveBlock->method = NULL;
    saveBlock->prevFrame = FP_FROM_SAVEAREA(breakSaveBlock);
    saveBlock->savedPc = NULL;                  // not required
    saveBlock->xtra.currentPc = NULL;           // not required?
    saveBlock->method = method;

    self->curFrame = FP_FROM_SAVEAREA(saveBlock);

    return true;
}
```

这两个函数基本上一样，区别在于：分配jni frame时是不计算函数的locals和outs的，因为native函数的执行是直接跳转到native栈上，而不是在dalvik虚拟机的栈上，所以这里没必要操心这个native函数的局部变量和调用其他函数的参数有多少。但是对于java函数就不一样了，所有的执行都是在dalvik虚拟机栈上，所有的参数也是要复制到栈帧中的，所以在分配栈帧时要考虑到所有参数，包括函数自身的参数，局部变量和调用其他函数的参数。

这里重点讨论Interp Frame，即Interpret Function的Frame，如下：

![这里写图片描述](https://github.com/dingjikerbo/Techs-Report/blob/master/images/dalvik_stack.jpg)


这里分配了两个frame，一个是break frame，一个是normal frame。代码中对break frame的解释如下：

> We start by inserting a "break" frame, which ensures that the interpreter
hands control back to us after the function we call returns or an
uncaught exception is thrown.

到这里栈的创建我们大致了解了，接下来就是将参数复制到栈中，起点地址对应栈帧图的ins下端：

```
ins = ((u4*)self->curFrame) + (method->registersSize - method->insSize);
```

值得注意的是，如果不是静态函数，还要给对应的obj拷到栈中作为第一个参数。如果是静态函数就不用，原因是可以获取到Method的clazz，就不用再另外复制一个参数了。

接下来讨论函数的真正执行过程了，如下

```C
if (dvmIsNativeMethod(method)) {
    (*method->nativeFunc)(self->curFrame, pResult, method, self);
} else {
    dvmInterpret(self, method, pResult);
}
```

如果是native函数，则直接跳转到Method中的nativeFunc。否则调用dvmInterpret，可以预见这个dvmInterpret就是dalvik执行Interpreted Code的总入口了。

```
void dvmInterpret(Thread* self, const Method* method, JValue* pResult) {
    InterpState interpState;

    interpState.method = method;
    interpState.fp = (u4*) self->curFrame;
    interpState.pc = method->insns;
    interpState.entryPoint = kInterpEntryInstr;

    typedef bool (*Interpreter)(Thread*, InterpState*);
    Interpreter stdInterp;

    if (gDvm.executionMode == kExecutionModeInterpFast)
        stdInterp = dvmMterpStd;
    else
        stdInterp = dvmInterpretStd;

    (*stdInterp)(self, &interpState);

    *pResult = interpState.retval;
}
```

这里主要是初始化InterpState，并根据executionMode来跳转到不同的解释器入口。如果executionMode是kExecutionModeInterpFast，字面意思是快速模式，猜想可能进行了某些优化，比如用汇编执行。我们这里重点研究普通模式，入口是dvmInterpretStd。调用的时候会传入interpState，且返回值设置在interpState.retval。

```
bool dvmInterpretStd(Thread* self, InterpState* interpState)
{
    JValue retval;
    const Method* curMethod;    // method we're interpreting
    const u2* pc;               // program counter
    u4* fp;                     // frame pointer
    u2 inst;                    // current instruction
    u2 ref;                     // 16-bit quantity fetched directly
    u2 vsrc1, vsrc2, vdst;      // usually used for register indexes
    const Method* methodToCall;
    bool methodCallRange;

    DEFINE_GOTO_TABLE(handlerTable);

    curMethod = interpState->method;
    pc = interpState->pc;
    fp = interpState->fp;
    retval = interpState->retval;   

    methodToCall = (const Method*) -1;

    FINISH(0);                  /* fetch and execute first instruction */

	..............................
```

这个dvmInterpretStd分成两个部分，在分析这个函数之前，我们可以先想象一下，如果我们自己做一个解释器的话该如何做？应该是在一个while循环中，依次从指令数组中取出指令，然后根据指令类型跳转到不同的地方去执行指令。所以这里有两个关键元素，一个是指令数组，一个是指令跳转表。这两个元素在dvmInterpretStd都有体现。

指令数组是interpState->pc，这个pc是u2 *的，所以一条指令对应两个字节。指令跳转表定义如下：

```
DEFINE_GOTO_TABLE(handlerTable);
```

这个宏是在哪里定义的呢？

```
#define DEFINE_GOTO_TABLE(_name) \
    static const void* _name[kNumDalvikInstructions] = {                   
        H(OP_NOP),                                                          \
        H(OP_MOVE),                                                         \
        H(OP_MOVE_FROM16),                                                  \
        H(OP_MOVE_16),                                                      \
        ...............
    }
```

可见，这是一个void *的数组，数组中的元素是什么呢？

```
# define H(_op)             &&op_##_op
```

这里&&的作用是取标签的地址，这个标签是用来goto的，这个用法可能大家不熟悉，可以参考

[http://gcc.gnu.org/onlinedocs/gcc-3.2/gcc/Labels-as-Values.html#fn-1](http://gcc.gnu.org/onlinedocs/gcc-3.2/gcc/Labels-as-Values.html#fn-1)

好了，接下来我们的目光落在最后一句上：

```
FINISH(0); 
```

这算是最终执行Interpreted Code的触发器了。

```
# define FINISH(_offset) {                                                  \
        ADJUST_PC(_offset);                                                 \
        inst = FETCH(0);                                                    \
        goto *handlerTable[INST_INST(inst)];                                \
    }

# define ADJUST_PC(_offset) do {                                            \
        pc += _offset;                                                      \
    } while (false)

#define FETCH(_offset)     (pc[(_offset)])

#define INST_INST(_inst)    ((_inst) & 0xff)
```

可见，FINISH(0)就是从pc中取出第一条指令，通过INST_INST获取指令码，然后goto到对应的标签去执行。

那这里不禁会升起一个疑问，在我自己的构想中，这里应该存在一个循环才对，但是我却没看到，怎么回事呢？

我们先取一条指令来看看它是怎么执行的，这里取最简单的NOP吧，就是什么也不做的意思。如下：

```
HANDLE_OPCODE(OP_NOP)
    FINISH(1);
OP_END
```

这个FINISH(1)就是给pc移到下一个指令，然后取出指令继续执行。

到这里我明白了，所有指令的解释末尾都会有这么一句FINISH，作用就是给pc移到下一个指令，然后取出指令继续执行，循环不息。那什么时候是个头呢？就是函数return的时候，如下：

```
HANDLE_OPCODE(OP_RETURN /*vAA*/)
    vsrc1 = INST_AA(inst);
    retval.i = GET_REGISTER(vsrc1);
    GOTO_returnFromMethod();
OP_END
```

这里结尾就没有FINISH了，之后就是函数执行完毕，该调dvmPopFrame了。

到这里似乎一切都结束了，不过我还是有点意犹未尽，假如函数中又要调别的函数呢？有这么一条指令OP_INVOKE_DIRECT如下：

```
HANDLE_OPCODE(OP_INVOKE_DIRECT)
    goto invokeDirect;
OP_END

invokeDirect:
    {
        u2 thisReg;

        vsrc1 = INST_AA(inst);      
        ref = FETCH(1);             /* method ref */
        vdst = FETCH(2);            /* 4 regs -or- first reg */

        EXPORT_PC();

        thisReg = vdst & 0x0f;

        methodToCall = dvmDexGetResolvedMethod(methodClassDex, ref);
        if (methodToCall == NULL) {
            methodToCall = dvmResolveMethod(curMethod->clazz, ref,
                            METHOD_DIRECT);
        }

		goto invokeMethod;
    }

	.......................

invokeMethod:
    {
        u4* outs;
        int i;

        u4 count = vsrc1 >> 4;

        outs = OUTS_FROM_FP(fp, count);

        switch (count) {
        case 5:
            outs[4] = GET_REGISTER(vsrc1 & 0x0f);
        case 4:
            outs[3] = GET_REGISTER(vdst >> 12);
        case 3:
            outs[2] = GET_REGISTER((vdst & 0x0f00) >> 8);
        case 2:
            outs[1] = GET_REGISTER((vdst & 0x00f0) >> 4);
        case 1:
            outs[0] = GET_REGISTER(vdst & 0x0f);
        default:
            ;
        }
    }
	
	{
        StackSaveArea* newSaveArea;
        u4* newFp;

        newFp = (u4*) SAVEAREA_FROM_FP(fp) - methodToCall->registersSize;
        newSaveArea = SAVEAREA_FROM_FP(newFp);

        newSaveArea->prevFrame = fp;
        newSaveArea->savedPc = pc;
        newSaveArea->method = methodToCall;

        if (!dvmIsNativeMethod(methodToCall)) {
            curMethod = methodToCall;
            methodClassDex = curMethod->clazz->pDvmDex;
            pc = methodToCall->insns;
            fp = self->curFrame = newFp;
            FINISH(0);                              // jump to method start
        } else {
            self->curFrame = newFp;

            (*methodToCall->nativeFunc)(newFp, &retval, methodToCall, self);

            dvmPopJniLocals(self, newSaveArea);
            self->curFrame = fp;
            FINISH(3);
        }
    }
```

我们先来分析invokeDirect，再来分析invokeMethod。这个invokeDirect首先取出要调用的函数ref，然后再取出参数的地址vdst，根据这个ref拿到要调用的函数Method，接着跳转到invokeMethod。值得注意的是ref和vdst各占一条指令，所以函数执行完后需要调用FINISH(3)。

接下来再来看invokeMethod，这里主要做了三件事，首先给参数拷贝到outs中，然后初始化栈帧，最后调用函数。

将参数拷贝到outs中，首先我们要对栈帧结构比较熟悉，我给栈帧的图再贴一遍

![这里写图片描述](https://github.com/dingjikerbo/Techs-Report/blob/master/images/dalvik_stack.jpg)


对着图看，当前栈帧在current fp处，而outs在最底部，要调用的函数参数都作为局部变量在locals处，所以这里要做的是将locals处的参数拷贝到outs处，为什么要这么做呢？其实这里非常巧妙，因为这里的outs就是接下来要调用的函数的ins，两个刚好重合了。为什么这么说呢？因为outs拷贝后，就是初始化栈帧，在save block下偏移methodToCall->registersSize作为新栈的fp，而这个registersSize就包括两部分，locals + ins。其中ins刚好和outs重合，这样之后就不用再次拷贝一次了。

初始化栈帧时有一句很重要：newSaveArea->savedPc = pc; 当函数执行完后返回时要还原pc就是从这里取的。

接下来是最重要的部分了：

```
if (!dvmIsNativeMethod(methodToCall)) {
    curMethod = methodToCall;
    pc = methodToCall->insns;
    fp = self->curFrame = newFp;
    FINISH(0);                              
} else {
    self->curFrame = newFp;

    (*methodToCall->nativeFunc)(newFp, &retval, methodToCall, self);

    dvmPopJniLocals(self, newSaveArea);
    self->curFrame = fp;
    FINISH(3);
}
```

如果是native函数，则直接调其nativeFunc，函数返回后调dvmPopJniLocals，恢复栈帧，再FINISH(3)取下一条指令。

如果不是native的函数，则将pc指向该函数的insns，这个insns是该函数的指令码数组的地址，fp指向为该函数新建的栈帧地址，然后FINISH(0)就是从pc的第一条指令开始执行了。那么有一个问题，pc指向了该函数的insns，那么什么时候恢复呢？应该是在该函数return的时候。

```
HANDLE_OPCODE(OP_RETURN /*vAA*/)
    vsrc1 = INST_AA(inst);
    retval.i = GET_REGISTER(vsrc1);
    goto returnFromMethod;
OP_END

returnFromMethod:
    {
        StackSaveArea* saveArea;

        saveArea = SAVEAREA_FROM_FP(fp);

        fp = saveArea->prevFrame;

        self->curFrame = fp;
        pc = saveArea->savedPc;

        FINISH(3);
    }
```

可见return的时候恢复了栈帧和pc。

到这里，所有的流程我们都走通了，对于Dalvik虚拟机的函数调用我们有了一个全局的认识。在这个过程中，我们为了让主线更分明，而有选择性的忽略了一些细节，不过没关系，在我看来有一个框架性的认识更重要，我们毕竟不是开发dalvik虚拟机的，不用对细节面面俱到，发现问题的时候再去细细研究不迟。有了框架性的认识，再追寻细节时就会事半功倍。






