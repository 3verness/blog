---
title: "PYNQ学习·第一个IP核"
slug: "pynq-1-verilog-ip"
subtitle: "基于 AXI4 接口的 Verilog IP 核开发教程"
defcription: ""
tags:
    - "pynq"
    - "fpga"
    - "vivado"
    - "tutorial"
date: 2023-03-14T12:21:12+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

最近我开始学习PYNQ板和Vivado开发流程，与之前接触的Quartus开发流程不同，Vivado开发主要集中在IP核的设计和集成上，通过IP核的调用连接实现目标功能。在查阅资料的过程中，我发现大部分PYNQ的入门教程都使用HLS高级语言综合作为起步项目，而基于Verilog等RTL原语开发的教程较少。此外，一些教程已经过时，不再适用于当前的软件版本。因此，我写下这篇文章，记录我作为新手小白的学习经验。

{{<note success>}}本文教程适用于Vivado 2022.1，并在搭载v3.0.1镜像的PYNQ-Z1板上验证可用。{{</note>}}

## 前期准备

关于PYNQ板的连接和启动，官网已有[详细说明](https://pynq.readthedocs.io/en/latest/getting_started/pynq_z1_setup.html)，这里不再赘述。

为在Vivado中导入PYNQ Z1开发板，需要先下载[开发板信息文件](https://raw.githubusercontent.com/cathalmccabe/pynq-z1_board_files/master/pynq-z1.zip)，并将其放置在`{Vivado Dir}\data\xhub\boards\XilinxBoardStore\boards\Xilinx`目录下。

{{<note info>}}如果使用2021.2或更早版本的Vivado，板信息文件则存放在`{Vivado Dir}\data\boards`目录下。{{</note>}}

## 创建IP模板

在Vivado工程中选择工具栏中的`Tools->Create and Package New IP`，在弹出的窗口中使用`Create a new AXI4 Peripheral`创建一个带有AXI4接口的IP核模板，并配置接口参数，这里以功能较为简单的AXI4 Lite接口作为演示，最后选择`Edit IP`打开生成的IP核工程，如果选择了其他选项，可以从Project Manager中的IP Catalog找到该工程，右键选择`Edit in IP Packager`打开该IP核。

![image-20230313190958912](https://img.ioyoi.me/202303141228052.webp)

![image-20230313190917622](https://img.ioyoi.me/202303141228394.webp)

模板工程中含有两个文件，其中`project.v`文件中定义了AXI总线的接口并例化了一个支持AXI总线通信功能的模块，`project_AXI.v`文件具体实现了AXI总线功能，为了快速入门，我们重点关注其实现用户逻辑功能的寄存器操作部分，其中，最核心的寄存器是四个带有`slv`前缀的寄存器（寄存器数量由之前接口设置中的`Number of Registers`决定）与输出数据的`reg_data_out`，模板实现的默认逻辑为：当向地址`X`写入数据时，数据会保存在对应的`slv_regX`中，而从地址`X`读出数据时，会读取到`slv_regX`中的数据；因此我们只需要对这些寄存器进行操作就可以借由模板中的方法实现通信的功能。

```verilog
	...	
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg0;
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg1;
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg2;
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg3;
	...
	reg [C_S_AXI_DATA_WIDTH-1:0]	 reg_data_out;
	...
```

作为从机，在写逻辑中，会根据地址将数据存入对应的`slv_reg`寄存器，通过将switch case内`slv_reg`改为其他寄存器，可以改变写入寄存器的位置，不过在模板中`slv_reg`仅用于发送与接收，为避免错误，建议不要直接修改模板内容，而在其他时序逻辑中转存其值。另外注意这里的default语句理论上是不会运行的，因此更改此处的赋值左寄存器并不会将写入数据储存到新位置处。

```verilog
	always @( posedge S_AXI_ACLK )
	begin
        if ( S_AXI_ARESETN == 1'b0 )...
	  else begin
	    if (slv_reg_wren)
	      begin
	        case ( axi_awaddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB] )
	          2'h0:
	            for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
	              if ( S_AXI_WSTRB[byte_index] == 1 ) begin
	                slv_reg0[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
	              end
	          2'h1:...
	          2'h2:...
	          2'h3:...
	          default : begin
	                      slv_reg0 <= slv_reg0;
	                      slv_reg1 <= slv_reg1;
	                      slv_reg2 <= slv_reg2;
	                      slv_reg3 <= slv_reg3;
	                    end
	        endcase
	      end
	  end
	end    
```

而在读逻辑中，首先当地址位变动时，会将指定寄存器数据读入`reg_data_out`中，并在接下来的读时序中将`reg_data_out`内数据送出。因此，可以修改switch case内赋值语句右值来将我们需要的寄存器数据通过指定地址读出。

```verilog
	assign slv_reg_rden = axi_arready & S_AXI_ARVALID & ~axi_rvalid;
	always @(*)
	begin
	      case ( axi_araddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB] )
	        2'h0   : reg_data_out <= slv_reg0;
	        2'h1   : reg_data_out <= slv_reg1;
	        2'h2   : reg_data_out <= slv_reg2;
	        2'h3   : reg_data_out <= slv_reg3;
	        default : reg_data_out <= 0;
	      endcase
	end

	always @( posedge S_AXI_ACLK )
	begin
	  if ( S_AXI_ARESETN == 1'b0 )
	    begin
	      axi_rdata  <= 0;
	    end 
	  else
	    begin    
	      if (slv_reg_rden)
	        begin
	          axi_rdata <= reg_data_out;
	        end   
	    end
	end    
```

## 编写IP逻辑

这里我们简单实现一个计算两数平均值的模块，

```verilog
	reg [31:0] average_reg;
	always @(posedge S_AXI_ACLK)
	begin
	   average_reg <= (slv_reg0 >> 1) + (slv_reg1 >> 1) + (slv_reg0 & slv_reg1 & 1) ;
	end
```

同时，将读逻辑中的`2'h2   : reg_data_out <= slv_reg2;`修改为`2'h2   : reg_data_out <= average_reg;`使计算结果可以通过访问`0b1000`地址读出。

编写完成，依次运行Synthesis、Implement，验证无误后，可以在`Package IP`页的`Review and Package`中选择`Re-Package IP`重新生成IP核。

![image-20230313201034505](https://img.ioyoi.me/202303141228649.webp)

## 调用IP核

返回原工程，在Block Design中加入封装好的IP核和ZYNQ7 PS，自动连线，Vivado会帮我们添加时钟和AXI控制器。

![image-20230313202403918](https://img.ioyoi.me/202303141228644.webp)

保存Design，在`Sources`中右键选择`Create HDL Wrapper`让Vivado自动生成一个含有设计例化的顶层文件，综合实现生成bit流。

![image-20230313202121696](https://img.ioyoi.me/202303141228083.webp)

## 上板验证

为在开发板上调用该IP核，需要使用Vivado生成三个文件：

1. `.bit`文件：该文件可视作可执行文件，在PYNQ板上通过`Overlay`导入，生成在`{Project Dir}\{Project}.runs\impl_1\{Top File}.bit`处；
2. `.hwh`文件：硬件描述文件，包含有对PYNQ各模块的配置，默认生成在`{Project Dir}\{Project}.gen/sources_1/bd/{Block Design}/hw_handoff/{Block Design}.hwh`处；
3. `.tcl`文件（可选）：早期版本的硬件描述文件，可读性较`.hwh`文件可读性更高，需要在工具栏`File->Export->Export Block Design`生成。

随后将`.bit`文件与`.hwh`文件改为同一名称，移动到PYNQ板内同一目录下，PYNQ默认将Overlay保存在`/home/xilinx/pynq/overlays`目录下。

{{<note info>}}

在早期版本中，需要将`.bit`文件与同名`.tcl`上传至PYNQ板，不需要`.hwh`文件。

{{</note>}}

PYNQ板中，PS层与PL层的交互通过内存映射实现，因此我们需要知到挂载IP核对应的内存基址，在`.tcl`文件中，找到内存映射部分，可以看到`average_0/S00_AXI`对应的基址为`0x43C00000`，掩码为`0x0000FFFF`。

```tcl
  # Create address segments
  assign_bd_address -offset 0x43C00000 -range 0x00010000 -target_address_space [get_bd_addr_spaces processing_system7_0/Data] [get_bd_addr_segs average_0/S00_AXI/S00_AXI_reg] -force
```

而在`.hwh`文件中，可以找到对应模块配置内的`PARAMATERS`部分，同样可以读出基址和地址范围：

```xml
<PARAMETERS>
        <PARAMETER NAME="C_S00_AXI_DATA_WIDTH" VALUE="32"/>
        <PARAMETER NAME="C_S00_AXI_ADDR_WIDTH" VALUE="4"/>
        <PARAMETER NAME="Component_Name" VALUE="design_1_average_0_11"/>
        <PARAMETER NAME="EDK_IPTYPE" VALUE="PERIPHERAL"/>
        <PARAMETER NAME="C_S00_AXI_BASEADDR" VALUE="0x43C00000"/>
        <PARAMETER NAME="C_S00_AXI_HIGHADDR" VALUE="0x43C0FFFF"/>
</PARAMETERS>
```

最后，在`jupyter notebook`中，通过`Overlay`加载`.bit`文件，通过`MMIO`读写调用我们的IP功能，演示效果如下：

![](https://img.ioyoi.me/202303141250845.webp)

为方便调用，我们可以为其封装一个接口，如：

```python
def avg(a, b):
    mmio = MMIO(0x43c00000, 0x10000)
    mmio.write(0x0, a)
    mmio.write(0x4, b)
    return mmio.read(0x8)
```

![image-20230314124949099](https://img.ioyoi.me/202303141249482.webp)
