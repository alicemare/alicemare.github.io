---
title: flamming-my-compiler
date: 2021-01-06 19:48:43
tags: compilor
---

以华为大学生编译器设计大赛的特等奖代码为例研究编译器的一些IR pass技巧和Code gen部分，对整个编译器的IR和后端有深入的掌握和了解

<!--more-->

项目地址: [CSC2020-USTC-FlammingMyCompiler](https://github.com/mlzeng/CSC2020-USTC-FlammingMyCompiler)

参考书：《现代编译原理 C语言描述》（虎书）


PM 
AccumulatePattern



PassManager
私有对象

private:
  std::vector<Pass *> passes_;
  Module *m_;
  IRCheck ir_check;

addPass模板方法
template <typename PassTy> void addPass(bool print_ir = false);

```c++
void run(bool print_ir = false) {
    auto i = 0;
    // check ast
    try {
      ir_check.run();
    } catch (...) {
      std::cerr << "IRCheck ERROR after SYSYCBuilder" << std::endl;
      exit(1);
    }
    for (auto pass : passes_) {
      i++;
      pass->run();//逐一调用pass的run
      if (print_ir || pass->isPrintIR()) {
        std::cerr << ">>>>>>>>>>>> After pass " << pass->getName()
                  << " <<<<<<<<<<<<" << std::endl;
        m_->print();
      }
      try {
        ir_check.run();
      } catch (...) {
        std::cerr << "IRCheck ERROR after pass " << pass->getName()
                  << std::endl;
        exit(i * 2);
      }
    }
```





Module

Pass 基于 Module
  Module *m_;
  std::string name_; 
  bool print_ir_ = false;
纯虚函数
vitrual void run() = 0

纯虚函数没有函数体,最后面的“=0”并不表示函数返回值为0，它只起形式上的作用，告诉编译系统“这是纯虚函数”;这是一个声明语句，最后应有分号。纯虚函数只有函数的名字而不具备函数的功能，不能被调用。它只是通知编译系统: “在这里声明一个虚函数，留待派生类中定义”。在派生类中对此函数提供定义后，它才能具备函数的功能，可被调用。

virtual关键字又是和override紧密不可分的，如果要实现virtual方法就必须要使用override或new

Analyse Transform IRCheck都直接继承
IRCheck的函数

void CheckParent();
void CheckPhiPosition();
void CheckRetBrPostion();
void CheckTerminate();
void CheckPredSucc();
void CheckEntry();
void CheckUseList();
void CheckOperandExit();



