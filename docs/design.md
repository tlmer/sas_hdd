# SAS HDD TLM 模型 — 设计方案（HDD 侧自含版）

[2026-10-09 建] 本文 = HDD 侧交付件的**自含设计规格**；联合/全链路视角见
`../../../sas_hba/tlm/md/设计方案.md`（**两份互为镜像**，改动必须同步 ✓）。

---

## 1. 定位

SAS 硬盘（SSP **target**）**帧级 SystemC 事务级模型**：收 COMMAND IU → 执行 SCSI 命令集
→ 回 DATA（可多帧）/RESPONSE ✓；可单独测（直驱 rx/采 tx ✓），也可与 HBA 模型经
`sas_link_tlm` 成对跑全链路 ✓。

- 技术栈：SystemC 2.3.4 + C++14（无 AMBA-PV 依赖 ✓ —— HDD 侧无主机寄存器面）；
- 行为依据：本参考实现 HDD RTL 已参考平台兑现的行为 + 与 HBA 侧**逐字节对拍**的帧口径 ✓
  （出处逐条注在代码与 `sw/sas_hdd_regs.h` ✓）。

## 2. 接口

| 面 | 形式 | 说明 |
|---|---|---|
| 时钟/复位 | `clk` / `rst_n` | 全 `SC_METHOD@clk.pos`、每拍一步 ✓ |
| SSP 收 | `rx_data[32]/rx_valid/rx_sof/rx_eof/rx_type[4]` + `rx_ready`（本模型**恒 1** ✓）| 帧装配：SOF 拍锁存帧型 ✓ |
| SSP 发 | `tx_data/tx_valid/tx_sof/tx_eof/tx_type` + `tx_ready`（对端反压 ✓）| **held-valid**：推进与呈现**分拍** ✓（每 beat 一完整窗口 ✓，末拍窗口不可丢 ✗——实踩修正 ✓）|
| 配置 | `cfg_blocks`（构造参数，512B/块 ✓）| 存储容量 ✓ |
| 白盒 | `o_rx_frames/o_tx_frames/o_cmds_good/o_cmds_chk/o_lba_read/o_lba_write/o_last_opcode/o_last_status/o_dbg_state` | ⛔ 只 TB 用；**必须绑定**（每拍写 ✓）|

## 3. 帧口径（与 HBA 侧逐值一致 ✓，`sw/sas_hdd_regs.h` §1）

- 帧字节 k = dword 内 bits[8k+7:8k] ✓
- **COMMAND**（14 dw）：byte0=0x10；tag@**16-17**（be16 ✓）；LUN@**24-31**；CDB@**36-51** ✓
- **DATA**（6 dw 头 + ≤250 dw 载荷）：byte0=0x60；DW4={0,tag be16}；DW5=**DataOffset be32** ✓
- **RESPONSE**（12 dw）：byte0=0x40；DW4 tag（be16）；**byte35=STATUS**；DW10=SENSE LEN（照 RTL 现状恒 0 ✓）
- 帧型侧带：1=COMMAND 6=DATA 4=RESPONSE 5=XFER_RDY ✓

## 4. SCSI 命令集（行为逐条照 RTL/参考平台 ✓）

TUR｜INQUIRY（36B：byte4=**32** ✓、vendor `-UME MIS`、product `-SAS-DHVM652`；**LUN≠0 ⇒ PQ=3** ✓）｜
MODE SENSE(6/10)（**64B**：0x3F/08/12 ✓）｜REQUEST SENSE（**64B**：0x70 + 附加长度 0x0A ✓，可注入 key/ASC ✓）｜
REPORT LUNS（**64B**、byte3=0x08 ✓）｜READ CAPACITY(10/16)（末 LBA BE32/BE64 + 块长 BE32 ✓）｜
READ(6/10/16)（DATA-in；**长读自动分帧**、DataOffset 递增 ✓）｜WRITE(6/10/16)（立即 GOOD —— **不消费 DATA-out** ✗ 明写简化）｜
START STOP / SYNC CACHE / MODE SELECT / FORMAT（GOOD 无数据 ✓）｜未知 opcode ⇒ **CHECK CONDITION**（byte35=0x02 ✓，sense 经 REQ SENSE ✓）

## 5. ⛔ 明写简化

1. **WRITE 不落盘**（收到即 GOOD ✓）；无介质错误注入（键/ASC 可 TB 注入 ✓）、无 DIF/NCQ/STP ✓
2. 单命令在飞（无命令队列重排 ✓）；XFER_RDY 不发（读路径直发 DATA ✓ —— 与 RTL 现状同 ✓）
3. 无自有 CSR 面（RTL 有 AXI-Lite 寄存器表 ⇒ 见 `sw/sas_hdd_regs.h` §2 留档，本模型未实现 ✗）
4. 扇区级 ECC/坏道、DIF/PI 不建模 ✓

## 6. 版本

`SAS_HDD_TLM_VERSION_STR`（头文件 ✓）；**改 `sw/sas_hdd_regs.h` 或本文件 ⇒ 必须同步
HBA 侧 `sas_hba_regs.h` / 联合《设计方案》** ✓（两侧对拍红线 ✓）。
