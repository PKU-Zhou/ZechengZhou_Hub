
仿真实际上不仅仅是前端的需求。在流片时，后端的多个节点都需要仿真。
粗略分类：
1. sim_pre_syn：综合前仿真；
2. sim_post_syn：综合后仿真；
3. （可能存在的）sim_with_xxx_pnr：模块xxx已经完成了后端，但是其他部分还未完成，想要先检查一下功能。可以选择用xxx模块的后端网表，与其余部分的前端RTL联合仿真。
4. sim_post_pnr_nosdf：后端完成后的无SDF仿真；
5. sim_post_pnr_sdf:后端完成后的SDF反标仿真（耗时通常较长）。

实际上，3是不推荐的。子模块和顶层都必须有单独的仿真环境。如果子模块没有单独的仿真环境，就只能选择3了。***偷懒并不能节约时间。***

### 仿真环境中需要什么？
参考：
```bash
|- testbench/
|- sim_xxx
   |- data/
      |- in_data/
      |- out_data/
      |- ref_out/
   |- waveform/
   |- filelists/
   |- sdf/
   |- Makefile
|- sim_yyy
   |- data/
      |- in_data/
      |- out_data/
      |- ref_out/
   |- waveform/
   |- filelists/
   |- sdf/
   |- Makefile
|- sim_zzz
   |- data/
      |- in_data/
      |- out_data/
      |- ref_out/
   |- waveform/
   |- filelists/
   |- sdf/
   |- Makefile
```