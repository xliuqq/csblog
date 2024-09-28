# JNI Demo

## Java Call C/C++ 

> 代码工程见 [JNI 示例](https://gitee.com/oscsc/jvm/tree/master/native/jni)

- 根据 java native 方法 生成 .h 

  ```shell
  # JDK 10以下
  javah  -classpath .  -jni .\src\main\java\com\xliu\cs\jvm\jnative\jni\JNIDemo.java
  # JDK 10之后，生成 .h 文件在 c/ 目录下
  javac  -classpath . -encoding UTF-8 .\src\main\java\com\xliu\cs\jvm\jnative\jni\JNIDemo.java -h c
  ```

- 编译时的链接路径

  ```shell
  # Linux
  gcc -shared -fPIC -D_REENTRANT -I${JAVA_HOME}/include/linux -I${JAVA_HOME}/include -I/home/test/jnidemo/ JNIDemo.c -o libJNIDemo.so
  ```

**关于C代码中的jobject obj 的解释:**

- 如果native方法不是static的话，这个obj就代表这个native方法的**类实例**；

- 如果native方法是static的话，这个obj就代表这个native方法的类的**class对象实例**(static方法不需要类实例的，所以就代表这个类的class对象)；

## C/C++ Call Java 

> C：(*env)->FindClass(env, "java/lang/String")
>
> C++：env->FindClass("java/lang/String")

- 需要指定jni.h的头文件路径，通过 `-I` 选项加在 gcc/g++ 编译时；

- 通过`-Lpath -ljvm`，运行时还需要加载链接库路径；

- 将`$JAVA_HOME/jre/lib/amd64/server`路径添加到PATH（可以搜索到链接库的地址）

  ```c++
  #include<jni.h>
  #include <stdio.h>
  
  // 创建 VM 虚拟机
  JNIEnv* create_vm(JavaVM** jvm, JNIEnv** env)
  {
          JavaVMInitArgs args;
          JavaVMOption options[1];
          args.version = JNI_VERSION_1_6;
          args.nOptions = 1;
          options[0].optionString = "-Djava.class.path=./";
          args.options = options;
          args.ignoreUnrecognized = JNI_FALSE;
          return JNI_CreateJavaVM(jvm, (void **)env, &args);
  }
  
  char* jstringtochar(JNIEnv *env, jstring jstr ) {
    char* rtn = NULL;
    jclass clsstring = env->FindClass("java/lang/String");
    jstring strencode = env->NewStringUTF("utf-8");
    jmethodID mid = env->GetMethodID(clsstring, "getBytes", "(Ljava/lang/String;)[B");
    jbyteArray barr= (jbyteArray)env->CallObjectMethod(jstr, mid, strencode);
    jsize alen = env->GetArrayLength(barr);
    jbyte* ba = env->GetByteArrayElements(barr, JNI_FALSE);
    if (alen > 0) {
      rtn = (char*)malloc(alen + 1);
      memcpy(rtn, ba, alen);
      rtn[alen] = 0;
    }
    env->ReleaseByteArrayElements(barr, ba, 0);
    return rtn;
  }
  
  JNIEXPORT jintArray JNICALL Java_com_example_arrtoc_MainActivity_arrEncode
    (JNIEnv *env, jobject obj, jintArray javaArr){
  	//获取Java数组长度
  	int lenght = (*env)->GetArrayLength(env,javaArr);
   
  	//根据Java数组创建C数组，也就是把Java数组转换成C数组
  	//    jint*       (*GetIntArrayElements)(JNIEnv*, jintArray, jboolean*);
  	int* arrp =(*env)->GetIntArrayElements(env,javaArr,0);
  	//新建一个Java数组
  	jintArray newArr = (*env)->NewIntArray(env,lenght);
   
  	//把数组元素值加10处理
  	int i;
  	for(i =0 ; i<lenght;i++){
  		*(arrp+i) +=10;
  	}
  	//将C数组种的元素拷贝到Java数组中
  	(*env)->SetIntArrayRegion(env,newArr,0,lenght,arrp);
   
  	return newArr;
   
  }
  
  
  int main(int argc, char **argv) {
      JavaVM* jvm;
      JNIEnv* env;
  
      jclass cls;
      int ret = 0;
  
      jmethodID mid;
  
      /* 1. create java virtual machine */
      if(create_vm(&jvm, &env)) {
          printf("can not create jvm\n");
          return -1;
      }
  
      /* 2. get class */
      cls = (*env)->FindClass(env, "Hello");
      if(cls == NULL) {
          printf("can not find hello class\n");
          ret = -1;
          goto destory;
      }
  
      /* 3. create object */
  
      /* 4. call method
       *  4.1 get method
       *  4.2 create parameter
       *  4.3 call method
       */
  
      mid = (*env)->GetStaticMethodID(env, cls, "main", "([Ljava/lang/String;)V");
      if(mid == NULL) {
          ret = -1;
          printf("can not get method\n");
          goto destory;
      }
  
      (*env)->CallStaticVoidMethod(env, cls, mid, NULL);
  
  destory:
      (*jvm)->DestroyJavaVM(jvm);
  
      return ret;
  }
  ```



## 注意

### Java调用C++再调用Java

**在java调用C++代码时在C++代码中调用了AttachCurrentThread方法来获取JNIEnv，此时JNIEnv已经通过参数传递进来，不需要再次AttachCurrentThread来获取。在释放时就会报错。** 

```c++
void cpp2jni(int msg){
    JNIEnv *env = NULL;
    int status;
    bool isAttached = false;
    status = jvm->GetEnv((void**)&env, JNI_VERSION_1_4);
    if (status < 0) {
        // 将当前线程注册到虚拟机中
        if (jvm->AttachCurrentThread(&env, NULL)) {
            return;
        }
        isAttached = true;
    }
    //实例化该类
    //分配新 Java 对象而不调用该对象的任何构造函数。返回该对象的引用。
    jobject jobject = env->AllocObject(global_class);
    //调用Java方法
    (env)->CallVoidMethod(jobject, mid_method,msg);
 
    if (isAttached) {
        jvm->DetachCurrentThread();
    }
}
```



### Java调用C++时的异常处理

**测试场景**：Java使用线程池，启动10个线程，其中一个线程通过JNI调用c++程序：

- 如果c++中没有处理异常，那么会导致jvm的崩溃；
- 如果c++中正确处理了异常，则jvm可以正常运行；
- c++中如果不是异常的错误，如**数组越界，则无法try-catch捕获**，JVM会崩溃。