# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"
    - 作者：[@scrail](https://github.com/scrail)
---

## 背景介绍

计算机专业，去年参加过一回但是因为各种原因没有完成，希望这此能够完整的参与下来。

## 专业阶段

完成CPU实验和GPGPU实验的基础部分，之后会尝试完成GPU的进阶实验。

### CPU实验
本实验从完成的角度而言并不困难，只需要理解TCG是如何从客户机指令转化为目标机指令的过程：
- RISCV架构的ISA定义与指令添加：这部分已经由decodetree提供了相当便利的译码设施，只需要按照指令集编码格式注册新的指令，相应的译码函数与翻译函数就会被自动添加进TCG翻译过程。
- ISA转换为TCG IR：对应到翻译函数——也就是trans_XXX的具体实现，是本任务的核心，在基础实验中只需要编写C helper函数，使用工具宏注册后再trans函数中调用gen_helper即可。
- TCG IR经过优化后变为二进制BB:这是最终的执行过程，从基础实验的角度不需要太过深入，不过这里也一并参考源码进行一些分析。

在完成任务的过程中，在AI的辅助下，我对QEMU TCG的整体代码进行了分析，下边对学习到的内容进行梳理。

#### 译码

> 客户机指令 -> trans_XXX()函数

译码是架构特定的，因此相关内容实现在`target/riscv/`下，主要是`translate.c`。

先看下函数的注册链条

- `TypeInfo riscv_cpu_type_infos`注册`.class_init = riscv_cpu_common_class_init`
- `riscv_cpu_common_class_init`注册`cc->tcg_ops = &riscv_tcg_ops`
- `riscv_tcg_ops`的`.translate_code`字段指向`riscv_translate_code`
- `riscv_translate_code`调用`translator_loop`传入了`riscv_tr_op`
- `riscv_tr_ops.translate_insn`字段指向`riscv_tr_translate_insn`
- `riscv_tr_translate_insn`调用`decode_opc`，这是译码的核心执行逻辑

至于调用过程发生在`translator_loop()`中，这里也是TB prologue和 epilogue被添加进IR的地方。

```c
void translator_loop(CPUState *cpu, TranslationBlock *tb, ...)
{
    /* Initialize DisasContext */
    ops->init_disas_context(db, cpu);

    /* Start translating.  */
    icount_start_insn = gen_tb_start(db, cflags);
    ops->tb_start(db, cpu);

    while (true) {
        db->num_insns++;
        ops->insn_start(db, cpu);
        db->insn_start = tcg_last_op();

        ops->translate_insn(db, cpu);   // riscv_tr_translate_insn

        // TB 终止判断
        // - 发生跳转
        // - op缓冲区满
        // - 达到指令上限
        if (db->is_jmp != DISAS_NEXT) { break; }    
        if (tcg_op_buf_full() || db->num_insns >= db->max_insns) {
            db->is_jmp = DISAS_TOO_MANY;
            break;
        }
    }

    /* Emit code to exit the TB, as indicated by db->is_jmp.  */
    ops->tb_stop(db, cpu);
    gen_tb_end(tb, cflags, icount_start_insn, db->num_insns);
}
```

decode_opc处理取指和译码两部分：
- 取指：从当前PC加载32位操作码`translator_ldl_end`，包含压缩指令与跨页分支逻辑
- 译码：从`ctx->decoders`指针表调用正确的译码函数。

```c
static void decode_opc(CPURISCVState *env, DisasContext *ctx)
{
    uint32_t opcode;
    ......
    for (guint i = 0; i < ctx->decoders->len; ++i) {
        riscv_cpu_decode_fn func = g_ptr_array_index(ctx->decoders, i);
        if (func(ctx, opcode)) {
            return;
        }
    }

    gen_exception_illegal(ctx);
}
```

全量`decoder_table`定义在`translate.c:1235`，每个译码函数都配备了guard函数，因为在实际场景下并不会将此数组全部注册上。
```c
const RISCVDecoder decoder_table[] = {
    { always_true_p, decode_insn32 },
    { has_xmips_p, decode_xmips},
    { has_xthead_p, decode_xthead},
    { has_XVentanaCondOps_p, decode_XVentanaCodeOps},
};

// target/riscv/tcg/tcg-cpu.c
void riscv_tcg_cpu_finalize_dynamic_decoder(RISCVCPU *cpu)
{
    GPtrArray *dynamic_decoders;
    dynamic_decoders = g_ptr_array_sized_new(decoder_table_size);
    for (size_t i = 0; i < decoder_table_size; ++i) {
        if (decoder_table[i].guard_func &&
            decoder_table[i].guard_func(&cpu->cfg)) {
            g_ptr_array_add(dynamic_decoders,
                            (gpointer)decoder_table[i].riscv_cpu_decode_fn);
        }
    }

    cpu->decoders = dynamic_decoders;
}
```
具体的译码过程，以`decode_insn32`为例，是`decodetree`工具从`target/riscv/insn32.decode`自动生成的模式匹配函数。它将opcode的bit字段与预定义的指令模式进行比对，匹配成功后调用对应的`trans_XXX()`函数。函数内容在自动生成的`decode-insn32.c.inc`中。

```c
static bool decode_insn32(DisasContext *ctx, uint32_t insn)
{
    union {
        arg_atomic f_atomic;
        arg_b f_b;
        arg_decode_insn3218 f_decode_insn3218;
        arg_decode_insn3219 f_decode_insn3219;
        ...
        ...
        arg_rnfvm f_rnfvm;
        arg_s f_s;
        arg_shift f_shift;
        arg_u f_u;
    } u;

    switch (insn & 0x0000007f) {
    case 0x00000003:
        /* ........ ........ ........ .0000011 */
        switch ((insn >> 12) & 0x7) {
        case 0x0:
            /* ........ ........ .000.... .0000011 */
            /* ../target/riscv/insn32.decode:141 */
            decode_insn32_extract_i(ctx, &u.f_i, insn);
            if (trans_lb(ctx, &u.f_i)) return true;
            break;
        
            ...
            ...
        }
    ...
    ...
    case 0x00000033:
        /* ........ ........ ........ .0110011 */
        switch (insn & 0x3e007000) {
        case 0x00000000:
            /* ..00000. ........ .000.... .0110011 */
            decode_insn32_extract_r(ctx, &u.f_r, insn);
            switch ((insn >> 30) & 0x3) {
            case 0x0:
                /* 0000000. ........ .000.... .0110011 */
                /* ../target/riscv/insn32.decode:159 */
                if (trans_add(ctx, &u.f_r)) return true;
                break;
            case 0x1:
                /* 0100000. ........ .000.... .0110011 */
                /* ../target/riscv/insn32.decode:160 */
                if (trans_sub(ctx, &u.f_r)) return true;
                break;
            }
            break;
    }
}
```
#### TCG IR生成

> trans_XXX() -> TCG IR

正如讲义中所说，trans_XXX()函数是为了生成TCG IR，QEMU中每条IR指令对应一个`TCGOp`节点，全部链接在`TCGContext->ops`双向链表中。
```c
// include/tcg/tcg.h:310
struct TCGOp {
    TCGOpcode opc   : 8;    // 操作码INDEX_op_XXX
    unsigned nargs  : 8;    // 参数个数
    unsigned param1 : 8;    // 灵活语义字段，下方通过宏定义包裹不同语义
    unsigned param2 : 8;    // 多数指令中1表示操作数类型，2表示操作标志位
    
    TCGLifeData life;       // 生命周期数据（优化过程使用）

    QTAILQ_ENTRY(TCGOp) link;   //链表指针

    TCGRegSet output_pref[2];   //输出寄存器偏好（优化过程使用）

    TCGArg args[];          //变长参数指针
};
```

以`trans_add()`为例，整体调用链如下
```
trans_add()
  → gen_arith(ctx, a, EXT_NONE, tcg_gen_add_tl, ...)
    → tcg_gen_add_tl(dest, src1, src2)       // include/tcg/tcg-op.h
      → tcg_gen_add_i64(dest, src1, src2)    // 64位平台
        → tcg_gen_op3_i64(INDEX_op_add, dest, src1, src2)  // tcg/tcg-op.c
          → tcg_gen_op3(INDEX_op_add, arg0, arg1, arg2)    // tcg/tcg.c
            → tcg_emit_op(opc, 3, args)
```
`tcg_emit_op()`分配`TCGOp`节点（包括操作数`TCGArg`的空间），设置opc，然后添加到ctx链表上。`tcg_gen_op3()`负责设置操作数类型和操作数。
```c
TCGOp *tcg_emit_op(TCGOpcode opc, unsigned nargs)
{
    TCGOp *op = tcg_op_alloc(opc, nargs);

    if (tcg_ctx->emit_before_op) {
        QTAILQ_INSERT_BEFORE(tcg_ctx->emit_before_op, op, link);
    } else {
        QTAILQ_INSERT_TAIL(&tcg_ctx->ops, op, link);
    }
    return op;
}

TCGOp * NI tcg_gen_op3(TCGOpcode opc, TCGType type, TCGArg a1,
                       TCGArg a2, TCGArg a3)
{
    TCGOp *op = tcg_emit_op(opc, 3);
    TCGOP_TYPE(op) = type;
    op->args[0] = a1;
    op->args[1] = a2;
    op->args[2] = a3;
    return op;
}
```

这里也是完成实验需要编写功能代码的位置，我们需要在自定义指令的trans函数中实现指令功能，不过考虑到大部分指令都涉及到向量运算，在不使用`tcg_gen_gvec`的情况下用IR的方式实现时，就像是用汇编写复杂的循环逻辑，并不方便开发，代码也很难理解，因此还是推荐使用Helper的方式。

在讲义里已经通过`cube`的例子告诉我们通过helper实现一个函数只需要三个步骤：
1. `DEF_HELPER`宏注册帮助函数
2. 编写`helper_XXX`函数
3. 在`trans_XXX`函数中调用`gen_helper_XXX`函数

看起来很简单？但是我当时对这个流程产生了一些困扰，比如`DEF_HELPER`宏具体做了什么？为什么它看起来存在三个不同的展开？比如`gen_helper`是什么时候定义的？为什么是`gen_helper`而不是`helper`？

由于Helper生成IR的额外复杂性，这里聚焦实现某个特定指令走一下整个流程。

##### HELPER注册
在`target/riscv/helper.h`中使用`DEF_HELPER`宏注册对应函数，这也是讲义中的内容。但我们需要找到它的展开形式
```c
DEF_HELPER_4(dma, void, env, tl, tl, tl)

//include/exec/helper-head.h.inc
#define DEF_HELPER_4(name, ret, t1, t2, t3, t4) \
    DEF_HELPER_FLAGS_4(name, 0, ret, t1, t2, t3, t4)
```
而随后事情便变得有意思起来，直接搜索字符串，能够找到三个不同的宏展开
```c
// include/exec/helper-info.c.inc
#define DEF_HELPER_FLAGS_4(NAME, FLAGS, RET, T1, T2, T3, T4)            \
    TCGHelperInfo glue(helper_info_, NAME) = {                          \
        .func = HELPER(NAME), .name = str(NAME),                        \
        .flags = FLAGS | dh_callflag(RET),                              \
        .typemask = dh_typemask(RET, 0) | dh_typemask(T1, 1)            \
                  | dh_typemask(T2, 2) | dh_typemask(T3, 3)             \
                  | dh_typemask(T4, 4)                                  \
    };

// include/exec/helper-proto.h.inc
#define DEF_HELPER_FLAGS_4(name, flags, ret, t1, t2, t3, t4) \
dh_ctype(ret) HELPER(name) (dh_ctype(t1), dh_ctype(t2), dh_ctype(t3), \
                            dh_ctype(t4)) DEF_HELPER_ATTR;

// include/exec/helper-gen.h.inc
#define DEF_HELPER_FLAGS_4(name, flags, ret, t1, t2, t3, t4)            \
extern TCGHelperInfo glue(helper_info_, name);                          \
static inline void glue(gen_helper_, name)(dh_retvar_decl(ret)          \
    dh_arg_decl(t1, 1), dh_arg_decl(t2, 2),                             \
    dh_arg_decl(t3, 3), dh_arg_decl(t4, 4))                             \
{                                                                       \
    tcg_gen_call4(glue(helper_info_,name).func,                         \
                  &glue(helper_info_,name), dh_retvar(ret),             \
                  dh_arg(t1, 1), dh_arg(t2, 2),                         \
                  dh_arg(t3, 3), dh_arg(t4, 4));                        \
}
```

同一个用于描述Helper函数的宏，被分别展开为三种不同的东西，第一个用于生成供QEMU使用的元数据`TCGHelperInfo`、第二个用于定义Helper函数本身的函数原型、第三个则是定义`gen_helper`，用于添加IR指令。

对于dma指令而言，会展开成如下三个东西

**`HelperInfo`**
```c
TCGHelperInfo helper_info_dma = {
    .func     = helper_dma,
    .name     = "dma",
    .flags    = 0,
    .typemask = dh_typemask(void,0)   // dh_typecode_void << 0  = 0 << 0 
        | dh_typemask(env,1)       // dh_typecode_ptr  << 3  = 6 << 3
        | dh_typemask(tl,2)        // dh_typecode_i64  << 6  = 4 << 6
        | dh_typemask(tl,3)        // dh_typecode_i64  << 9  = 4 << 9
        | dh_typemask(tl,4)        // dh_typecode_i64  << 12 = 4 << 12  
}; 
```
`typecode`通过宏定义在`include/exec/helper-head.h.inc`，总共六种，一个类型占3b，编码在typemask中。

**`HelperProto`**
```c
void helper_dma(CPUArchState *, target_ulong, target_ulong, target_ulong) __attribute__((noinline));
```

**`HelperGen`**
```c
extern TCGHelperInfo helper_info_dma; 
static inline void gen_helper_dma(TCGv_ptr arg1, TCGv_i64 arg2, 
                                  TCGv_i64 arg3, TCGv_i64 arg4) 
{
      tcg_gen_call4(helper_dma,                      // func
                    &helper_info_dma,                // info 
                    NULL,                            // retvar (void → NULL)
                    tcgv_ptr_temp(arg1),             // env → (TCGTemp*)arg1
                    tcgv_i64_temp(arg2),             // tl  → (TCGTemp*)arg2  
                    tcgv_i64_temp(arg3),             // tl  → (TCGTemp*)arg3 
                    tcgv_i64_temp(arg4));            // tl  → (TCGTemp*)arg4
}
```
在`gen_call`函数中会根据`HeleprInfo`填入恰当的参数（除了输入输出外还有`helper_dma`与`helepr_info_dma`的地址），创建一个`INDEX_op_call`IR。

#### TCG IR 优化

优化过程在生成二进制的`tcg_gen_code()`函数内，在代码生成前执行一系列优化pass。

```c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb, uint64_t pc_start)
{
    int i, num_insns;
    TCGOp *op;

    // IR 优化
    tcg_optimize(s);        // Pass 1 主要优化
    reachable_code_pass(s); // Pass 2 死代码消除
    liveness_pass_0(s);     // Pass 3 活性分析（全局）
    liveness_pass_1(s);     // Pass 4 活性分析（局部）

    if (s->nb_indirects > 0) {  // Pass 5 活性分析（间接）
        /* Replace indirect temps with direct temps.  */
        if (liveness_pass_2(s)) {
            /* If changes were made, re-run liveness.  */
            liveness_pass_1(s);
        }
    }

    // 寄存器分配与后端编码
    ...
    ...
}
```

不过在说明优化前，需要对TCG的TCGTemp寄存器进行说明，在之前流程中的`TCGv`、`TCGArg`是`TCGTemp`的不同表示
- `TCGv`: 结构体相对于ctx的偏移量（注意并非ctx.temps中的下标，这是为了保留0作为NULL的语义）
- `TCGArg`: 结构体的地址
- `TCGTemp`: 实际结构体
```c
typedef struct TCGTemp {
    TCGReg reg:8;           
    TCGTempVal val_type:8;  
    TCGType base_type:8;
    TCGType type:8;         
    TCGTempKind kind:3;
    unsigned int indirect_reg:1;
    unsigned int indirect_base:1;
    unsigned int mem_coherent:1;
    unsigned int mem_allocated:1;
    unsigned int temp_allocated:1;
    unsigned int temp_subindex:2;

    int64_t val;
    struct TCGTemp *mem_base;
    intptr_t mem_offset;
    const char *name;

    /* Pass-specific information that can be stored for a temporary.
       One word worth of integer data, and one pointer to data
       allocated separately.  */
    uintptr_t state;
    void *state_ptr;
} TCGTemp;
```

在优化阶段，只需要关注
- `kind`: 临时寄存器活性边界
- `val_type`: 值位置CONST/REG/MEM
- `type`: 数据位宽
- `mem_base`: 间接依赖
- `mem_offset`: 间接依赖
- `indirect_reg`: 间接依赖，触发判断
- `state`: TS_DEAD TS_MEM 
- `state_ptr`: 多语义字段,pass1 `TempOptInfo`;pass3 EBB指针; pass4 TCGRegSet偏好; pass5 直接temp映射

##### `tcg_optimize`
此pass流程为正向遍历IR，处理复制传播，并根据指令码派发到不同的处理函数进行常量折叠。
- 复制传播：检查当前操作数的复制链，按照优先级依次寻找先前定义的常量、全局变量、TB内变量、EBB内变量，并修正Op的arg字段指向。
- 常量折叠：当操作数均为常量时，直接预计算结果提供下游指令使用
```
foreach op in s->ops (正向遍历):
    ├─ opc == INDEX_op_call  →  fold_call(ctx, op)
    │   (helper 调用: 处理读写全局/副作用标志位)
    └─ 其他:
        ├─ init_arguments(op)      ← 为参数懒初始化 TempOptInfo
        ├─ copy_propagate(op)      ← 复制传播: 改写 args[]
        └─ switch(opc):
            ├─ add/sub/and/or/xor/...  → fold_*()
            ├─ set_label/br/exit_tb    → finish_ebb()
            └─ default                 → finish_folding()
```

##### `reachable_code_pass`

此pass负责死代码消除，在这个pass中会根据标签和跳转指令删除所有冗余或不可达的代码段，主要有以下四种形式：
- 无条件跳转后的不可达代码
- 冗余标签
- 优化产生的空跳转
- 孤儿标签

```asm
// 无条件跳转后的不可达代码
add x5, x10, 3
br label1           // 无条件跳转
add x6, x5, 1       // 被消除
set_label label2    // 下一个标签
add x7, x7, 1

// 冗余标签
set_label label1    
set_label label2    //将所有跳转到label1的分支指令跳转到后者，然后删除label1

// 空跳转
br label1           // 移除无用的br
set_label label1

// 
set_label label_no_use  // 删除没有被引用的标签
add x7, x7, 1           // 和之后的代码
```

##### 活性分析

活性分析的这几个pass主要是为了更细致的标注寄存器生命周期，为生成代码时更好的分配寄存器提供信息。
- `liveness_pass_0`: 临时寄存器生命周期缩减。扫描生命周期定义在整个TB上的`TEMP_TB`，如果它们只在单个EBB内使用，则降级为`TEMP_EBB`，以便提前回收可用的寄存器资源。
- `liveness_pass_1`：反向遍历TB，对每个使用到的临时寄存器，分析其是否存活，如果不再有用则可以释放。
- `liveness_pass_2`：间接临时寄存器，间接访存，显式展开为`ld/st`操作。

其中`liveness_pass_1`，尽管其逻辑可以简单的总结为以下处理方式，但是对于特殊情况需要细致的判断，比如特定架构中进位`carry`是隐含状态、分支跳转、函数调用会导致寄存器使用的一致性被破坏因而需要存入内存等等。这里不过多深入。
```
处理输出:
  if temp.state & TS_DEAD  → arg_life |= DEAD_ARG  (无人读取)
  if temp.state & TS_MEM   → arg_life |= SYNC_ARG  (需写回)
  temp.state = TS_DEAD      (重置: 向前 (向上游) 还不知道谁会用)

处理输入:
  if temp.state & TS_DEAD  → arg_life |= DEAD_ARG  (这是最后一次使用)
  清除 temp.state 的 TS_DEAD (它被当前 op 读取, 对上游来说是活跃的)
```

#### 代码生成

目标代码同样是架构特定的，QEMU在这里使用了基于函数指针表的分派架构，每个操作码对应一个TCGOutOp*结构，包含指向特定变体的函数指针：
```c
// tcg/tcg.c:1159
static const TCGOutOp * const all_outop[NB_OPS] = {
    [INDEX_op_add]  = &outop_add.base,
    [INDEX_op_sub]  = &outop_sub.base,
    [INDEX_op_mul]  = &outop_mul.base,
    // ... 所有操作码 ...
};
```

每个后端（如 tcg/x86_64/）提供实现这些函数指针的结构体：
```c
// tcg/x86_64/tcg-target.c.inc:2408
static const TCGOutOpBinary outop_add = {
    .base.static_constraint = C_O1_I2(r, r, re),  // 输出: 寄存器, 输入: 寄存器, 寄存器/常量
    .out_rrr = tgen_add,    // reg + reg
    .out_rri = tgen_addi,   // reg + imm
};
```

---

### GPU实验

GPGPU实验的基础部分的目标是在QEMU中实现一个简单的GPGPU设备，核心工作包含两个部分：
- 设备建模：实现符合PCI规范的GPU设备，提供MMIO寄存器接口
- 内核模拟：实现一个RISCV SIMT解释器，支持完整的内核执行功能，这里参照NEMU实现。

设备模拟部分相对而言不算复杂，反而是内核模拟部分为了读懂NEMU花了不少功夫，感觉又学了一遍怎么写译码（）

#### GPU设备

实验提供的代码已经将设备注册、初始化等代码完成，向设备注册了三个BAR，并提供了若干BAR的读写回调函数框架，只需要按照功能逐个完成。

| BAR | 大小 | 用途 |
|---|---|---|
| BAR0 | 1 MB | 控制寄存器（设备信息、全局控制、中断、内核调度、DMA、SIMT 上下文） |
| BAR2 | 64 MB | 显存（VRAM），存放内核代码与数据 |
| BAR4 | 64 KB | 门铃寄存器（预留） |

由于BAR0的控制寄存器繁多，在`gpgpu_ctrl_write/read`中进行派发，由单独的函数处理各个不同的功能。

| 回调组 | 偏移范围 | 职责 |
|---|---|---|
| dev_info | `0x0000` | 只读：设备 ID、版本号、能力位掩码、VRAM 大小 |
| global_ctrl | `0x0100` | 全局使能/复位；STATUS 反映设备忙闲状态，DISPATCH 后置 BUSY，完成恢复 READY |
| irq_ctrl | `0x0200` | 中断使能、挂起状态、ACK 清除；内核完成和 DMA 完成时触发 MSI |
| kernel_dispatch | `0x0300` | 内核地址（64 位）、Grid/Block 维度、共享内存大小；写 DISPATCH 触发内核执行 |
| dma_ctrl | `0x0400` | DMA 源/目标地址、传输大小；写 CTRL 启动双向拷贝（host ↔ VRAM），完成时更新 STATUS |
| simt_context | `0x1000` | 线程坐标（thread_id / block_id / warp_id / lane_id），内核执行时硬件自动填入 |
| sync_ctrl | `0x2000` | barrier 同步、活跃线程掩码 |

- **DISPATCH 回调**是设备层和执行引擎的唯一耦合点。写 DISPATCH 时校验内核地址是否在 VRAM 范围内，然后调用 `gpgpu_core_exec_kernel()`，状态寄存器同步置为 BUSY；执行完成恢复 READY 并触发中断。

- **全局控制寄存器**的 RESET 位写入时触发软复位：清空所有 SIMT 上下文寄存器（thread_id、block_id 等归零），复位完成后自动清除 RESET 位。ENABLE 位控制设备上电，写入后状态置 READY。

#### GPU内核模拟

##### 执行模型

执行模型借鉴CUDA的三级线程层次：
```
Grid
├── Block(0,0)                Block(0,1)
│   ├── Warp 0                ├── Warp 0
│   │   ├── Lane 0            │   ├── Lane 0
│   │   ├── Lane 1            │   ├── Lane 1
│   │   └── ... (最多 32)      │   └── ...
│   └── Warp 1                └── Warp 1
└── Block(1,0)                ...
```

- **Grid**: 最大三维 `(Nx, Ny, Nz)`，决定 Block 总数
- **Block**: 最大三维 `(Bx, By, Bz)`，线程数 = Bx × By × Bz
- **Warp**: 固定 32 Lane，是硬件调度和执行的基本单元
- **Lane**: 单个线程，拥有独立的寄存器文件（32 个 GPR + 32 个 FPR）

##### NEMU风格的指令解释器

全套解码/执行框架复用 NEMU 的设计模式，由三部分组成：

**指令列表（X-Macro）**

在 `isa-all-instr.h` 中把所有指令分入三个宏列表——`INSTR_BINARY`（加载存储、浮点转换）、`INSTR_TERNARY`（ALU、分支跳转、浮点运算）、`INSTR_TERNARY_CSR`（CSR 访问）。新增指令只需在对应列表中加一行，`def_all_EXEC_ID()` 自动生成执行表索引，`def_all_THelper()` 自动生成分发表桩函数。

**两级解码**

第一级按 opcode 分发到格式解码器 `DHelper(I/R/S/B/U/J/csr/fp)`。每个 `DHelper` 把操作数绑定到 `Decode` 结构的 `src1`/`src2`/`dest` 字段——`src1` 和 `src2` 通过 `preg` 指针指向寄存器值，`dest` 指向目标寄存器地址。x0 寄存器通过返回静态零变量地址实现零值只读语义，浮点寄存器则通过 `decode_op_fr` 单独处理。

第二级在格式内根据 `funct3`/`funct7`/`funct5` 进一步分发到具体指令的执行函数。FP 指令采用多层分发链：`opcode → op_fp → funct7（区分 bf16/fp8/fp4）→ rs2（区分转换方向）→ 执行函数`。分发最终返回 `EXEC_ID`，在全局 `g_exec_table` 查表获得 `EHelper` 函数指针。

**EHelper 执行函数**

每条指令对应一个 `def_EHelper(name)` 函数，通过 `Decode` 中的操作数指针直接读写寄存器。浮点指令在执行前调用 `set_rm()` 从指令的 `rm` 位段设置 softfloat 舍入模式，执行后调用 `FP_POSTOP` 将 softfloat 异常标志同步回 Lane 的 `fcsr`。

##### SIMT执行循环

`gpgpu_core_exec_warp` 是主循环：

1. 每个时钟周期从 lane 0 的 PC 取指（Warp 内锁步执行同一指令）
2. 遍历所有活跃 Lane：先解码再执行，`isa_fetch_decode` 完成后将 `EHelper` 写入 `Decode`，执行时通过操作数指针直接读写该 Lane 的寄存器
3. 每条指令完成后更新 PC 和 `active_mask`
4. `ebreak` 指令将当前 Lane 的 `active` 置 false
5. 循环在 `active_mask` 全零或超过 100000 周期时终止

`gpgpu_core_exec_kernel` 作为外层调度，通过 `FOR_EACH_3D` 宏在 Grid 维度上遍历所有 Block，每个 Block 内再遍历 Warp，逐 Warp 初始化和执行。

```c
#define FOR_EACH_3D(_i, _x, _y, _z, _dim) \
    for (uint32_t _i = 0, \
         _total = (_dim)[0] * (_dim)[1] * (_dim)[2]; \
         _i < _total; _i++) \
        for (uint32_t _z = _i / ((_dim)[0] * (_dim)[1]), \
                      _tmp = _i % ((_dim)[0] * (_dim)[1]), \
                      _y = _tmp / (_dim)[0], \
                      _x = _tmp % (_dim)[0], \
                      _once = 1; \
             _once; _once = 0)
```

##### 低精度浮点实现

新增 8 条自定义 RISC-V 指令，覆盖 bf16 / e4m3 / e5m2 / e2m1 四个格式的各两个方向（f32 -> LP 和 LP -> f32）：

| 指令 | funct7 | 语义 |
|---|---|---|
| `fcvt.s.bf16` / `fcvt.bf16.s` | 0x22 | bf16 <---> f32，rs2=0/1 区分方向 |
| `fcvt.s.e4m3` / `fcvt.e4m3.s` | 0x24 | e4m3 <---> f32，rs2=0/1 区分方向 |
| `fcvt.s.e5m2` / `fcvt.e5m2.s` | 0x24 | e5m2 <---> f32，rs2=2/3 区分方向 |
| `fcvt.s.e2m1` / `fcvt.e2m1.s` | 0x26 | e2m1 <---> f32，rs2=0/1 区分方向 |

所有指令均使用 `OP-FP` (0x53) 操作码，`funct7` 区分格式族，`rs2` 字段在族内区分方向。

E2M1 因 softfloat 无原生支持，自实现了一个基于 8 值查找表加中点舍入的转换函数 `float32_to_float4_e2m1`。其他格式的转换借助 softfloat 的 `bfloat16` 和 `float8` 系列函数，通过 `bfloat16_to_float32` 作为中间桥接完成 LP -> F32 的扩展。

> 碎碎念：一开始被AI误导softfloat没有支持低精度，越写越复杂，后面突然意识到自己在造轮子，又仔细检查后发现只需要实现e2m1的处理。。。

##### 访存与线程上下文

Lane 的加载/存储指令直接通过 `memcpy` 读写 GPGPU 的 VRAM 区域，在 `vram_store` 中增加地址越界检查。内核需要获取线程坐标时，通过加载 `0x80000000` 以上的地址来实现——`vram_load` 中硬编码了该区间的地址译码，直接返回 SIMT 上下文字段（thread_id、block_id、block_dim、grid_dim 等），使内核以普通 `lw` 指令即可获取线程坐标信息。

## 进阶实验
TODO

## 总结

通过完成CPU和GPU实验，在阅读QEMU源码的过程中，我对于QEMU本身的理解更加深入。在实现GPU内核的过程中，对于GPU的计算模型有了更细致的理解。