#NDK技术总结

Java向native传int数组，native中类型为jintArray，拿到数组metaData后，若要向该数组赋值，则需要做如下处理

```Java
jint* const ints = (*env)->GetIntArrayElements(env, metaData, 0);
ints[0] = width;
ints[1] = height;
ints[2] = ImageCount;
ints[3] = errorCode;
(*env)->ReleaseIntArrayElements(env, metaData, ints, 0);
```
