# PI-Futex 快速验证补丁

## 改动文件
1. runtime/base/mutex.h - 添加 PI 模式标志
2. runtime/base/mutex.cc - 实现 PI-futex 逻辑

## 使用方法
```cpp
// 创建支持优先级继承的 Mutex
Mutex pi_lock("test_lock", kDefaultMutexLevel, true /* use_pi */);
```

## 验证测试
创建 3 个线程：
- 高优先级线程 (nice=-10)
- 中优先级线程 (nice=0) - CPU 密集
- 低优先级线程 (nice=10) - 持有锁

观察高优先级线程是否能快速获取锁。
