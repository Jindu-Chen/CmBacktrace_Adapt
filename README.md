

## CmBacktrace_Adapt
CmBacktrace是针对Cortex-M内核芯片提供程序运行异常时的堆栈回溯工具/中间件，将其移植到单片机中，可以在程序异常时打印相应的故障信息包括函数调用链、变量值、寄存器值等。

此项目为32mcu对cmbacktrace的移植适配Demo

## 移植说明

### 源码工程添加

工程源码见`//cm_backtrace/`，本文以国民技术芯片之gcc开发环境为例，其它环境参考修改即可

添加头文件搜索路径：
`cm_backtrace/`

添加编译C源文件：
```
cm_backtrace/cm_backtrace.c
cm_backtrace/fault_test.c
```

添加汇编文件：
`cm_backtrace/fault_handler/gcc/cmb_fault.S`


### 工程适配

注释掉原有工程的`HardFault_Handler`函数，避免与`cm_backtrace/fault_handler/`内的`HardFault_Handler`函数冲突


添加宏定义，查看`cm_backtrace/cmb_def.h`文件，如下，其提供了对各编译器的支持，主要是获取不同编译环境下的 堆栈起始结束地址、代码段起始结束地址
```c
#if defined(__ARMCC_VERSION)
    /* C stack block name, default is STACK */
    #ifndef CMB_CSTACK_BLOCK_NAME
    #define CMB_CSTACK_BLOCK_NAME          STACK
    #endif
    /* code section name, default is ER_IROM1 */
    #ifndef CMB_CODE_SECTION_NAME
    #define CMB_CODE_SECTION_NAME          ER_IROM1
    #endif
#elif defined(__ICCARM__)
    /* C stack block name, default is 'CSTACK' */
    #ifndef CMB_CSTACK_BLOCK_NAME
    #define CMB_CSTACK_BLOCK_NAME          "CSTACK"
    #endif
    /* code section name, default is '.text' */
    #ifndef CMB_CODE_SECTION_NAME
    #define CMB_CODE_SECTION_NAME          ".text"
    #endif
#elif defined(__GNUC__)
    /* C stack block start address, defined on linker script file, default is _sstack */
    #ifndef CMB_CSTACK_BLOCK_START
    #define CMB_CSTACK_BLOCK_START         _sstack
    #endif
    /* C stack block end address, defined on linker script file, default is _estack */
    #ifndef CMB_CSTACK_BLOCK_END
    #define CMB_CSTACK_BLOCK_END           _estack
    #endif
    /* code section start address, defined on linker script file, default is _stext */
    #ifndef CMB_CODE_SECTION_START
    #define CMB_CODE_SECTION_START         _stext
    #endif
    /* code section end address, defined on linker script file, default is _etext */
    #ifndef CMB_CODE_SECTION_END
    #define CMB_CODE_SECTION_END           _etext
    #endif
#else
    #error "not supported compiler"
#endif
```
在GCC编译环境下，如果没有`_sstack`、`_estack`、`_stext`、`_etext`这些宏定义，则在链接文件补全即可。

在`xxx.ld`文件中，添加内容节选如下：
```
xxx
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector)) /* Startup code */
    . = ALIGN(4);
  } >FLASH

  _stext = .;
  
xxx

  .text :
  {
    xxx
    . = ALIGN(4);
  } >FLASH

    _etext = .;        /* define a global symbols at end of code */

  .bss :
  {
    xxx
  } >RAM

  _sstack = .;

  ._user_heap_stack :
  {
    . = ALIGN(4);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    _estack = .;
    . = ALIGN(4);
  } >RAM
```

**适配串口打印输出：**重定向printf函数

GCC环境重写`_write`函数，将数据通过串口发送出去。
```c
int _write(int fd, char* data, int len)
{
    drv_uart_write(DEV_UART1, (uint8_t *)data, len);    // 用户自行实现此接口
    return len;
}
```

**添加初始化调用：**

在执行初始化的用户文件内，添加头文件包含`#include "cm_backtrace.h"`，此处以`main.c`之`main`函数为例：

```
#include "cm_backtrace.h"
#define HARDWARE_VERSION               "V1.0.0"
#define SOFTWARE_VERSION               "V0.1.0"

extern void fault_test_by_unalign(void);
extern void fault_test_by_div0(void);

int main(void)
{
    xxx
    cm_backtrace_init("CmBacktrace", "", SOFTWARE_VERSION);

    // 调用如下函数，则为测试实际的Demo报错效果
    fault_test_by_div0();
    fault_test_by_unalign();
}

```

