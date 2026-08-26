# 开源 ASIC NPU RTL 仓库调研

> 调研日期：2026-08-25  
> 目标：寻找与 NVIDIA NVDLA、Google Coral NPU 类似，使用 Verilog/SystemVerilog、面向深度学习或大模型加速、具有较高社区认可度，并尽可能公开 ASIC 综合与布局布线流程的项目。

## 1. 核心结论

如果“时间不晚于 NVDLA”按字面理解为“公开时间不晚于 NVDLA 首次发布的 2017-09-25”，那么除 NVDLA 自身外，没有发现同时满足以下条件的第二个仓库：

- 原生 Verilog/SystemVerilog RTL；
- 面向深度学习模型加速的 NPU，而非普通 GPU 或课程项目；
- 面向 ASIC，并具有论文、后端实现或流片佐证；
- RTL、验证和工具链的开放程度较高；
- 具有较高 Star 和社区认可度。

如果实际含义是“不早于 NVDLA，即考察 2017 年之后的项目”，较值得研究的候选包括：

1. **SAURIA**：原生 SystemVerilog 和论文/22 nm 后端结果较完整，综合条件最均衡。
2. **RedMulE**：工程质量高，具有 PULP 生态和芯片项目背景。
3. **ITA**：直接面向 Transformer 多头注意力和整数 softmax。
4. **CGRA4ML**：公开 Cadence Genus/Innovus Tcl 流程，最适合关注 ASIC 工具流的研究。
5. **FEATHER**：ISCA 2024 项目，提供 ASIC 综合和布局布线报告，但未公开完整可复现后端脚本。

目前仍没有一个项目在完整 RTL、编译器、驱动、验证、性能模型、ASIC 脚本和社区规模方面全面达到 NVDLA。

## 2. PLENA 是否已经开源 RTL

截至 2026-08-25，**PLENA 没有公开可访问的 RTL**。

- [PLENA 主仓库](https://github.com/AICrossSim/PLENA)的 README 一方面声称项目包含完整 RTL，另一方面明确说明 `PLENA_RTL` 仍在开发，计划在 2026 年 8 月底开源。
- 主仓库 `.gitmodules` 指向 `AICrossSim/PLENA_RTL`，但该仓库当前返回 404，无法访问。
- 目前公开的 [PLENA Simulator](https://github.com/AICrossSim/PLENA_Simulator)是 transaction-level simulator，不是周期精确 RTL，不能用于逻辑综合。
- 主仓库约 14 Star，Simulator 约 24 Star，尚不符合“高 Star、受到社区充分验证”的要求。

因此，当前不能把 PLENA 归类为“RTL 已开源项目”。建议在 2026 年 8 月底以后重新检查，并重点确认：

- 是否公开全部 PE、互连、存储控制器和系统顶层；
- 是否具有明确的开源许可证；
- 是否包含 testbench、参考模型和可重复测试；
- 是否公开 SDC、综合 Tcl、SRAM macro 接口和 P&R 流程；
- 论文中的面积、频率和功耗能否由公开仓库复现。

## 3. 2017 年以后的主要候选

Star 数为调研时的近似值。硬件 RTL 项目的社区规模通常远小于软件项目，因此 80～150 Star 在学术 ASIC 加速器中已经不算低。

| 项目 | 约 Star | RTL与定位 | ASIC及论文证据 | 后端流程 | 综合判断 |
|---|---:|---|---|---|---|
| [SAURIA](https://github.com/bsc-loca/sauria) | 112 | 原生 SystemVerilog；CNN/GEMM systolic accelerator，包含 AXI、DMA、SRAM wrapper | [IEEE TVLSI 论文](https://www.research-collection.ethz.ch/handle/20.500.11850/623491)；22 nm post-layout，论文报告面积和能效 | 未发现公开可重跑的完整 P&R 脚本 | **原生 SV 项目中最推荐** |
| [RedMulE](https://github.com/pulp-platform/redmule) | 114 | 原生 SystemVerilog；FP16/FP8 GEMM/矩阵乘协处理器，包含 HAL 和验证 | [DATE 论文](https://arxiv.org/abs/2204.11192)；22 nm 实现，相关 IP 具有 PULP 芯片项目背景 | 有综合入口文件，但不是完整公开 P&R 流 | **工程质量及硅验证背景较强** |
| [FEATHER](https://github.com/maeri-project/FEATHER) | 90 | 原生 Verilog；灵活数据流 DNN 加速器，包含 MINISA/compiler 和 DSE 工具 | ISCA 2024 项目；仓库提供多个规模的 DC/Innovus 结果报告 | 有综合和布局布线报告；未发现可直接重跑的 DC/Innovus Tcl、SDC | **论文水平高，但公开计算部分不等于完整 SoC** |
| [ITA](https://github.com/pulp-platform/ITA) | 86 | 原生 SystemVerilog；Transformer MHA 和整数 softmax 加速器 | [ISLPED 论文](https://arxiv.org/abs/2307.03493)；22 nm FD-SOI post-layout 数据 | 公开仿真环境，未发现完整后端脚本 | **与大模型最直接相关，但属于注意力加速 IP** |
| [CGRA4ML](https://github.com/KastnerRG/cgra4ml) | 126 | 公开 SystemVerilog 硬件生成框架；支持 CNN、PointNet、Transformer | [项目论文](https://arxiv.org/abs/2408.15561)；同时面向 FPGA 和 ASIC | **包含 Cadence Genus/Innovus Tcl 流程** | **后端开放程度最好，但更接近 CGRA 框架** |
| [OpenEye](https://github.com/Learning-Chips-Lab/OpenEye) | 61 | 原生 Verilog；基于 EyerissV2 思路的稀疏 CNN 推理加速器 | 设计依据较充分，但缺少同等级独立芯片论文和流片证据 | 主要为 Yosys/nextpnr 和 FPGA 相关流程 | 可作为 RTL 参考，学术及 ASIC 证据较弱 |

### 3.1 SAURIA

- 原生 SystemVerilog，而不是 Chisel、Bluespec 或 Python 生成 RTL。
- 覆盖阵列计算、数据搬运、AXI 接口和 SRAM wrapper，完整度较高。
- 论文给出了 22 nm 后端实现和面积、性能、能效数据。
- 仓库没有发现完整公开的商业工具 P&R 脚本，因此论文结果不能直接一键复现。

适合：研究传统 CNN/GEMM systolic NPU 微架构、数据流和 RTL 实现。

### 3.2 RedMulE

- PULP 平台下的浮点矩阵乘加速器，使用原生 SystemVerilog。
- 支持 FP16/FP8 等格式，定位介于矩阵计算 IP 和嵌入式 NPU 之间。
- 具有论文、22 nm 实现数据和 PULP 芯片生态支持。
- 仓库包含构建、验证和部分综合入口，但不是完整开放的 floorplan-to-GDS 流程。

适合：研究低精度浮点矩阵乘、共享内存耦合和可集成 accelerator IP。

### 3.3 FEATHER

- ISCA 2024 项目，核心计算部分使用 Verilog。
- 关注灵活数据流及不同神经网络层的高利用率映射。
- 除 RTL 外还提供 MINISA、compiler 和设计空间探索工具。
- 仓库提供 Design Compiler、Innovus 产生的 ASIC 结果报告，但逐树检查未发现完整 `.tcl`、`.sdc` 后端流脚本。
- 公开描述主要是“完整片上计算部分”，不能直接等同于带主机接口、外部内存控制器和软件驱动的完整 NPU SoC。

适合：研究数据流重构、编译映射以及架构级设计空间探索。

### 3.4 ITA

- 原生 SystemVerilog，专门加速 Transformer 的多头注意力。
- 包括整数 softmax 等传统 CNN NPU 通常没有的模块。
- 论文提供 22 nm FD-SOI post-layout 结果。
- 不是独立、完整的 LLM NPU，通常需要集成到 PULP SoC 或其他系统中。

适合：研究 Transformer attention、softmax、低精度和 LLM 专用算子硬件。

### 3.5 CGRA4ML

- 面向深度学习的 CGRA/加速器生成框架，硬件部分公开 SystemVerilog。
- 支持 CNN、PointNet 和部分 Transformer 工作负载。
- 项目同时覆盖 FPGA 和 ASIC。
- 候选中后端开放程度最高，仓库包含 Cadence Genus 和 Innovus Tcl 流程。
- 缺点是其抽象层次和定位更接近可生成 CGRA，而不是固定微架构、NVDLA 式完整 NPU。

适合：需要研究 RTL 到综合、布局布线完整路径，或希望修改架构并重复获取 PPA 的场景。

### 3.6 NEureka：低 Star 但强芯片证据

[PULP NEureka](https://github.com/pulp-platform/neureka)只有约 38 Star，但学术和硅实现可信度较高：

- 使用原生 SystemVerilog；
- 面向稀疏卷积和低比特推理；
- 集成于 Siracusa 16 nm MRAM SoC；
- 具有[芯片论文](https://arxiv.org/abs/2312.14750)。

其主要不足是项目属于嵌入式 accelerator IP，不是独立完整 NPU，社区规模也不大。

## 4. 严格限定“不晚于 NVDLA”的结果

NVDLA 首次公开发布于 **2017-09-25**。根据[NVDLA 官方时间线](https://nvdla.org/updates.html)，第一次发布已经包含：

- large configuration RTL；
- trace-player testbench；
- synthesis scripts；
- 性能模型；
- 完整文档。

在这一时间点以前，较接近但不合格的项目包括：

### OpenTPU

[OpenTPU](https://github.com/UCSBarchlab/OpenTPU)约 777 Star，仓库创建于 2017 年 4 月，但存在以下问题：

- 源代码是 PyRTL/Python，Verilog 属于生成结果，不是原生开发语言；
- 没有完整 ASIC 综合与布局布线流程；
- 没有独立的芯片实现论文；
- 卷积、池化、归一化和存储控制等存在缺失或简化。

因此不满足本次筛选条件。

### DNNWeaver2

[DNNWeaver2](https://github.com/hsharma35/dnnweaver2)对应研究可追溯至 MICRO 2016，但公开的 v2 仓库创建于 2018 年，而且主要针对 FPGA/Vivado，不是 ASIC 开源流程。

### 其他早期项目

Eyeriss、DianNao、Origami、YodaNN/HWCE 等项目具有重要论文甚至真实流片，但没有发现同时满足以下条件的第一方仓库：

- 完整原生 Verilog/SystemVerilog RTL；
- 较高 Star；
- 当前可访问；
- 公开验证和 ASIC 后端流程；
- 足以复现实验或芯片结果。

MIAOW 虽然是早期、高 Star 的 Verilog 项目并有论文，但它是 AMD GPGPU 实现，不是 NPU，也缺少面向 AI 模型的完整编译和 ASIC 后端栈。

因此，在“发布时间不晚于 2017-09-25”的严格条件下，**基本只有 NVDLA 自身完全合格**。

## 5. Coral NPU 的语言和开放程度

[Google Coral NPU](https://github.com/google-coral/coralnpu)具有较高 Star 和组织背景，但不应被视为纯 Verilog 项目：

- 处理器核心、顶层和部分主要模块使用 Chisel/Scala；
- 向量后端等部分采用 SystemVerilog；
- 公开仓库中的工具流程主要偏 FPGA/Vivado；
- 未发现完整公开的 ASIC 综合、floorplan、CTS、routing 和 signoff 流程。

所以 Coral NPU 很适合作为现代开源 NPU ISA、向量架构和软件栈参考，但不完全符合“以 Verilog 为主要开发语言、ASIC 后端充分开放”的限定。

## 6. 不建议纳入严格候选的项目类型

| 项目或类型 | 排除原因 |
|---|---|
| Gemmini | 高质量、具有流片和论文，但主要使用 Chisel/Scala，不是原生 Verilog |
| Coral NPU | Chisel/Scala 与 SystemVerilog 混合，缺少公开 ASIC 后端流 |
| OpenTPU | PyRTL/Python 生成 Verilog，功能和 ASIC 流不完整 |
| MAERI | Bluespec SystemVerilog，不是原生 Verilog/SystemVerilog RTL |
| DNNWeaver2 | 主要面向 FPGA/Vivado |
| Vortex、MIAOW | GPU/GPGPU，不是专用 NPU |
| SwiftTron | 使用 VHDL，不符合开发语言要求 |
| PLENA | 截至调研日，RTL 子仓库尚未公开 |
| 普通 systolic-array 教学仓库 | 缺少论文、完整系统、ASIC 证据和社区验证 |

## 7. 最终选择建议

按不同研究目标，推荐如下：

| 研究目标 | 首选 | 次选 |
|---|---|---|
| 原生 Verilog/SystemVerilog NPU RTL | SAURIA | RedMulE |
| Transformer/大模型算子 | ITA | CGRA4ML |
| 可运行综合和布局布线脚本 | CGRA4ML | NVDLA 的综合流程 |
| 高水平论文和数据流研究 | FEATHER | SAURIA |
| 已有芯片项目或硅实现背景 | RedMulE、NEureka | SAURIA |
| NVDLA 级完整软硬件栈 | NVDLA | 暂无全面等价替代品 |

如果后续要实际选择仓库进行二次开发，建议优先对 **SAURIA、RedMulE、ITA、CGRA4ML、FEATHER** 做 clone 后的结构化审计，检查 RTL 模块数、顶层接口、SRAM 模型、testbench、综合约束、工具版本及论文结果的可复现性。

## 8. 主要链接

- [NVIDIA NVDLA](https://github.com/nvdla)
- [NVDLA Hardware](https://github.com/nvdla/hw)
- [NVDLA 官方发布时间线](https://nvdla.org/updates.html)
- [Google Coral NPU](https://github.com/google-coral/coralnpu)
- [PLENA](https://github.com/AICrossSim/PLENA)
- [PLENA Simulator](https://github.com/AICrossSim/PLENA_Simulator)
- [SAURIA](https://github.com/bsc-loca/sauria)
- [RedMulE](https://github.com/pulp-platform/redmule)
- [FEATHER](https://github.com/maeri-project/FEATHER)
- [ITA](https://github.com/pulp-platform/ITA)
- [CGRA4ML](https://github.com/KastnerRG/cgra4ml)
- [OpenEye](https://github.com/Learning-Chips-Lab/OpenEye)
- [NEureka](https://github.com/pulp-platform/neureka)
- [OpenTPU](https://github.com/UCSBarchlab/OpenTPU)
- [DNNWeaver2](https://github.com/hsharma35/dnnweaver2)
