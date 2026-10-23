# v1.0.0 — PA0 LED 200 ms / 800 ms

## 功能

- PA0 外接 LED：**亮 200 ms、灭 800 ms**，周期 1 s，占空比 20%
- 系统时钟 72 MHz（HSE 8 MHz × PLL9），APB1 36 MHz / APB2 72 MHz
- 使用 CubeMX 生成 HAL 工程，构建系统为 `Makefile`，可在 Neovim 等任意编辑器下开发

## 硬件

| 项 | 说明 |
|---|---|
| MCU | STM32F103C6T6A（`0x412`，32 KB / 10 KB）或 C8T6（`0x410`，实测 128 KB / 20 KB）|
| 调试器 | ST-Link V2（SWD）|
| 供电 | 3.3 V |
| LED | `PA0 → 220Ω → LED → GND`（高电平点亮）|

工程按 32 KB Flash / 10 KB RAM 的保守配置生成，同一份镜像可在上述两种板卡上运行。

## 构建

```bash
cd project1
make -j$(nproc)
arm-none-eabi-size build/project1.elf
```

## 烧录

```bash
st-flash --reset write project1/build/project1.bin 0x08000000
# 或
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
        -c "program project1/build/project1.elf verify reset exit"
```

## 资源占用（`-Og`）

| 资源 | 占用 | 容量（32 KB / 10 KB 配置）| 比例 |
|---|---|---|---|
| Flash | 3388 B | 32768 B | 10.3 % |
| RAM | 1584 B | 10240 B | 15.5 % |

## 验证

- 编译：0 error / 0 warning（`-Wall -Wextra`）
- 烧录：`4/4 pages written`，`Flash written and verified!`，软复位成功
- 现象：LED 亮约 0.2 s、灭约 0.8 s（目测，2026-10-23）

## 已知事项

- 工程按 32 KB 保守配置；如需用满更大容量，需按实际型号重新生成工程
- USB 预分频当前为 `/1`（72 MHz），未启用 USB；若启用需改为 `/1.5`（48 MHz）
- Neovim/clangd 使用前需配置 `--query-driver=/usr/bin/arm-none-eabi-*` 并生成 `compile_commands.json`

## 附件

- `led-blink-1.0.0.bin` / `.hex` / `.elf` — 固件
- `stm32f103-blink-1.0.0-src.zip` — 源码（含 `docs/` 文档与截图）
- `SHA256SUMS.txt` — 校验和

详细的配置过程、截图与问题排查记录见仓库中的 [docs/tech-log.md](../docs/tech-log.md)。
