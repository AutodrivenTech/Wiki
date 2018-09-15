<!-- TITLE: Build Sd So C Platform For Zcu 102 -->
<!-- SUBTITLE: A quick summary of Build Sd So C Platform For Zcu 102 -->

# 为ZCU102搭建SDSoC Platform
`SDSoC`包含了`ZCU102`的`SDSoC Platform`，该Platform提供了三种运行环境————`Standalone`，`FreeRTOS`，`Linux`。Platform提供的Linux只能用来运行SDSoC编译出来的elf文件，断电之后丢失全部数据，没有图形界面，也不能从板上接口(如USB)获取数据，功能十分有限。为了解决这个问题，我们要重新编译ZCU102的内核，使能USB接口等外设，并使用Ubuntu的rootfs而非Petalinux产生的rootfs。本Tutorial在2018.2下测试通过。


## Step1 搭建zcu102 SDSoC的Vivado Platform

### 新建一个Vivado工程命名为zcu102，新建一个bd命名为zcu102，按照下图搭建bd

![ZCU102的Vivado工程](images/1.png)
clk_wiz_0的输出时钟频率设置为75，100，150，200，300，400，600M

### 配置DP端口

双击PS进入Re-customize IP对话框，点击I/O Configuration> High Speed，取消勾选PCIe，使能Display Port并将Lane设置为Dual Lower如下图所示，配置完成后保
存Block Design

![配置DP端口](images/2.png)

### 输入SDSoC相关的配置属性
在Tcl中输入

```sh
set_property PFM_NAME "xilinx.com:zcu102:zcu102:1.0" [get_files zcu102.bd]
set_property PFM.CLOCK { \
clk_out1 {id "0" is_default "false" proc_sys_reset "proc_sys_reset_0" } \
clk_out2 {id "1" is_default "false" proc_sys_reset "proc_sys_reset_1" } \
clk_out3 {id "2" is_default "false" proc_sys_reset "proc_sys_reset_2" } \
clk_out4 {id "3" is_default "true" proc_sys_reset  "proc_sys_reset_3" } \
clk_out5 {id "4" is_default "false" proc_sys_reset "proc_sys_reset_4" } \
clk_out6 {id "5" is_default "false" proc_sys_reset "proc_sys_reset_5" } \
clk_out7 {id "6" is_default "false" proc_sys_reset "proc_sys_reset_6" } \
} [get_bd_cells /clk_wiz_0]

set_property PFM.AXI_PORT { \
M_AXI_HPM0_FPD {memport "M_AXI_GP"} \
M_AXI_HPM1_FPD {memport "M_AXI_GP"} \
M_AXI_HPM0_LPD {memport "M_AXI_GP"} \
S_AXI_HPC0_FPD {memport "S_AXI_HPC" sptag "HPC0" memory "ps_e HPC0_DDR_LOW"} \
S_AXI_HPC1_FPD {memport "S_AXI_HPC" sptag "HPC1" memory "ps_e HPC1_DDR_LOW"} \
S_AXI_HP0_FPD {memport "S_AXI_HP" sptag "HP0" memory "ps_e HP0_DDR_LOW"} \
S_AXI_HP1_FPD {memport "S_AXI_HP" sptag "HP1" memory "ps_e HP1_DDR_LOW"} \
S_AXI_HP2_FPD {memport "S_AXI_HP" sptag "HP2" memory "ps_e HP2_DDR_LOW"} \
S_AXI_HP3_FPD {memport "S_AXI_HP" sptag "HP3" memory "ps_e HP3_DDR_LOW"} \
} [get_bd_cells /ps_e]
set intVar []
for {set i 0} {$i < 8} {incr i} {
lappend intVar In$i {}
}
set_property PFM.IRQ $intVar [get_bd_cells /xlconcat_0]
set_property PFM.IRQ $intVar [get_bd_cells /xlconcat_1]
```

注意get_bd_cells/后面的名称要与BD中的名称对应

### 点击Generate Bitstream，完成后File > Export > Export Hardware，勾选Include Bistream然后点击OK， 然后Launch SDK，可以在<PATH\>/zcu102.sdk文件夹下看到.hdf文件和.bit文件
生成的hdf文件用于创建Petalinux工程，2017.4之前的Petalinux只能用hdf文件创建工程，2018.1只支持dsa文件，2018.2版本同时支持dsa和hdf文件

###将硬件工程打包成dsa文件

在Tcl中输入

```sh
write_dsa –force <path_to_project>/zcu102.dsa -include_bit
validate_dsa <path_to_project>/zcu102.dsa

```
## Step2 使用Petalinux编译Linux的内核以及根文件系统

```sh
$ petalinux-create -t project -s xilinx-zcu102-v2018.2-final.bsp --name zcu102
```
一定要使用bsp文件创建工程

```sh
$ cd zcu102/
```
将刚才生成的dsa或hdf文件复制到该目录下

```sh
$ petalinux-config --get-hw-description ./
```

导入硬件工程，此时会弹出设置界面，这里需要改一些配置，使得rootfs改为SD卡模式

DTG Settings->Kernel Bootargs->取消Generate boot args automatically，在user set kernel bootargs 输入
```sh
earlycon clk_ignore_unused earlyprintk root=/dev/mmcblk0p2 rw rootwait cma=1024M
```

Image Packaging Configurations->Root filesystem type->SD card

在`<PATH\>/project-spec/metauser/recipes-kernel/linux/linux-xlnx/bsp.cfg`添加

```sh
# CONFIG_BLK_DEV_INITRD is not set
CONFIG_XILINX_APF=y
CONFIG_XILINX_DMA_APF=y

CONFIG_CMA_SIZE_MBYTES=1024
CONFIG_STAGING=y

# CONFIG_CPU_IDLE is not set
# CONFIG_CPU_FREQ is not set
CONFIG_EDAC_CORTEX_ARM64=y
```

在`<PATH\>/project-spec/metauser/recipes-bsp/device-tree/files/system-user.dtsi`添加

```sh
/{
xlnk {
compatible = "xlnx,xlnk-1.0";
};
};

```

将0001-Fix-for-gtr_sel0-polarity-correct-for-dual-lane-DP.patch复制到与system-user.dtsi同路径，device-tree-generation_%.bbappend改为

```sh
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
SRC_URI += "file://system-user.dtsi\
            file://0001-Fix-for-gtr_sel0-polarity-correct-for-dual-lane-DP.patch"

```

编译工程

```sh
$ petalinux-build
```

在`<PATH\>/images/linux/`路径下可以看到生成的文件，将`u-boot.elf, bl31.elf, pmufw.elf, zynqmp_fsbl.elf`,  替换掉SDSoC自带的`zcu102/sw/a53_linux/boot`下的对应文件，注意，`zynqmp_fsbl.elf`需要改名为`fsbl.elf`；将`image.ub`替换掉`SDSoC`自带的`zcu102/sw/a53_linux/a53_linux/image`下的对应文件；将Step1产生的`.hdf`和`.bit`文件替换掉`SDSoC`自带的`zcu102/sw/a53_linux/a53_linux/prebuilt`下的对应文件，注意改名。



