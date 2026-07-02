# FPGA FIR Filter IP Core Vivado 2023.1 trên PYNQ-Z2

README này mô tả toàn bộ dự án FIR Filter bằng IP Core trong Vivado 2023.1, gồm chuẩn bị từng khối IP, cách nối dây, chức năng từng tín hiệu chính, ý nghĩa input/output, công thức FIR và notebook Jupyter để kiểm tra trên PYNQ-Z2.

## 1. Thông tin dự án

| Mục | Giá trị |
|---|---|
| GitHub repo | `fpga-fir-filter-vivado-2023-1` |
| Vivado project | `fir_filter_ip_2023_1` |
| Block Design | `fir_filter_bd` |
| HDL Wrapper | `fir_filter_bd_wrapper` |
| Board | PYNQ-Z2 |
| FPGA part | `xc7z020clg400-1` |
| Vivado | 2023.1 |
| Bitstream | `fir_filter_ip_2023_1.bit` |
| HWH | `fir_filter_ip_2023_1.hwh` |
| LTX debug | `fir_filter_ip_2023_1.ltx` |
| Notebook | `test_fir_filter_dma.ipynb` |

## 2. Mục tiêu hệ thống

Luồng dữ liệu chính:

```text
DDR → AXI DMA MM2S → FIFO IN → FIR Compiler → FIFO OUT → Subset Converter → AXI DMA S2MM → DDR
```

Ý nghĩa:
- Jupyter tạo tín hiệu input `x[n]`.
- AXI DMA đọc input từ DDR và phát ra AXI4-Stream.
- FIR Compiler lọc tín hiệu theo vector hệ số.
- AXI DMA nhận output `y[n]` và ghi về DDR.
- Jupyter vẽ sóng input/output và FFT.
- System ILA quan sát waveform AXI4-Stream trong Vivado Hardware Manager.

## 3. Lý thuyết FIR Filter

Bộ lọc FIR có công thức:

```math
y[n] = \sum_{k=0}^{M-1} h[k]x[n-k]
```

Trong đó:
- `x[n]`: mẫu tín hiệu đầu vào.
- `y[n]`: mẫu tín hiệu đầu ra.
- `h[k]`: hệ số FIR.
- `M`: số tap.

Vector hệ số dùng trong dự án:

```text
h = [6, 0, -4, -3, 5, 6, -6, -13, 7, 44, 64, 44, 7, -13, -6, 6, 5, -3, -4, 0, 6]
```

Số tap:

```text
M = 21
```

Công thức đáp ứng tần số:

```math
H(e^{j\omega}) = \sum_{k=0}^{M-1} h[k]e^{-j\omega k}
```

Vì hệ số chưa normalize, output có thể lớn hơn input. Khi vẽ đồ thị nên có thêm bước chuẩn hóa để dễ nhìn.

## 4. Danh sách 10 IP chính

| STT | Instance | IP Core | Chức năng |
|---:|---|---|---|
| 1 | `zynq_ps` | ZYNQ7 Processing System | ARM PS, DDR, clock, reset, AXI |
| 2 | `axi_dma_0` | AXI DMA | Truyền dữ liệu DDR ↔ AXI4-Stream |
| 3 | `fir_compiler_0` | FIR Compiler | Bộ lọc FIR chính |
| 4 | `axi_smc_mem` | AXI SmartConnect | DMA truy cập DDR qua HP0 |
| 5 | `axi_smc_ctrl` | AXI SmartConnect | PS điều khiển DMA qua AXI-Lite |
| 6 | `proc_sys_reset_0` | Processor System Reset | Reset đồng bộ |
| 7 | `axis_fifo_in` | AXI4-Stream Data FIFO | FIFO trước FIR |
| 8 | `axis_fifo_out` | AXI4-Stream Data FIFO | FIFO sau FIR |
| 9 | `axis_subset_conv_0` | AXI4-Stream Subset Converter | Chuẩn hóa AXI4-Stream về DMA |
| 10 | `system_ila_axis_0` | System ILA | Debug interface AXI4-Stream |

IP phụ:
- `xlconstant_1b1 = 1`: nối vào `proc_sys_reset_0/dcm_locked`.
- `xlconstant_1b0 = 0`: nối vào `proc_sys_reset_0/aux_reset_in` và `mb_debug_sys_rst`.

> Lưu ý quan trọng: Không dùng ILA Native nối trực tiếp từng chân `tdata/tvalid/tready/tlast`, vì cách này có thể làm Vivado báo override interface và làm `TREADY` bị tie-off về 0. Bản đúng dùng **System ILA monitor interface**.

## 5. Sơ đồ khối tổng thể

```text
                    ┌─────────────────────┐
                    │ zynq_ps             │
                    │ DDR, FCLK, Reset    │
                    │ M_AXI_GP0, S_AXI_HP0│
                    └───────┬───────▲─────┘
                            │       │
                     AXI-Lite       │ AXI HP0 to DDR
                            │       │
                            ▼       │
                    ┌───────────────┴─────┐
                    │ AXI SmartConnect    │
                    │ axi_smc_ctrl        │
                    └────────┬────────────┘
                             ▼
                       ┌───────────┐
                       │ AXI DMA   │
                       │ axi_dma_0 │
                       └───┬───▲───┘
                           │   │
                  AXIS MM2S│   │AXIS S2MM
                           ▼   │
                    ┌─────────────┐
                    │ FIFO IN     │
                    └──────┬──────┘
                           ▼
                    ┌─────────────┐
                    │ FIR Compiler│
                    └──────┬──────┘
                           ▼
                    ┌─────────────┐
                    │ FIFO OUT    │
                    └──────┬──────┘
                           ▼
                    ┌─────────────┐
                    │ Subset Conv │
                    └──────┬──────┘
                           └──────────► AXI DMA S2MM

      AXI DMA M_AXI_MM2S/M_AXI_S2MM → axi_smc_mem → zynq_ps/S_AXI_HP0 → DDR
```


## 6. Chuẩn bị và cấu hình từng IP

### 6.1. `zynq_ps` — ZYNQ7 Processing System

Cấu hình:
```text
Run Block Automation = Yes
Apply Board Preset   = Yes
FCLK_CLK0            = 100 MHz
M_AXI_GP0            = Enable
S_AXI_HP0            = Enable
DDR                  = External
FIXED_IO             = External
```

Tín hiệu chính:

| Tín hiệu | Chức năng |
|---|---|
| `DDR` | Kết nối DDR RAM vật lý |
| `FIXED_IO` | Kết nối MIO/PS clock/reset |
| `M_AXI_GP0` | ARM PS điều khiển AXI DMA qua AXI-Lite |
| `S_AXI_HP0` | Cổng tốc độ cao để PL/DMA truy cập DDR |
| `FCLK_CLK0` | Clock 100 MHz cấp cho hệ thống |
| `FCLK_RESET0_N` | Reset active-low từ PS |

### 6.2. `axi_dma_0` — AXI DMA

Cấu hình:
```text
Scatter Gather Engine = OFF
Micro DMA             = OFF
Control/Status Stream = OFF
Buffer Length Width   = 23
Address Width         = 32
Read Channel MM2S     = ON
Write Channel S2MM    = ON
MM Data Width         = 32
Stream Data Width     = 32
Max Burst Size        = 16
```

Tín hiệu/chân chính:

| Tín hiệu | Vai trò | Ý nghĩa |
|---|---|---|
| `S_AXI_LITE` | Slave AXI-Lite | PS ghi thanh ghi start/status DMA |
| `M_AXI_MM2S` | AXI Master | DMA đọc input từ DDR |
| `M_AXI_S2MM` | AXI Master | DMA ghi output về DDR |
| `M_AXIS_MM2S` | AXI4-Stream Master | Phát input sang FIFO/FIR |
| `S_AXIS_S2MM` | AXI4-Stream Slave | Nhận output từ FIR về DMA |
| `axi_resetn` | Reset | Reset active-low |
| `m_axi_mm2s_aclk` | Clock | Clock kênh đọc |
| `m_axi_s2mm_aclk` | Clock | Clock kênh ghi |
| `s_axi_lite_aclk` | Clock | Clock điều khiển AXI-Lite |

Output đúng:
- Jupyter không treo ở `dma.sendchannel.wait()` và `dma.recvchannel.wait()`.
- `output_buffer` có dữ liệu.
- ILA thấy `TVALID`, `TREADY`, `TDATA`, `TLAST`.

### 6.3. `fir_compiler_0` — FIR Compiler

Cấu hình hệ số:
```text
Coefficient Vector = 6,0,-4,-3,5,6,-6,-13,7,44,64,44,7,-13,-6,6,5,-3,-4,0,6
Number of taps     = 21
Filter Type        = Single Rate
```

Cấu hình dữ liệu:
```text
Input Data Type    = Signed
Input Data Width   = 32
Output Width       = 32
Rounding Mode      = Truncate LSBs
Architecture       = Systolic Multiply Accumulate
TLAST              = Packet Framing
Output TREADY      = Enable
ARESETn            = Enable
```

Tín hiệu chính:

| Tín hiệu | Chức năng |
|---|---|
| `S_AXIS_DATA` | Nhận input stream `x[n]` |
| `M_AXIS_DATA` | Xuất output stream `y[n]` |
| `s_axis_data_tdata` | Mẫu đầu vào |
| `s_axis_data_tvalid` | Input hợp lệ |
| `s_axis_data_tready` | FIR sẵn sàng nhận |
| `s_axis_data_tlast` | Mẫu cuối frame |
| `m_axis_data_tdata` | Mẫu sau lọc |
| `m_axis_data_tvalid` | Output hợp lệ |
| `m_axis_data_tready` | Khối sau sẵn sàng nhận |
| `m_axis_data_tlast` | Mẫu cuối frame output |

### 6.4. `axis_fifo_in` và `axis_fifo_out`

Cấu hình chung:
```text
FIFO Depth       = 512
TDATA_NUM_BYTES  = 4
HAS_TKEEP        = 1
HAS_TLAST        = 1
HAS_TSTRB        = 0
TID/TDEST/TUSER  = 0
```

Lưu ý:
```text
TDATA_NUM_BYTES = 4 nghĩa là 4 byte = 32 bit.
Không nhập 32 vì 32 byte = 256 bit.
```

Chức năng:
- `axis_fifo_in`: đệm giữa DMA MM2S và FIR.
- `axis_fifo_out`: đệm giữa FIR và Subset Converter.

### 6.5. `axis_subset_conv_0` — AXI4-Stream Subset Converter

Cấu hình:
```text
S_TDATA_NUM_BYTES = 4
M_TDATA_NUM_BYTES = 4
S_HAS_TREADY      = 1
M_HAS_TREADY      = 1
S_HAS_TKEEP       = 1
M_HAS_TKEEP       = 1
S_HAS_TLAST       = 1
M_HAS_TLAST       = 1
S_HAS_TSTRB       = 0
M_HAS_TSTRB       = 0
TDATA_REMAP       = tdata[31:0]
TKEEP_REMAP       = tkeep[3:0]
TLAST_REMAP       = tlast[0]
TSTRB_REMAP       = 1'b0
```

Chức năng:
- Đảm bảo dữ liệu ra FIR/FIFO có đủ `TDATA`, `TKEEP`, `TLAST` cho DMA S2MM.
- Output cuối của khối này đi vào `axi_dma_0/S_AXIS_S2MM`.

### 6.6. `axi_smc_mem` — SmartConnect memory

Cấu hình:
```text
NUM_SI = 2
NUM_MI = 1
NUM_CLKS = 1
HAS_ARESETN = 1
```

Nối:
```text
axi_dma_0/M_AXI_MM2S → axi_smc_mem/S00_AXI
axi_dma_0/M_AXI_S2MM → axi_smc_mem/S01_AXI
axi_smc_mem/M00_AXI  → zynq_ps/S_AXI_HP0
```

### 6.7. `axi_smc_ctrl` — SmartConnect control

Cấu hình:
```text
NUM_SI = 1
NUM_MI = 1
NUM_CLKS = 1
HAS_ARESETN = 1
```

Nối:
```text
zynq_ps/M_AXI_GP0    → axi_smc_ctrl/S00_AXI
axi_smc_ctrl/M00_AXI → axi_dma_0/S_AXI_LITE
```

### 6.8. `proc_sys_reset_0`

Nối input:
```text
zynq_ps/FCLK_CLK0     → proc_sys_reset_0/slowest_sync_clk
zynq_ps/FCLK_RESET0_N → proc_sys_reset_0/ext_reset_in
xlconstant_1b1/dout   → proc_sys_reset_0/dcm_locked
xlconstant_1b0/dout   → proc_sys_reset_0/aux_reset_in
xlconstant_1b0/dout   → proc_sys_reset_0/mb_debug_sys_rst
```

Nối output:
```text
interconnect_aresetn → axi_smc_mem/aresetn, axi_smc_ctrl/aresetn
peripheral_aresetn   → axi_dma_0/axi_resetn, fir_compiler_0/aresetn,
                       axis_fifo_in/s_axis_aresetn,
                       axis_fifo_out/s_axis_aresetn,
                       axis_subset_conv_0/aresetn
```

### 6.9. `system_ila_axis_0` — System ILA

Cấu hình:
```text
Monitor slots = 3
Slot type     = AXI4-Stream
Data depth    = 1024
```

Nối đúng:
```text
axi_dma_0/M_AXIS_MM2S       → system_ila_axis_0/SLOT_0_AXIS
fir_compiler_0/M_AXIS_DATA  → system_ila_axis_0/SLOT_1_AXIS
axis_subset_conv_0/M_AXIS   → system_ila_axis_0/SLOT_2_AXIS
zynq_ps/FCLK_CLK0           → system_ila_axis_0/clk
```

Không nối ILA Native vào từng pin rời `tdata/tready/tvalid/tlast`.


## 7. Bảng nối dây đầy đủ

### 7.1. AXI-Lite control path

| Từ | Đến | Ý nghĩa |
|---|---|---|
| `zynq_ps/M_AXI_GP0` | `axi_smc_ctrl/S00_AXI` | PS phát lệnh AXI-Lite |
| `axi_smc_ctrl/M00_AXI` | `axi_dma_0/S_AXI_LITE` | Điều khiển thanh ghi DMA |

### 7.2. AXI memory path

| Từ | Đến | Ý nghĩa |
|---|---|---|
| `axi_dma_0/M_AXI_MM2S` | `axi_smc_mem/S00_AXI` | DMA đọc input từ DDR |
| `axi_dma_0/M_AXI_S2MM` | `axi_smc_mem/S01_AXI` | DMA ghi output về DDR |
| `axi_smc_mem/M00_AXI` | `zynq_ps/S_AXI_HP0` | Truy cập DDR qua HP0 |

### 7.3. AXI4-Stream datapath

| Từ | Đến | Ý nghĩa |
|---|---|---|
| `axi_dma_0/M_AXIS_MM2S` | `axis_fifo_in/S_AXIS` | Input từ DMA sang FIFO |
| `axis_fifo_in/M_AXIS` | `fir_compiler_0/S_AXIS_DATA` | FIFO đưa dữ liệu vào FIR |
| `fir_compiler_0/M_AXIS_DATA` | `axis_fifo_out/S_AXIS` | FIR xuất dữ liệu sau lọc |
| `axis_fifo_out/M_AXIS` | `axis_subset_conv_0/S_AXIS` | FIFO output sang Subset |
| `axis_subset_conv_0/M_AXIS` | `axi_dma_0/S_AXIS_S2MM` | Output quay về DMA |

### 7.4. Clock

Tất cả dùng `zynq_ps/FCLK_CLK0 = 100 MHz`.

| Từ | Đến |
|---|---|
| `zynq_ps/FCLK_CLK0` | `zynq_ps/M_AXI_GP0_ACLK` |
| `zynq_ps/FCLK_CLK0` | `zynq_ps/S_AXI_HP0_ACLK` |
| `zynq_ps/FCLK_CLK0` | `axi_dma_0/s_axi_lite_aclk` |
| `zynq_ps/FCLK_CLK0` | `axi_dma_0/m_axi_mm2s_aclk` |
| `zynq_ps/FCLK_CLK0` | `axi_dma_0/m_axi_s2mm_aclk` |
| `zynq_ps/FCLK_CLK0` | `axi_smc_mem/aclk` |
| `zynq_ps/FCLK_CLK0` | `axi_smc_ctrl/aclk` |
| `zynq_ps/FCLK_CLK0` | `fir_compiler_0/aclk` |
| `zynq_ps/FCLK_CLK0` | `axis_fifo_in/s_axis_aclk` |
| `zynq_ps/FCLK_CLK0` | `axis_fifo_out/s_axis_aclk` |
| `zynq_ps/FCLK_CLK0` | `axis_subset_conv_0/aclk` |
| `zynq_ps/FCLK_CLK0` | `system_ila_axis_0/clk` |

## 8. Ý nghĩa các tín hiệu AXI4-Stream

| Tín hiệu | Ý nghĩa |
|---|---|
| `TDATA` | Dữ liệu mẫu |
| `TVALID` | Bên gửi báo dữ liệu hợp lệ |
| `TREADY` | Bên nhận báo sẵn sàng nhận |
| `TLAST` | Mẫu cuối của frame |
| `TKEEP` | Byte hợp lệ trong `TDATA` |

Điều kiện truyền một mẫu:

```text
TVALID = 1 và TREADY = 1
```

Nếu:
- `TVALID = 1`, `TREADY = 0`: bên sau chưa sẵn sàng.
- `TVALID = 0`, `TREADY = 1`: bên trước chưa gửi.
- Không có `TLAST`: DMA S2MM có thể treo khi chờ kết thúc frame.

## 9. File cần xuất ra

Sau khi Generate Bitstream:

```text
C:/VIVADO/fir_filter_export_pynq/fir_filter_ip_2023_1.bit
C:/VIVADO/fir_filter_export_pynq/fir_filter_ip_2023_1.hwh
C:/VIVADO/fir_filter_export_pynq/fir_filter_ip_2023_1.ltx
```

Copy `.bit` và `.hwh` lên PYNQ:

```text
/home/xilinx/jupyter_notebooks/fir_filter/
```

File `.ltx` dùng cho Vivado Hardware Manager để xem ILA.


# 10. Jupyter Notebook triển khai từng cell

Tên notebook: `test_fir_filter_dma.ipynb`

Đặt cùng thư mục với:
```text
fir_filter_ip_2023_1.bit
fir_filter_ip_2023_1.hwh
```

## Cell 1 — Import thư viện

```python
from pynq import Overlay, allocate
import numpy as np
import matplotlib.pyplot as plt
```

Thư viện:
- `Overlay`: nạp bitstream và đọc `.hwh`.
- `allocate`: cấp buffer vật lý cho DMA.
- `numpy`: tạo tín hiệu, xử lý số, FFT.
- `matplotlib`: vẽ đồ thị.

Input: không có.  
Output: môi trường Python có đủ thư viện.

## Cell 2 — Load overlay

```python
overlay = Overlay("fir_filter_ip_2023_1.bit")
overlay.download()

print("Overlay loaded successfully")
print("IP blocks found in overlay:")
for ip_name in overlay.ip_dict.keys():
    print(" -", ip_name)
```

Input:
- `fir_filter_ip_2023_1.bit`
- `fir_filter_ip_2023_1.hwh`

Output:
- FPGA được program.
- Danh sách IP trong overlay. Cần thấy `axi_dma_0`.

## Cell 3 — Lấy AXI DMA

```python
dma = overlay.axi_dma_0

print("DMA object:", dma)
print("Send channel:", dma.sendchannel)
print("Recv channel:", dma.recvchannel)
```

Ý nghĩa:
- `sendchannel`: MM2S, gửi input từ DDR xuống FIR.
- `recvchannel`: S2MM, nhận output từ FIR về DDR.

## Cell 4 — Tạo tín hiệu test

```python
Fs = 48000
N = 1024

f_low = 1000
f_high = 10000

n = np.arange(N)

x_float = (
    0.7 * np.sin(2 * np.pi * f_low * n / Fs)
    + 0.3 * np.sin(2 * np.pi * f_high * n / Fs)
)

scale = 2**20
x_int = np.round(x_float * scale).astype(np.int32)

print("Number of samples:", N)
print("Sampling frequency:", Fs, "Hz")
print("Input min:", x_int.min())
print("Input max:", x_int.max())
print("First 16 input samples:")
print(x_int[:16])
```

Công thức input:

```math
x[n] = 0.7\sin(2\pi \cdot 1000n/48000) + 0.3\sin(2\pi \cdot 10000n/48000)
```

Sau đó:

```math
x_{int}[n] = round(x[n] \cdot 2^{20})
```

Output: mảng `x_int` kiểu `int32`.

## Cell 5 — Cấp buffer DMA

```python
input_buffer = allocate(shape=(N,), dtype=np.int32)
output_buffer = allocate(shape=(N,), dtype=np.int32)

input_buffer[:] = x_int
output_buffer[:] = 0

print("Input buffer physical address :", hex(input_buffer.physical_address))
print("Output buffer physical address:", hex(output_buffer.physical_address))
```

Input: `x_int`.  
Output:
- `input_buffer`: chứa input.
- `output_buffer`: vùng nhận output.

## Cell 6 — Chạy DMA

```python
input_buffer.flush()
output_buffer.flush()

dma.recvchannel.transfer(output_buffer)
dma.sendchannel.transfer(input_buffer)

dma.sendchannel.wait()
dma.recvchannel.wait()

output_buffer.invalidate()

y_int = np.array(output_buffer, dtype=np.int32)

print("DMA transfer completed")
print("Output min:", y_int.min())
print("Output max:", y_int.max())
print("First 16 output samples:")
print(y_int[:16])
```

Thứ tự đúng:
1. Flush input.
2. Start receive trước.
3. Start send sau.
4. Wait cả hai kênh.
5. Invalidate output.
6. Đọc `y_int`.

Output: `y_int`, chính là tín hiệu sau FIR.

## Cell 7 — Vẽ input/output miền thời gian

```python
plt.figure(figsize=(14, 5))
plt.plot(x_int[:300], label="Input x[n]")
plt.plot(y_int[:300], label="Output y[n] after FIR")
plt.title("FIR Filter Input and Output - Time Domain")
plt.xlabel("Sample index n")
plt.ylabel("Amplitude")
plt.grid(True)
plt.legend()
plt.show()
```

Output: đồ thị so sánh input và output.

## Cell 8 — Chuẩn hóa để dễ nhìn

```python
x_norm = x_int.astype(np.float64)
y_norm = y_int.astype(np.float64)

x_norm = x_norm / np.max(np.abs(x_norm))
y_norm = y_norm / np.max(np.abs(y_norm))

plt.figure(figsize=(14, 5))
plt.plot(x_norm[:300], label="Normalized input")
plt.plot(y_norm[:300], label="Normalized output")
plt.title("Normalized Input and Output")
plt.xlabel("Sample index n")
plt.ylabel("Normalized amplitude")
plt.grid(True)
plt.legend()
plt.show()
```

Mục đích: vì hệ số FIR chưa normalize, output có thể lớn hơn input.

## Cell 9 — FFT trước/sau lọc

```python
freq = np.fft.rfftfreq(N, d=1/Fs)

X = np.abs(np.fft.rfft(x_int))
Y = np.abs(np.fft.rfft(y_int))

plt.figure(figsize=(14, 5))
plt.plot(freq, X, label="Input FFT")
plt.plot(freq, Y, label="Output FFT")
plt.title("FFT Before and After FIR Filter")
plt.xlabel("Frequency (Hz)")
plt.ylabel("Magnitude")
plt.grid(True)
plt.legend()
plt.xlim(0, 20000)
plt.show()
```

DFT:

```math
X[k] = \sum_{n=0}^{N-1} x[n]e^{-j2\pi kn/N}
```

Output: phổ input/output.

## Cell 10 — Đo gain tại 1 kHz và 10 kHz

```python
def nearest_bin(freq_array, target_freq):
    return np.argmin(np.abs(freq_array - target_freq))

idx_1k = nearest_bin(freq, 1000)
idx_10k = nearest_bin(freq, 10000)

gain_1k = Y[idx_1k] / X[idx_1k]
gain_10k = Y[idx_10k] / X[idx_10k]

print("Nearest bin for 1 kHz :", freq[idx_1k], "Hz")
print("Nearest bin for 10 kHz:", freq[idx_10k], "Hz")
print("Gain at 1 kHz :", gain_1k)
print("Gain at 10 kHz:", gain_10k)
```

Công thức:

```math
Gain(f) = |Y(f)| / |X(f)|
```

## Cell 11 — Test ramp đơn giản

```python
N2 = 64

input_ramp = allocate(shape=(N2,), dtype=np.int32)
output_ramp = allocate(shape=(N2,), dtype=np.int32)

input_ramp[:] = np.arange(N2, dtype=np.int32)
output_ramp[:] = 0

input_ramp.flush()
output_ramp.flush()

dma.recvchannel.transfer(output_ramp)
dma.sendchannel.transfer(input_ramp)

dma.sendchannel.wait()
dma.recvchannel.wait()

output_ramp.invalidate()

print("Input ramp:")
print(np.array(input_ramp))

print("Output ramp:")
print(np.array(output_ramp))
```

Mục đích: kiểm tra nhanh DMA/FIR có chạy không. Nếu cell này treo, cần kiểm tra `TLAST`, `TREADY`, đường AXI4-Stream.

## Cell 12 — Giải phóng buffer

```python
input_buffer.freebuffer()
output_buffer.freebuffer()

try:
    input_ramp.freebuffer()
    output_ramp.freebuffer()
except Exception:
    pass

print("Buffers released")
```

## 11. Xem waveform bằng System ILA

Jupyter không hiển thị ILA trực tiếp. Jupyter chỉ tạo giao dịch DMA. Waveform xem trong Vivado Hardware Manager.

Trình tự:
1. Cắm JTAG vào PYNQ-Z2.
2. Vivado → Open Hardware Manager.
3. Open Target → Auto Connect.
4. Program Device bằng:
   - `fir_filter_ip_2023_1.bit`
   - `fir_filter_ip_2023_1.ltx`
5. Set trigger:
   - `SLOT_0_AXIS_TVALID == 1`, hoặc
   - `SLOT_0_AXIS_TVALID == 1 && SLOT_0_AXIS_TREADY == 1`, hoặc
   - `SLOT_2_AXIS_TLAST == 1`.
6. Bấm Run Trigger.
7. Quay lại Jupyter chạy DMA cell.
8. ILA sẽ bắt waveform.

Dấu hiệu đúng:
```text
TVALID có xung
TREADY có xung
TDATA thay đổi
TLAST lên 1 ở mẫu cuối frame
```

## 12. Lỗi cũ cần tránh

### Native ILA làm hỏng AXI4-Stream

Không nối:
```text
axi_dma_0/m_axis_mm2s_tdata  → ila_axis_0/probe0
axi_dma_0/m_axis_mm2s_tready → ila_axis_0/probe2
```

Vì Vivado có thể báo:
```text
The connection to the pin has been overridden by the user.
This pin will not be connected as a part of the interface connection.
```

Hậu quả:
```text
TREADY bị tie-off về 0
DMA có thể treo
```

Cách đúng:
```text
Dùng System ILA monitor interface AXI4-Stream.
```

### Tcl `glob -recursive`

Không dùng:
```tcl
glob -recursive
```

Vivado Tcl 2023.1 có thể báo `bad option "-recursive"`.

## 13. Checklist chạy dự án

| Bước | Yêu cầu |
|---|---|
| Validate Design | Không còn lỗi `BD 41-1271`, `BD 41-759` |
| Generate Bitstream | Có `.bit` |
| Export HWH | Có `.hwh` |
| Export LTX | Có `.ltx` nếu dùng ILA |
| Copy lên PYNQ | `.bit` và `.hwh` cùng thư mục notebook |
| Load Overlay | Thấy `axi_dma_0` |
| DMA transfer | Không treo |
| Plot output | Có sóng output |
| FFT | Có phổ trước/sau lọc |
| ILA | Có `TVALID`, `TREADY`, `TDATA`, `TLAST` |
