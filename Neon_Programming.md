# 高性能Android编程：Neon

NEON指令是ARM上的SIMD指令集，它用于高性能的多媒体计算以及矩阵向量计算。

GCC工具链提供了ARM－NEON的生成器以及arm_neon的头文件，便于使用，免去了直接编写NEON汇编代码的痛苦。

arm_neon支持的指令可以参照 [GCC Neon文档](https://gcc.gnu.org/onlinedocs/gcc-4.4.1/gcc/ARM-NEON-Intrinsics.html) ，neon接口包含了几大类操作：装载，存储，算术运算，移位，倒数，位运算，比较等等。

命名规则是`v{[qual|dual]operation}_{datatype}`。比如vld1q_dup_s32，v代表vector，ld是load的缩写，1q表示加载一个四元组，dup是duplicate复制，s32是signed 32bit，代表每个元素是32位整形，很好理解吧。

以下代码示例是说明用neon指令优化数组反转的算法，通过在三星S4手机上的测试，neon效率是一般实现的1.8倍，优化效果显著。

``` cpp
const int DATA_SIZE = 1920*1080;

void test(JNIEnv * env, jobject jRoot, jobject jObj) {
    int *testSet1 = (int*)malloc(sizeof(int)*DATA_SIZE);
    for(uint32_t i = 0; i<DATA_SIZE; i++) {
        testSet1[i] = i;
    }
    clock_t begin = clock();
    for (uint32_t i=0; i<DATA_SIZE/4/2; i++) {
        int32_t *src = testSet1+i*4;
        int32_t *dest = testSet1+DATA_SIZE - 4*(i+1);
        int32x4_t tmp = vld1q_dup_s32(src);
        int32x4_t destData = vld1q_dup_s32(dest);
        int32x4_t rDestData = vrev64q_s32(destData);
        vst1q_s32(src, rDestData);
        vst1q_s32(dest, tmp);
    }
    clock_t end = clock();
    for (uint32_t i = 0; i<DATA_SIZE/2; i++) {
        int t = testSet1[i];
        int d = testSet1[DATA_SIZE-1-i];
        testSet1[i] = d;
        testSet1[DATA_SIZE-1-i] = t;
    }
    clock_t end2 = clock();
    clock_t cost1 = end-begin;
    clock_t cost2 = end2-end;
    __android_log_print(ANDROID_LOG_DEBUG, "NEON", "last number is %d, acc=%.1fx", testSet1[DATA_SIZE-1], 1.f*cost2/cost1);
    free(testSet1);

    jclass clasz = env->FindClass("com/tencent/helloneon/BenchListener");
    jmethodID method = env->GetMethodID(clasz, "onResult", "(Ljava/lang/String;)V");

    std::stringstream out;
    out << "benchResult:" << 1.f*cost2/cost1;
    env->CallVoidMethod(jObj, method, env->NewStringUTF(out.str().c_str()));
}
```

完整代码在Github [SampleCode](https://github.com/TsinStudio/AndroidDevSample) 上。
