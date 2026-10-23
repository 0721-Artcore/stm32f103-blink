# stm32f103-blink

STM32F103C6T6A 最小系统板（Blue Pill）上的 LED 闪烁示例，使用 **STM32CubeMX + HAL + Makefile** 工程结构，可直接用 `arm-none-eabi-gcc` 命令行编译、`ST-Link` 烧录。

## 功能

PA0 上的 LED 循环闪烁：

```
亮 200 ms  →  灭 800 ms  →  亮 200 ms  →  ...
└──────────── 周期 1 s ────────────┘
```

| 参数 | 值 |
|---|---|
| 系统时钟 | 72 MHz（HSE 8 MHz × PLL9）|
| APB1 | 36 MHz |
| 输出引脚 | PA0（推挽输出，低速度）|
| 时序 | `HAL_Delay(200)` / `HAL_Delay(800)` |
| SysTick | 1 ms（HAL 默认）|

## 硬件

| 项 | 型号 / 说明 |
|---|---|
| MCU | 默认按 **STM32F103C6T6A**（低密度，32 KB Flash / 10 KB RAM，chip_id `0x412`）配置；同一份镜像也可运行在 C8T6（chip_id `0x410`，实测 128 KB / 20 KB）上 |
| 板子 | Blue Pill 最小系统板（8 MHz 晶振）|
| 调试器 | ST-Link V2（克隆版，SWD）|
| LED | 外接，PA0 驱动 |
| 供电 | 3.3 V |

### 接线

```
PA0 ── 220Ω ── LED 阳极 ── LED 阴极 ── GND
```

> 高电平点亮。若你的 LED 接在 `3V3 → LED → 电阻 → PA0`（低电平点亮），需把 `main.c` 里的 `GPIO_PIN_SET` / `GPIO_PIN_RESET` 对调。

### SWD 接线（ST-Link V2 → 板子）

| ST-Link | Blue Pill |
|---|---|
| 3.3V | 3V3（板子无其他供电时才接）|
| GND | GND |
| SWDIO | PA13 |
| SWCLK | PA14 |

`BOOT0` 跳线必须接 **0（GND）**，否则从系统存储器启动，程序不会运行。

## 目录结构

```
STM32F103C6Tx/
├── .gitignore
├── README.md
└── project1/                     # CubeMX 工程
    ├── project1.ioc              # CubeMX 配置文件（改引脚/时钟用）
    ├── Makefile                  # 由 CubeMX 生成，GCC 构建
    ├── STM32F103xx_FLASH.ld      # Flash 32K @0x08000000 / RAM 10K @0x20000000
    ├── startup_stm32f103x6.s     # 低密度启动文件
    ├── Core/
    │   ├── Inc/                  # main.h、stm32f1xx_hal_conf.h、gpio.h ...
    │   └── Src/                  # main.c、gpio.c、stm32f1xx_it.c ...
    └── Drivers/
        ├── CMSIS/                # 内核头文件与设备头文件
        └── STM32F1xx_HAL_Driver/ # HAL 驱动（仅启用所需模块）
```

## 环境依赖

Arch Linux：

```bash
sudo pacman -S arm-none-eabi-gcc arm-none-eabi-binutils arm-none-eabi-newlib \
               arm-none-eabi-gdb openocd stlink make
```

可选（代码提示 / 调试）：

```bash
sudo pacman -S clangd bear picocom
```

版本参考：GCC 16.2.0、stlink 1.9.0、openocd 0.12。

## 编译

```bash
cd project1
make -j$(nproc)          # 产物在 build/
make clean               # 清理
arm-none-eabi-size build/project1.elf
```

产物：

```
build/project1.elf   # 带调试信息，用于 GDB
build/project1.bin   # 裸二进制，用于烧录
build/project1.hex   # Intel HEX
build/project1.map   # 链接映射
```

当前资源占用：

```
   text    data     bss     dec
   3376      12    1572    4960      → Flash ≈ 3.3 KB / 32 KB，RAM ≈ 1.5 KB / 10 KB
```

## 烧录

```bash
# 方式 1：st-flash（最快）
st-flash --reset write build/project1.bin 0x08000000

# 方式 2：openocd（校验 + 复位）
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
        -c "program build/project1.elf verify reset exit"

# 确认调试器与芯片
st-info --probe
# flash: 32768, sram: 10240, chipid: 0x412
```

## 在线调试

```bash
# 终端 1：启动 GDB server（端口 3333）
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg

# 终端 2：连接
arm-none-eabi-gdb build/project1.elf \
  -ex 'target extended-remote :3333' \
  -ex 'monitor reset halt' \
  -ex 'load'
```

常用命令：`b main`、`c`、`n`、`s`、`info reg`、`monitor reset halt`。

## 用 CubeMX 修改工程

1. 打开 `project1/project1.ioc`
2. 修改引脚 / 时钟 / 外设
3. **Project Manager → Toolchain/IDE** 保持 `Makefile`
4. Generate Code

> ⚠️ **只修改 `USER CODE BEGIN` / `USER CODE END` 之间的代码**，其余部分重新生成时会被覆盖。
>
> 闪灯逻辑位于 `Core/Src/main.c` 的 `USER CODE BEGIN 3` 块内。

生成后核对三项，确保仍是低密度配置：

```bash
grep -E 'STM32F103x6|startup_stm32f103x6' Makefile
grep -A3 MEMORY STM32F103xx_FLASH.ld     # 32K / 10K
st-info --probe | grep -E 'chipid|flash|sram'
```

## Neovim / clangd（可选）

CubeMX 的 Makefile 不生成编译数据库，clangd 需要它才能正确解析交叉编译参数：

```bash
make clean && bear -- make -j$(nproc)      # 生成 compile_commands.json
```

clangd 需允许调用交叉编译器，否则找不到 `stdint.h` 等系统头：

```lua
-- ~/.config/nvim/lua/plugins/clangd.lua
return {
  {
    "neovim/nvim-lspconfig",
    opts = {
      servers = {
        clangd = {
          cmd = { "clangd", "--background-index", "--clang-tidy",
                  "--header-insertion=iwyu",
                  "--query-driver=/usr/bin/arm-none-eabi-*" },
        },
      },
    },
  },
}
```

常用操作：`:make` 编译、`:copen` 查看错误、`gd` 跳定义、`K` 查看文档。

## 已知问题 / 注意事项

1. **芯片容量**：工程按 **32 KB Flash / 10 KB RAM** 的保守配置生成（链接脚本、启动文件、`-DSTM32F103x6` 一致）。该镜像在 32 KB 与 128 KB 的板子上都能运行；若要使用更大容量，需按实际型号重新生成工程。容量以 `st-info --probe` 读数为准。
2. **链接脚本**：若切换为 64 KB 的 C8T6 并希望用满容量，需同时修改 `STM32F103xx_FLASH.ld` 的 `LENGTH`（64K/20K）、启动文件和 `-DSTM32F103xB`。
3. **SWD 引脚**：本工程已配置 `SYS → Debug = Serial Wire`（PA13/PA14），并在 `stm32f1xx_hal_msp.c` 中执行 `__HAL_AFIO_REMAP_SWJ_NOJTAG()`；请勿把这两个引脚改为普通 GPIO，否则调试器将无法连接。
4. **NRST 未接**：本项目通过 SWD 的 `SYSRESETREQ` 软复位，若程序进入低功耗或关闭调试，需接 NRST 或使用 `st-flash --connect-under-reset`。
5. **LED 极性**：默认高电平点亮，接法不同需对调 `SET`/`RESET`。
6. **空间限制**：32 KB Flash 下 HAL 工程需保持 `-ffunction-sections -fdata-sections -Wl,--gc-sections`，必要时把 `OPT` 从 `-Og` 改为 `-Os`。
7. **USB 时钟**：当前 USB 预分频为 `/1`（72 MHz），不满足 USB 2.0 全速所需 48 MHz。本项目未使用 USB；若启用 USB，请改为 `/1.5`。

详细的配置过程、问题排查与验证记录见 [docs/tech-log.md](docs/tech-log.md)。

## 参考

- [STM32F103C6 数据手册](https://www.st.com/en/microcontrollers-microprocessors/stm32f103c6.html)
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)
- [stlink 工具](https://github.com/stlink-org/stlink)
