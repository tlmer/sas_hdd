# SAS HDD TLM 模型 — 验证方案（HDD 侧自含版）

[2026-10-09 建] 口径照全工程统一纪律 ✓：**只认台自带终判行**（`TB_<NAME> PASS`）；
`regress.sh` 只收集/汇总、不重新解释判据 ✓；`make -q` **陈旧二进制守卫** ✓。
联合/全链路验证（HBA⇄link⇄HDD）见 `../../../sas_hba/tlm/md/验证方案.md` ✓。

---

## 1. 怎么跑

```sh
cd sas_hdd/tlm/model && make && sh regress.sh      # tb_hdd_scsi：25 条判据
./tb/tb_hdd_scsi                                    # 单跑；退出码 0 = 全 PASS ✓
```
环境变量：`SYSTEMC_HOME`（默认 `/opt/tools/systemc2.34` ✓）。

## 2. 台 `tb_hdd_scsi`（25 条，**预登记** ✓）

结构：**直驱** HDD 的 rx（逐拍送帧、同拍采 tx ✓），无 HBA/链路 ✓ —— 纯字节级核 ✓。

| 组 | 判据 |
|---|---|
| D1 | TUR：仅 1 帧 RESPONSE（12 dw、帧型 4 ✓）；dw0 byte0=0x40 ✓；**tag 回显 0x1234**（be16：byte16=0x12/byte17=0x34 ✓）；byte35=0x00 ✓ |
| D2 | INQUIRY：DATA 帧 6+9 dw ✓；byte0=0x60 ✓；载荷 **byte4=32** ✓；vendor/product 逐字 ✓；PQ=0 ✓；后随 RESPONSE ✓ |
| D3 | INQUIRY **LUN≠0** ⇒ 载荷 byte0 高 3 位 = 011b（**PQ=3** ✓）|
| D4 | MODE SENSE(6)：DATA 载荷 **64B** ✓；byte0=0x3F / byte4=0x08 / byte5=0x12 ✓ |
| D5 | READ CAPACITY(10)：载荷 8B ✓；**末 LBA = 255**（BE ✓ = 块数-1 ✓）；**块长 = 0x00000200**（BE @byte4-7 ✓）|
| D6 | REPORT LUNS：64B ✓；byte3 = 0x08（BE 长度 ✓）|
| D7 | READ(10) 2048B：**3 个 DATA 帧**（>1000B 自动分帧 ✓）；**DataOffset = 0/1000/2000**（be32 ✓）；**载荷 2048B 逐字节 == 存储模式** ✓；+ RESPONSE ✓ |
| D8 | 未知 opcode 0x7F：RESPONSE byte35 = **0x02**（CHECK CONDITION ✓）|
| D9 | START STOP / SYNC CACHE / MODE SELECT：RESPONSE GOOD、无 DATA ✓ |
| D10 | 计数：`o_cmds_good=10` / `o_cmds_chk=1`（11 条命令 ✓）；`o_lba_read=4` ✓ |

## 3. 判据纪律（与全工程一致 ✓）

1. **预登记**（期望值含出处写在台头 ✓）；2. **终判行**（`TB_HDD_SCSI PASS|FAIL` ✓）；
3. **每拍一步**（台逐拍驱动/采样 ✓）；4. **不采半拍值**（收采样经 `SC_ZERO_TIME` 稳定窗 ✓）；
5. **CDB/LBA 字节位置对**（READ10 的 LBA 大端：LSB 在 **byte5** ✓ —— 台曾写错位 ⇒ 读到 LBA 0x1000 全零 ✗ 实踩 ✓）；
6. **改动即重跑**（改模型/`sw` ⇒ 本台 + 联合台都得全绿才提交 ✓）。

## 4. 本轮抓出并修复的模型缺陷（记档 ✓）

| # | 缺陷 | 现象 | 修复 |
|---|---|---|---|
| M8a | DATA 帧「先入队后覆写载荷」✗ | INQUIRY 等载荷全 0 | 改「**先填齐头+载荷再入队**」✓ |
| M8b | 帧缓冲 64 dw 上限 ⇒ 长读被静默截断 ✗ | READ 512B 只剩 232B | 缓冲扩至 256 dw + **自动分帧**（DataOffset 递增 ✓）|
| M8c | RCAP10 按 64 位 BE 写 ✗ | 末 LBA 低字节错位到 byte7 | 照 RTL 改 **BE32@0-3 + 块长 BE32@4-7** ✓（台 D5 ✓）|
| M8d | INQUIRY byte4 写 31 ✗ | 与 RTL `INQ_STD_LEN-4=32` 不符 | 改 **32** ✓（台 D2 ✓）|
| M8e | 发送侧「推进与呈现同拍」✗ | 末拍窗口被清 ⇒ 对端收不到 EOF（全链路超时 ✗）| 改**分拍**（每 beat 一完整窗口 ✓）|

## 5. 回归快照（2026-10-09 ✓）

```
tb_hdd_scsi   PASS   [合计] PASS=25 FAIL=0      ⇒ 全部 PASS ✓ (1/1)
（联合：sas_hba/tlm 侧 3 台 45 条 ⇒ 全工程合计 70 条判据全绿 ✓）
```
