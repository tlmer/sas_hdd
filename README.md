# SAS HDD — SystemC TLM 参考模型

SAS 硬盘（**SSP target**）的**帧级 SystemC 事务级模型**：收 COMMAND IU → 执行 SCSI 命令集
→ 回 DATA（长读自动分帧）/RESPONSE ✓；**自含可独立跑**（直驱 rx / 采 tx，无需 HBA）✓

- 语言/工具：**SystemC 2.3.4**（`SYSTEMC_HOME` 可覆盖）+ C++14
- 行为基准：SAS-5 SSP 帧布局 + 公开的 Linux `hisi_sas`/libsas 目标侧口径（**逐字节**，含边界例）
- 自带 **字节级自检台 `tb_hdd_scsi`（25 条判据）+ `regress.sh`**：`make && sh regress.sh` 一条命令见绿 ✓

## 快速开始

```sh
cd model && make && sh regress.sh     # 25 条判据全绿 ⇒ TB_HDD_SCSI PASS
```
单独跑：`./tb/tb_hdd_scsi`（退出码 0 = 全 PASS ✓）。

## 组成

```
model/hdd/   sas_hdd_tlm.{h,cpp}   模型本体（帧收发 + SCSI 命令集 + 内存 LBA 存储）
model/tb/    tb_hdd_scsi.cpp       字节级专项台（25 判据 ✓）
sw/          sas_hdd_regs.h        帧常量 + 自有 CSR 表（共同基准 ✓）
docs/        design.md（设计规格）· verification.md（验证方案）· 
```

## 命令集（判据逐条见 `docs/verification.md` ✓）

TUR｜INQUIRY（36B，vendor `-UME MIS`、product `-SAS-DHVM652`；**LUN≠0 ⇒ PQ=3**）｜
MODE SENSE（**64B**）｜REQUEST SENSE（**64B**）｜REPORT LUNS（**64B**）｜
READ CAPACITY(10/16)｜READ(6/10/16)（多帧、DataOffset 递增）｜WRITE｜START STOP / SYNC CACHE /
MODE SELECT / FORMAT｜未知 opcode ⇒ CHECK CONDITION ✓

## 与 HBA 侧配对（可选 ✓）

本盘可与 **[`tlmer/sas_hba`](https://github.com/tlmer/sas_hba)** 的 HBA/链路模型经帧级链路
成对跑**全链路**（`HBA ⇄ link ⇄ HDD + 内存`，7 条命令端到端 ✓）——联合顶层与联合文档见该仓 ✓。

## 声明

参考模型，**不替代 RTL/硅验证**；范围与明写简化见 `docs/design.md` ✓
