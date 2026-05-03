# QEMU CPU 方向

!!! note "caspian"

    - 作者：[@caspian](https://github.com/trace1729)

---

## 背景介绍

某双非研二，计算机专业，之前参加过中国科学院大学 ["一生一芯" 项目][1]。体系结构方向，涉及二进制翻译，需要使用 QEMU，所以想通过参加这次训练营来加深自己的理解。同时也想看心智社会中提到的[大型系统的分析方法](#总结)，处理一个中型大小的代码库是否有效。

感谢 @zevorn 老师搭建的实验框架和这一笔记分享与展示交流平台，让我能够分享自己阅读源码时的感悟。

## 专业阶段

从 CPU 建模阶段来建立自己对 QEMU 这个大型系统的理解，为之后在 QEMU 做其他方向打下基础。

## 功能实现

> 本文通过 QEMU 中的翻译流程切入问题，再根据遇到的问题逐步深入

CPU 方向的核心任务是根据 G233 CPU 指令扩展手册 中的指令规格，在 QEMU TCG 前端为 Xg233ai 扩展实现指令翻译。整体流程为 Decodetree 译码 → Helper 实现 → 测试验证。

### 翻译流程

要在指令翻译流程中添加自定义指令，首先需要了解 QEMU 的指令翻译流程

```sql
cpu_exec_loop()
  └─ tb_gen_code()
       └─ setjmp_gen_code()
            └─ riscv_translate_code()        [translate.c:1438]
                 └─ translator_loop(..., &riscv_tr_ops, ...)
                      │  iterates per instruction:
                      └─ riscv_tr_translate_insn()    [translate.c:1368]
                           └─ decode_opc(env, ctx)    [translate.c:1242]
                                ├─ 16-bit: decode_insn16() directly
                                └─ 32-bit: iterate ctx->decoders[]
                                     └─ decode_insn32()  ← THE GENERATED FUNCTION
                                          ├─ matches bit pattern → calls trans_ADD(ctx, &u)
                                          └─ no match → returns false
```

可以看到，调用链末端是 decode_opc，其译码逻辑比较简单

- 如果指令是 2 字节压缩指令，使用 `decode_insn16` 解码
- 如果指令是 4 字节常规指令，遍历 `ctx->decoders`, 使用 opcode 匹配的 decoder 进行译码

```cpp
// target/riscv/translate.c:decode_opc()
    if (ctx->cur_insn_len == 2) {
        ctx->opcode = (uint16_t)opcode;
        if ((has_ext(ctx, RVC) || ctx->cfg_ptr->ext_zca) &&
            decode_insn16(ctx, opcode)) {
            return;
        }
    } else {
        // handle alignment
        // ...
        for (guint i = 0; i < ctx->decoders->len; ++i) {
            riscv_cpu_decode_fn func = g_ptr_array_index(ctx->decoders, i);
            if (func(ctx, opcode)) {
                return;
            }
        }
    }
```

继续追踪 `ctx->decoders`

- `ctx->decoders` 在 `riscv/cpu.c:riscv_tr_init_disas_context` 由 `cpu->decoder` 赋值
- `cpu->decoder` 在 `riscv/tcg-cpu.c:riscv_tcg_cpu_finalize_dynamic_decoder` 中用预定义的 decoder_table 赋值
- [模拟客户机指令](https://qemu.gevico.online/tutorial/2026/ch2/qemu-insn/#_5) 中提到 `decode_insn32` 使用 `decodetree.py` 读取对应 .decode 文件生成的中间产物

```cpp
const RISCVDecoder decoder_table[] = {
    { always_true_p, decode_insn32 },
    { has_xmips_p, decode_xmips},
    { has_xthead_p, decode_xthead},
    { has_XVentanaCondOps_p, decode_XVentanaCodeOps},
};
```

追踪到这里，QEMU 的译码逻辑就比较清晰了：为了添加自定义指令集，需要在 decoder_table 中添加 g233 相关的项，然后在 .decode 文件中定义指令格式

<!-- ??? 标签可以折叠内容 -->

??? "decoder_table 什么时候被设置？"

    上文提到 riscv_tcg_cpu_finalize_dynamic_decoder 设置了 decoder_table，那又是谁调用的它呢？继续反向回溯：

    ```cpp
    riscv/tcg-cpu.c:riscv_tcg_cpu_finalize_dynamic_decoder <-
    riscv/cpu.c:riscv_cpu_finalize_features <-
    riscv/cpu.c:riscv_cpu_realize <-
    riscv/cpu.c:riscv_cpu_common_class_init 设置了 riscv_cpu_realize 函数指针
    ```

    接下来分析其源头 riscv_cpu_common_class_init，它保存了很多 riscv-isa 专用处理函数的函数指针，并且被设置为 RISCVCPUCLASS 的初始化函数（_class_init），保存在 riscv_cpu_type_infos 中。QEMU 使用 DEFINE_TYPES 为 riscv_cpu_type_infos 数组中的所有类型生成类型注册的桩代码（关于 QEMU 中的类型系统可参考 [QOM 面向对象解析](https://qemu.gevico.online/tutorial/2026/ch1/qemu-qom)）。

    ```typescript
    // DEFINE_TYPES(riscv_cpu_type_infos) 会被展开为
    static void do_qemu_init_riscv_cpu_type_infos(void) {
        type_register_static_array(riscv_cpu_type_infos, ARRAY_SIZE(riscv_cpu_type_infos));
        // create a TypeImpl object and append to type linked-list
            //  riscv_cpu_common_class_init is saved in TypeImpl->class_init 
    }
    type_init(do_qemu_init_riscv_cpu_type_infos)

    type_init -> module_init -> 

    // constructor 表示函数在 main 前，由 crt 运行时自动调度运行，反向回溯到此结束
    static void __attribute__((constructor)) do_qemu_init_riscv_cpu_type_infos(void) {
        register_module_init(do_qemu_init_riscv_cpu_type_infos, MODULE_INIT_QOM);
        // create a moduleEntry object append the module to module linked-list
            // do_qemu_init_riscv_cpu_type_infos is saved in moduleEntry->init
    }
    ```

    > TypeInfo 定义了一个 class 的 schema:
    >
    > - 这个 class 在整个系统中的 hierarchy 是什么 (parent)
    > - 如何创建 class (class_init)
    > - 如何创建 class 对应 instance (instance_init)
    > - 该类型生产出来的 instance 占据多大空间 (instance_size)

    正向分析：
    
    依次分析在回溯时定位到的函数调用链 `register_module_init -> riscv_cpu_common_class_init -> riscv_cpu_realize -> ...`

    > 在 main 之前，所有 module 已经被挂到 module linked list 中了

    在 QEMU 初始化时，会依次调用 main() → qemu_init() → module_call_init()，module_call_init 会为此前使用 register_module_init 注册的所有 type 调用对应的 init 函数，也就是 do_qemu_init_xxxx_infos()，将 type 储存在全局的 type hash table 中

    ```
    完成静态分析后，可以使用 gdb 进行动态分析来验证
    # 调试命令 
    gdb --args build/qemu-system-riscv64 -M g233 -m 2G -display none -semihosting -device loader,file=test_gdb.elf

    b do_qemu_init_riscv_cpu_type_infos
    (gdb) bt
    #0  do_qemu_init_riscv_cpu_type_infos () at ../target/riscv/cpu.c:3405
    #1  0x00005555564ceaf4 in module_call_init (type=MODULE_INIT_QOM) at ../util/module.c:109
    #2  0x0000555555f844a2 in qemu_init_subsystems () at ../system/runstate.c:979
    #3  0x0000555555f50f0a in qemu_init (argc=10, argv=0x7fffffffd7c8) at ../system/vl.c:2895
    #4  0x00005555563f8725 in main (argc=10, argv=0x7fffffffd7c8) at ../system/main.c:71
    ```

    接着，来看 riscv_cpu_common_class_init 是什么时候被调用的。

    ```
    #0  riscv_cpu_common_class_init (c=0x5555569009f0, data=0x0) at ../target/riscv/cpu.c:2728
    #1  0x000055555629ff26 in type_initialize (ti=0x5555568ee7c0) at ../qom/object.c:417
    #2  0x000055555629fc70 in type_initialize (ti=0x5555568eed00) at ../qom/object.c:365
    #3  0x000055555629fc70 in type_initialize (ti=0x5555568f1720) at ../qom/object.c:365
    #4  0x000055555629fc70 in type_initialize (ti=0x5555568f1fe0) at ../qom/object.c:365
    #5  0x00005555562a1a6d in object_class_foreach_tramp (key=0x5555568f2160, value=0x5555568f1fe0, opaque=0x7fffffffd480) at ../qom/object.c:1110
    #6  0x00007ffff76f29eb in g_hash_table_foreach () at /lib/x86_64-linux-gnu/libglib-2.0.so.0
    #7  0x00005555562a1b6b in object_class_foreach (fn=0x5555562a1d13 <object_class_get_list_tramp>, implements_type=0x5555559de4c8 "machine", include_abstract=false, opaque=0x7fffffffd4d0)
        at ../qom/object.c:1132
    #8  0x00005555562a1da1 in object_class_get_list (implements_type=0x5555559de4c8 "machine", include_abstract=false) at ../qom/object.c:1189
    #9  0x0000555555f4daa0 in select_machine (qdict=0x5555568fd190, errp=0x7fffffffd520) at ../system/vl.c:1679
    #10 0x0000555555f4f085 in qemu_create_machine (qdict=0x5555568fd190) at ../system/vl.c:2192
    #11 0x0000555555f53722 in qemu_init (argc=10, argv=0x7fffffffd8b8) at ../system/vl.c:3766
    #12 0x00005555563f890f in main (argc=10, argv=0x7fffffffd8b8) at ../system/main.c:71
    ```

    这个 backtrace 容易让人产生误解——riscv_cpu_common_class_init 为什么会在 qemu_create_machine 时被调用呢？

    - 这和 QEMU 中初始化类型的逻辑有关。
    - qemu_create_machine 通过 select_machine 在 type-hash-table 中查找命令行指定的 machine type（比如 -M g233）
    - 此处的“查找”并非按 key-value 查找，而是遍历 hashmap，将符合要求的 type object 保存在链表中，同时将路径上遇到的所有 type object 都初始化了。

    select_machine 获取 machine_type 之后，会根据 type 实例化对应的 instance：virt_machine_instance_init

    - 根据 machine_instance 中设置的 default_cpu_type 实例化 cpu
    - 之后会调用 riscv_cpu_realize 设置 CPU 运行的线程函数 mttcg_start_vcpu_thread（关于 QEMU 创建 CPU 模型的详情，可参考 [QEMU 启动流程分析](https://qemu.gevico.online/tutorial/2026/ch1/qemu-init/#virt-machine) 和 [CPU 建模流程](https://qemu.gevico.online/tutorial/2026/ch2/qemu-cpu-model/)）

    - 也就是 cpu_exec_loop

    ```cpp
    #0  riscv_tcg_cpu_finalize_dynamic_decoder (cpu=0x555556b1ad90) at ../target/riscv/tcg/tcg-cpu.c:1202
    #1  0x00005555561218bb in riscv_cpu_finalize_features (cpu=0x555556b1ad90, errp=0x7fffffffcdd8) at ../target/riscv/cpu.c:915
    #2  0x0000555556121973 in riscv_cpu_realize (dev=0x555556b1ad90, errp=0x7fffffffce40) at ../target/riscv/cpu.c:938
    ...
    #9  0x00005555560f54a5 in riscv_hart_realize (s=0x555556b02140, idx=0, cpu_type=0x555556b1aa30 "gevico-cpu-v1-riscv-cpu", errp=0x7fffffffd100) at ../hw/riscv/riscv_hart.c:145
    #10 0x00005555560f5534 in riscv_harts_realize (dev=0x555556b02140, errp=0x7fffffffd100) at ../hw/riscv/riscv_hart.c:160
    #11 0x000055555629af8a in device_set_realized (obj=0x555556b02140, value=true, errp=0x7fffffffd210) at ../hw/core/qdev.c:523
    ...
    #18 0x000055555611b516 in virt_machine_init (machine=0x555556b01f90) at ../hw/riscv/g233.c:1585
    #19 0x0000555555d2f958 in machine_run_board_init (machine=0x555556b01f90, mem_path=0x0, errp=0x7fffffffd410) at ../hw/core/machine.c:1754
    ...
    #22 0x0000555555f5371d in qemu_init (argc=10, argv=0x7fffffffd7c8) at ../system/vl.c:3847
    #23 0x00005555563f8725 in main (argc=10, argv=0x7fffffffd7c8) at ../system/main.c:71
    ```

    文件总结：

    - target/$ARCH/cpu.c 定义 $ARCH 下的 CPU 蓝图
    - hw/$ARCH/$MACHINE 定义 $ARCH 下 $MACHINE 的虚拟化实现

### 添加虚拟板卡

通过查看 Makefile 构建系统，发现板卡已经添加好了，不过需要手动打开对应的 extension。

### 添加自定义指令集扩展

#### 插桩

> 从在命令行指定扩展 到 在后端翻译时启用扩展支持，这个过程是怎么在 QEMU 中发生的

Summary: 关键结构 `RISCVCPUConfig`， `isa_edata_arr`，`riscv_cpu_vendor_exts`

时间线

1. 在 `riscv_tcg_cpu_instance_init` 创建 instance 时，会遍历 `riscv_cpu_vendor_exts`，为每一个扩展在 instance 的 property hashtable 注册

   ```cpp
   object_property_add(obj, "xtheadba", "bool",
       cpu_get_multi_ext_cfg,       // getter: 读 cfg->ext_xtheadba
       cpu_set_multi_ext_cfg,       // setter: 写 cfg->ext_xtheadba
       NULL, (void*)&ENTRY);        // OPAQUE = 数组条目
   ```

2. 命令行中指定的 isa 和 exts 做比较，使用对应的 getter/setter handler 设置 `riscvcpucfg (cfg->ext_xg233=true)`.

3. 使用 `isa_edata_arr` 做权限检查

4. 在 decoder 中，根据 RISCVCPUConfig 中扩展域的布尔值进行译码

所以为了添加自定义扩展，我们需要分别在 `RISCVCPUConfig`、`isa_edata_arr`、`riscv_cpu_vendor_exts` 三个关键数据结构中各添加一项。

> 为了兼容 QEMU 的构建系统，还需要修改 meson.build，并在 build 目录中添加对应的头文件（decodetree 构建产物和用户自定义的 trans_xg233.c.inc）

这些完成之后就能通过编译，接下来正式开始加入指令实现。由于指令集中的指令语义比较复杂，我选择使用 helper function 来辅助实现。

#### 指令实现

具体的指令实现可分为 5 步

1. 在 `target/riscv/Xg233ai.decode` 中插入指令模式  

2. 在 `target/riscv/insn_trans/trans_Xg233ai.c.inc` 中编写对应的 `trans_{insn}` 函数，该函数调用 `op_helper.c` 中的 `gen_helper_{insn}` 辅助函数  

3. 在 `target/riscv/helper.h` 中插桩

4. 根据指令语义，在 `target/riscv/op_helper.c` 中实现 `helper_{insn}` 辅助函数  

5. 使用 `make -C build/tests/gevico/tcg/riscv64-softmmu run-insn-{insn}` 检查实现是否正确

要完成第 1 步，首先要分析 .decode 文件遵循的模式

根据 [模拟客户机指令](https://qemu.gevico.online/tutorial/2026/ch2/qemu-insn/#_5) 可以了解到 .decode 文件的四层抽象

- Field 字段提取器
- Argument Set 字段容器
- Format 编码布局容器
- Pattern 具体指令，调用相应的 trans_add 函数

自顶向下分析：`Pattern`提取指令的`opcode`、`func3`、`func7`字段，与模板比较；匹配成功后，使用 Format 提取字段。Format 具体提取什么字段由 Argument 指定并打包为结构体，Field 则提供提取字段的具体方法。

2-5：比较机械，我就不再赘述指令的具体实现，而是列出实现的通用方法，以及在实现过程中的一些思考。
通过观察 [硬件手册](https://qemu.gevico.online/exercise/2026/stage1/cpu/cpu-datasheet/)，可以总结出对三种数据元素的操作：

1. 获取寄存器的值：可以直接通过 `env->gpr[rs1]` 获取

2. 获取 guest 地址所存储的值：`cpu_ld{b|w|l|q}_mmu`, b for byte, w for word, l for long, q for quad

3. 向 guest 地址写入值：`cpu_st{b|w|l|q}_mmu`

不过需要注意的是，地址的读写操作要传入正确的 MemOpIdx。为此我们需要理解 QEMU 是如何对 guest 地址进行读写的。

```cpp
uint32_t cpu_ld{}_mmu(CPUArchState *env, vaddr addr,
                     MemOpIdx oi, uintptr_t ra)
```

`cpu_xxx_mmu` 系列 API 会生成对应的 TCG 中间表示，如 `tcg_gen_qemu_ld_tl(dest, addr, ctx->mem_idx, MO_TESQ)`，这里 `ctx->mem_idx` 就是 MMU 索引，也就是我们传入的 MemOpIdx。（详情可见 [MMU](https://qemu.gevico.online/tutorial/2026/ch2/qemu-softmmu/)）QEMU 通过 mem_idx 来确定使用哪一个 MMU 来做 Guest 虚拟地址到 Host 虚拟地址的映射（因为 QEMU 是运行在 Host 上的一个程序，guest 的访存操作最终还是要转化为对 Host 的访存操作）。

有了这三类 API 之后，实现指令其实和写简单的 C 语言程序没什么差别。（~~所以后五个指令实现逃课了~~）

### 调试

1. 在实现 DMA 指令的时候，忘记打开对应机器的扩展选项，但运行测试程序时 QEMU 并没有 crash，而是 crash 在 test 的断言里，令人摸不着头脑。

按照 AI 给出的调试建议，使用 -d 参数查看 log，发现确实生成了 **illegal instruction** ，那为什么 QEMU 在这种情况下还可以正常运行呢？

在[模拟中断和异常](https://qemu.gevico.online/tutorial/2026/ch2/qemu-intr/)这一节中，发现 QEMU 在翻译流程中如果遇到了异常，会设置 pc 为 stvec。stvec 的值又是在哪里设置的呢？

在编译 `test-xxx` 测试用例时，会将其和裸机运行时环境 `tests/gevico/tcg/riscv64/crt` 一起构建，在 `crt/crt.S` 中指定了 `exception_handler`，并将其设置为 stvec 的值。exception_handler 的处理逻辑为直接跳过该条指令，所以就算出现 illegal instruction，程序也能跑下去。

1. 在实现 `trans_vdot` 系列函数的时候，如果直接传递 `a->rd` 会导致编译报错。
之所以要包裹一层 `tcg_constant_tl`，是因为在 trans_ 系列函数内部，所有操作都应和生成 TCG 的中间表示。

- TCG IR 的每条指令的参数都必须是 TCGv（虚拟寄存器），即使是立即数。`tcg_constant_tl` 把 C 的 int 包装成 TCG 能理解的"常量寄存器"
- `gen_helper_*` 也是类似，它生成一条 `INDEX_op_call` 的 TCG IR 指令，用于在翻译时插入对一个 C 函数的调用

```c
static bool trans_vdot(DisasContext *ctx, arg_r *a) {
    //  wrong: gen_helper_vdot(tcg_env, a->rd ,...);    

    gen_helper_vdot(tcg_env, tcg_constant_tl(a->rd), tcg_constant_tl(a->rs1), tcg_constant_tl(a->rs2));    
    return true;
}

```

## 总结

### 如何分析像 QEMU 这样的大型系统

我认为 *Marvin Minsky* 在 心智社会 中提到的方法比较实用：

1. 一个大型系统由数百万个各自只做琐碎小事的部件组成。
2. 理解一个复杂系统的方式：
    - 第一，理解每个子模块各自的功能；
    - 第二，理解子模块之间如何交互；
    - 第三，从更大的视角理解系统整体"做什么"。
    > 因为要真正“知道”一个东西，必须知道它的外部效应。
3. 学习，需要把新知识挂载到记忆中。
4. 人类的记忆是树状结构的：从框架出发，逐步展开到细节。
5. 借助比喻/类比，我们学得更好，因为它复用了已有的记忆结构。

??? note "还原论和整体论"
    实际上是两种观点的综合

    - [还原论](https://en.wikipedia.org/wiki/Reductionism)：还原论是若干相关的哲学观念之一，涉及可以用更简单或更基础的现象来描述的现象之间的关联。它也被描述为一种智识和哲学立场，将复杂系统解释为其各部分的总和，与整体论相对
    - [整体论](http://en.wikipedia.org/wiki/Holism)：整体论是一种跨学科理念，认为系统作为整体具有超越其组成部分属性的特性。格言"整体大于部分之和"常被视作这一主张的概括。

具体到 QEMU：

1. **从外部效应建立基本框架**：在本题框架内，QEMU 的核心工作流是：guest 二进制指令 → [解码] → [翻译成 TCG 中间表示] → [编译成 host 机器码] → [在真实 CPU 上执行]
2. **展开到细节**：设备的接入、CPU 的配置、内存的地址翻译，都是围绕这条主线的工作。
3. **继续拆解到对象交互**:  QOM 类型系统

QEMU 由数百个模块组成，每个模块：CPU 类型、机器板卡、网卡、PCI 总线，都是一个 QOM 类，QOM 定义了一个通用接口

1. **TypeInfo** ：注册一个类：名字、大小、父类、初始化函数
2. **Class** 和 **Instance**：
    1. `callback function`：定义一个对象与外界交互的方式
    2. `persistant data structure`：存储自身信息的数据结构

通过 LLM，可以快速的将 QEMU 所执行的功能，映射为 QOM 之间的交互 [opencode 示例](https://opncd.ai/share/pnjNBbeR)

> 其实 The Society of Mind 这段描述是 Marvin Minsky 对智能的定义，和现在的 Agent 系统比较相似。

### 如何理解函数回调

QEMU 的实现中充斥着各种函数回调，理解起来比较困难

- 抓住两个关键行为进行分析
  - 函数回调设置点
  - 函数回调调用点

[1]: https://ysyx.oscc.cc/

## 附录

<!-- ??? 标签可以折叠内容 -->

??? opencode 示例
    # QEMU 启动流程分析：Machine × CPU × Accelerator 的对象交互视角

    > generated by deepseek4-flash, in opencode

    > 方法参考：Marvin Minsky, *The Society of Mind*
    >
    > 1. 一个大型系统由数百万个各自只做琐碎小事的部件组成。
    > 2. 理解一个复杂系统的方式：
    >    - 第一，理解每个子模块各自的功能；
    >    - 第二，理解子模块之间如何交互；
    >    - 第三，从更大的视角理解系统整体"做什么"。
    > 3. 学习，需要把新知识挂载到记忆中。
    > 4. 人类的记忆是树状结构的：从框架出发，逐步展开到细节。
    > 5. 借助比喻/类比，我们学得更好，因为它复用了已有的记忆结构。

    ## 0. 框架：QEMU 核心工作流

    ```
    guest 二进制 → [解码指令] → [翻译成 TCG IR] → [编译成 host 机器码] → [在真实 CPU 上执行]
    ```

    设备接入、CPU 配置、内存地址翻译，都是围绕这条主线的工作。

    QEMU Object Model (QOM) 是连接所有子模块的骨架：

    - **TypeInfo**：注册一个类——名字、大小、父类、初始化函数
    - **Class**：与外界交互的方式（member function）及存储自身信息的数据结构（member variable）——`object_class_property_*`
    - **Instance**：具体的对象实例——`object_property_*`

    ---

    ## 1. 对象创建与连接：三个 QOM 对象的诞生

    ```
    时间轴                                  QEMU 进程地址空间
    │
    │  [1] qemu_create_machine()                [system/vl.c:2190]
    │       object_new_with_class(MachineClass)  ───→  MachineState (e.g. "virt")
    │       cpu_exec_init_all()                          └─ .accelerator = NULL
    │                                                    └─ .cpu_type = NULL
    │
    │  [2] configure_accelerators()             [system/vl.c:2426]
    │       object_new_with_class(AccelClass)   ───→  AccelState (e.g. TCGState)
    │       accel_init_machine()                         │
    │         └─ tcg_init_machine()                      │
    │              ├─ page_init()                        │
    │              ├─ tb_htable_init()                   │
    │              ├─ tcg_init()  ← TCG 核心初始化       │
    │              └─ tcg_prologue_init()  ← 生成 prologue│
    │                                                    │
    │       machine->accelerator = accel  ←─── 连线：Machine → Accel
    │
    │  [3] qemu_init_board()                   [system/vl.c:2707]
    │       machine_run_board_init()            [hw/core/machine.c:1668]
    │         ├─ accel_init_interfaces()  ← 把 AccelOpsClass、
    │         │                              AccelCPUClass 挂到 CPUClass 上
    │         └─ machine_class->init(machine)  ← 板卡 init
    │              └─ cpu_create(cpu_type)
    │                   object_new(CPUType)    ───→  CPUState
    │                   qdev_realize()                  │
    │                     ├─ accel_cpu_common_realize()  │
    │                     │   ├─ accel_cpu->             │
    │                     │   │    cpu_target_realize()   │
    │                     │   └─ acc->cpu_common_realize()│
    │                     │       └─ tcg_exec_realizefn() │
    │                     │            └─ tcg_ops->       │
    │                     │              initialize()     │
    │                     │              (arch-specific)  │
    │                     └─ cpu_common_realizefn()       │
    │                          └─ qemu_init_vcpu()        │
    │                               └─ cpus_accel->       │
    │                                 create_vcpu_thread()│
    │                                  ┌──────────────────┘
    │                                  ▼
    │                            pthread_create()
    │                            └─ mttcg_cpu_thread_fn()
    │                                 └─ tcg_cpu_exec() loop
    ```

    ---

    ## 2. 对象之间的关系（交互图）

    ```
    MachineState                        CPUState
    ┌──────────────────┐               ┌────────────────────┐
    │ .accelerator ────────??           │ .cc (CPUClass) ───────┐
    │ .cpu_type      │  │               │ .thread             │  │
    │ .ram           │  │               │ .stopped            │  │
    │ .smp           │  │               │ .halted             │  │
    │ .numa_state    │  │               │ .tcg_ops            │  │
    └──────────────────┘  │               └────────────────────┘  │
                        │                                       │
            ┌──────────┘          ┌────────────────────────────┘
            ▼                     ▼
    AccelState (TCGState)          CPUClass
    ┌──────────────────┐            ┌──────────────────────────┐
    │ .tb_size         │            │ .tcg_ops  → TCGCPUOps    │
    │ .mttcg_enabled   │            │ .accel_cpu → AccelCPUClass│
    │ .splitwx_enabled │            │ .class_by_name            │
    │                   │            │ .parse_features           │
    │ AccelClass:       │            └──────────────────────────┘
    │ .init_machine     │
    │ .ops → AccelOpsClass          AccelOpsClass (cpus_accel)
    │        ┌──────────────────────┐
    │        │ .create_vcpu_thread  │ ← mttcg_start_vcpu_thread
    │        │ .cpu_exec_all        │     or rr_start_vcpu_thread
    │        │ .cpu_exec            │
    │        └──────────────────────┘
    └──────────────────┘
    ```

    ### 关键交互连线

    | 连线 | 地点 | 作用 |
    |------|------|------|
    | `Machine → Accel` | `system/vl.c:2411` → `machine_run_board_init` 前 | Machine 持有 Accelerator 实例引用 |
    | `Accel → Machine` | `accel/tcg/tcg-all.c:104` `tcg_init_machine(accel, machine)` | Accelerator 用 Machine 信息（smp 等）配置自身 |
    | `Accel → CPU` | `accel/accel-common.c:80` `accel_init_interfaces()` | 把 AccelCPUClass 注入到 CPUClass |
    | `Accel → CPU thread` | `system/cpus.c:709` `cpus_accel->create_vcpu_thread()` | Accelerator 负责创建和管理 VCPU 线程 |
    | `CPU → Accel(class)` | `hw/core/cpu-common.c:231` `accel_cpu_common_realize()` | CPU realize 时回调 accelerator 的初始化 |
    | `CPU exec → TCG ops` | `accel/tcg/cpu-exec.c:970` `cc->tcg_ops->translate_code()` + `tcg_gen_code()` | 运行时通过 CPUClass 的 tcg_ops 调用翻译 |

    **核心模式**：Machine 负责编排（orchestration），Accelerator 负责执行策略（execution strategy），CPU 负责提供架构描述（translation callbacks）。三者通过 QOM 属性/方法指针松散耦合。

    ---

    ## 3. 核心路径逐层展开

    ### 3.1 Machine → Accelerator 环境准备

    ```c
    // system/vl.c:2426
    configure_accelerators()
    └─ accel_find("tcg")  → 找到 AccelClass
    └─ object_new_with_class(ac)  → 创建 TCGState 实例
    └─ accel_init_machine(accel, machine)            // accel/accel-system.c:38
        └─ ACCEL_GET_CLASS(accel)->init_machine(accel, ms)
                └─ tcg_init_machine()                  // accel/tcg/tcg-all.c:104
                    ├─ 确定 mttcg_enabled (单线程/多线程)
                    ├─ page_init()           —— 页表缓存初始化
                    ├─ tb_htable_init()      —— TranslationBlock 哈希表
                    ├─ tcg_init()            —— TCG 上下文、寄存器池
                    └─ tcg_prologue_init()   —— 生成 host prologue 代码
    ```

    ### 3.2 Machine → CPU 创建

    ```c
    // hw/core/machine.c:1668
    machine_run_board_init(machine)
    ├─ accel_init_interfaces(accel_class)             // accel/accel-common.c:80
    │    ├─ accel_init_ops_interfaces()
    │    │    └─ 查找 "tcg-accel-ops" → 得到 AccelOpsClass
    │    │         └─ cpus_accel = 这个 ops class  ← 全局指针
    │    │
    │    └─ accel_init_cpu_interfaces()
    │         └─ 查找 "tcg-<arch>-cpu" → 得到 AccelCPUClass
    │              └─ 挂到 cpu->cc->accel_cpu  ← 每个 CPUClass 都有
    │
    └─ machine_class->init(machine)  ← 板卡特定的 init

            ┌─ 以 RISC-V virt 为例 ──────────────────────────────────┐
            │ virt_machine_init()        [hw/riscv/virt.c]            │
            │   └─ cpu_create(machine->cpu_type)                      │
            │        = riscv-cpu-<arch>                               │
            │                                                         │
            │        cpu_create()          [hw/core/cpu-common.c:59]  │
            │          ├─ object_new(typename)                        │
            │          │    → riscv_cpu_initfn()                      │
            │          │       └─ 设置 CSR 默认值、CPUFeature         │
            │          │                                              │
            │          └─ qdev_realize(DEVICE(cpu), NULL)             │
            │               ├─ accel_cpu_common_realize()             │
            │               │    ├─ accel_cpu->cpu_target_realize()   │
            │               │    │    └─ (riscv: 无额外操作)          │
            │               │    └─ acc->cpu_common_realize()         │
            │               │         └─ tcg_exec_realizefn()         │
            │               │              └─ tcg_ops->               │
            │               │                initialize(cpu)          │
            │               │                = riscv_translate_init   │
            │               │                  └─ 注册 CSR 名→编号映射 │
            │               │                    创建 gen_helper_* 等 │
            │               │                                         │
            │               └─ cpu_common_realizefn()                  │
            │                    └─ qemu_init_vcpu(cs)                 │
            │                                                         │
            └─────────────────────────────────────────────────────────┘
    ```

    ### 3.3 Accelerator → VCPU 线程创建

    ```c
    // system/cpus.c:709
    qemu_init_vcpu(cpu)
    └─ cpu_address_space_init(cpu, ...)  ← 初始化地址空间
    └─ cpus_accel->create_vcpu_thread(cpu)
        │
        │  mttcg_start_vcpu_thread()    // accel/tcg/tcg-accel-ops-mttcg.c:124
        │    ├─ tcg_cpu_init_cflags(cpu)  ← 设置并行/icount 标志
        │    └─ qemu_thread_create(thread, "CPU %d/TCG",
        │                           mttcg_cpu_thread_fn, cpu)
        │
        ▼
    // accel/tcg/tcg-accel-ops-mttcg.c:65
    mttcg_cpu_thread_fn(arg=cpu)
        ├─ tcg_register_thread()
        ├─ cpu_thread_signal_created(cpu)
        │    → 主线程在 qemu_init_vcpu() 的 while(!cpu->created) 中被唤醒
        │
        └─ 主循环：
            do {
            if (cpu_can_run(cpu)) {
                r = tcg_cpu_exec(cpu);  ← 进入翻译执行循环
                ├─ cpu_exec_start(cpu)
                ├─ cpu_exec(cpu)      ← 核心执行函数
                └─ cpu_exec_end(cpu)
            }
            } while (!cpu->unplug || cpu_can_run(cpu));
    ```

    ### 3.4 CPU 执行：TranslationBlock 的诞生与执行

    ```c
    // accel/tcg/tcg-accel-ops.c:77
    tcg_cpu_exec(cpu)
    └─ cpu_exec(cpu)                              // accel/tcg/cpu-exec.c:1019
        └─ cpu_exec_loop(cpu, sc)                // accel/tcg/cpu-exec.c:933
                └─ 内层循环 (无中断/异常时):
                    while (1) {
                    tb = tb_lookup(cpu, s);
                    if (tb == NULL) {
                        tb = tb_gen_code(cpu, s);
                    }
                    cpu_loop_exec_tb(cpu, tb, ...);
                        └─ cpu_tb_exec(cpu, tb);  ← 跳转到 host code
                    }

    // accel/tcg/translate-all.c:261
    tb_gen_code(cpu, s)
        ├─ 1. get_page_addr_code_hostp(env, pc) → 解析 guest 虚拟地址
        │                                        → 得到 host 地址
        │
        ├─ 2. tcg_tb_alloc() → 分配 TranslationBlock
        │
        ├─ 3. setjmp_gen_code(env, tb, ...)
        │    ├─ tcg_func_start(tcg_ctx)
        │    │
        │    ├─ cs->cc->tcg_ops->translate_code(cs, tb, ...)
        │    │    = riscv_translate_code()         [target/riscv/translate.c:1441]
        │    │    └─ translator_loop(cs, tb, ..., &riscv_tr_ops, &ctx.base)
        │    │         └─ while (true):
        │    │              ops->translate_insn(db, cpu)
        │    │              = riscv_tr_translate_insn()
        │    │                └─ decode RISC-V insn → emit TCG IR
        │    │                   (gen_load, gen_store, gen_arith, ...)
        │    │
        │    └─ tcg_gen_code(tcg_ctx, tb, pc)      // tcg/tcg.c
        │         └─ 遍历 TCG IR → 寄存器分配 → host 指令发射
        │            └─ tcg_out_op() 系列函数 → 写 host 机器码到 code_gen_buffer
        │
        └─ 4. tb_link_page(tb) → 插入物理页面的 TB 哈希表

    // accel/tcg/cpu-exec.c
    cpu_tb_exec(cpu, tb)
        └─ 通过函数指针跳转到 tb->tc.ptr (host code)
            └─ 执行翻译好的 host 机器码
                └─ 遇到 exit_tb / 中断 / 异常 → 返回 cpu_exec_loop
    ```

    ### 3.5 translator_loop 细节：从 guest 指令到 TCG IR

    ```c
    // accel/tcg/translator.c:122
    translator_loop(cpu, tb, max_insns, pc, host_pc, ops, db)
    ├─ ops->init_disas_context(db, cpu)    ← 初始化反汇编上下文
    ├─ gen_tb_start(db, cflags)            ← 发射 TB 开始标记 (exit check)
    │
    └─ while (true):
        ├─ ops->insn_start(db, cpu)       ← 标记一条新指令开始
        ├─ ops->translate_insn(db, cpu)   ← 核心：解码 + 发射 TCG IR
        │    ↓
        │    riscv_tr_translate_insn()    [target/riscv/translate.c]
        │      ├─ decode_insn16() / decode_insn32()
        │      ├─ gen_arith() / gen_load() / gen_store() / ...
        │      │    ↓
        │      │    tcg_gen_addi_i64()     ← TCG IR 发射
        │      │    tcg_gen_ld_i64()       ← TCG IR 发射
        │      │    tcg_gen_brcond()       ← TCG IR 发射
        │      │    gen_helper_*()         ← helper 函数调用
        │      │
        │      └─ db->pc_next += insn_len  ← 更新 PC
        │
        ├─ (if plugin_enabled) plugin_gen_insn_end()
        │
        └─ break if: db->is_jmp != DISAS_NEXT
                    || tcg_op_buf_full()
                    || num_insns >= max_insns

    └─ ops->tb_stop(db, cpu)              ← 发射 TB 结束代码 (exit_tb)
    └─ gen_tb_end(tb, ...)                ← 结束标记
    ```

  

    ---

    ## 4. 关键源码索引

    | 功能 | 文件 | 关键函数 |
    |------|------|----------|
    | 入口 | `system/main.c:69` | `main()` |
    | 初始化编排 | `system/vl.c:2842` | `qemu_init()` |
    | Machine 创建 | `system/vl.c:2190` | `qemu_create_machine()` |
    | Machine 类型注册 | `hw/core/machine.c:1839` | `machine_info` |
    | Machine 板卡初始化 | `hw/core/machine.c:1668` | `machine_run_board_init()` |
    | Accelerator 配置 | `system/vl.c:2426` | `configure_accelerators()` |
    | Accelerator 类型 | `include/qemu/accel.h:32` | `TYPE_ACCEL` |
    | AccelOps 注入 | `accel/accel-common.c:80` | `accel_init_interfaces()` |
    | Accel 通用 realize | `accel/accel-common.c:93` | `accel_cpu_common_realize()` |
    | TCG init_machine | `accel/tcg/tcg-all.c:104` | `tcg_init_machine()` |
    | CPU QOM 类型 | `hw/core/cpu-common.c:392` | `cpu_type_info` |
    | CPU 创建 | `hw/core/cpu-common.c:59` | `cpu_create()` |
    | CPU realize | `hw/core/cpu-common.c:247` | `cpu_common_realizefn()` |
    | CPU exec realize | `hw/core/cpu-common.c:231` | `cpu_exec_realizefn()` |
    | VCPU 线程初始化 | `system/cpus.c:709` | `qemu_init_vcpu()` |
    | MTTCG VCPU 线程 | `accel/tcg/tcg-accel-ops-mttcg.c:65` | `mttcg_cpu_thread_fn()` |
    | RR VCPU 线程 | `accel/tcg/tcg-accel-ops-rr.c` | `rr_cpu_thread_fn()` |
    | TCG CPU exec | `accel/tcg/tcg-accel-ops.c:77` | `tcg_cpu_exec()` |
    | CPU 执行主循环 | `accel/tcg/cpu-exec.c:1019` | `cpu_exec()` |
    | CPU 执行内层循环 | `accel/tcg/cpu-exec.c:933` | `cpu_exec_loop()` |
    | TB 生成 | `accel/tcg/translate-all.c:261` | `tb_gen_code()` |
    | setjmp 编译 | `accel/tcg/translate-all.c:238` | `setjmp_gen_code()` |
    | 通用翻译器循环 | `accel/tcg/translator.c:122` | `translator_loop()` |
    | TCG IR → host 代码 | `tcg/tcg.c` | `tcg_gen_code()` |
    | RISC-V translate_code | `target/riscv/translate.c:1441` | `riscv_translate_code()` |
    | ARM translate_code | `target/arm/tcg/translate.c:6878` | `arm_translate_code()` |
    | RISC-V translate_insn | `target/riscv/translate.c` | `riscv_tr_translate_insn()` |

    ---

    ## 5. 总结：对象视角下的系统架构

    ### 第一层（子系统功能）

    - **Machine** (`hw/core/machine.c`): 拓扑描述 + 板卡初始化编排。拥有 CPU 类型名、内存配置、SMP 拓扑、NUMA 信息。
    - **Accelerator** (`accel/`): 执行策略提供者。负责初始化 TCG 引擎（或 KVM fd）、创建 VCPU 线程、定义 `cpu_exec` 的实现策略。
    - **CPU** (`hw/core/cpu-common.c` + `target/`): 架构描述者。提供指令翻译回调（`translate_code`）、初始化回调（`initialize`）、寄存器状态定义。

    ### 第二层（子模块交互）

    ```
    Machine → Accelerator:     machine->accelerator 引用
                            accel_init_interfaces() 将加速器 ops 注入全局
    Machine → CPU:             cpu_create() 通过 QOM 创建
                            qemu_init_vcpu() 触发线程创建
    Accelerator → CPU:         accel_cpu_common_realize() 回调 CPU 初始化
                            cpus_accel->create_vcpu_thread() 启动 VCPU 线程
    CPU ↔ Accelerator(运行时): tb_gen_code() 中调用 cc->tcg_ops->translate_code()
                            cpu_tb_exec() 执行 tcg_gen_code() 产出的 host code
    ```

    ### 第三层（系统整体做什么）

    QEMU (TCG 模式) = 一个动态二进制翻译器。

    它把 guest ISA 指令通过 `translator_loop` → `translate_insn` 解码为 TCG IR，再通过 `tcg_gen_code` 编译成 host ISA，最后在真实 CPU 上执行。

    Machine / Accelerator / CPU 三个 QOM 对象的交互，本质上是在完成 **"谁（CPU）来运行什么（Machine 配置）以及怎么跑（Accelerator 策略）"** 的三方协议。

    ??? note
        The mind are the most complexed thing in the world. And the most complex craft is man-made. So there must be a way to understand all the complex craft by understanding how the mind works.
