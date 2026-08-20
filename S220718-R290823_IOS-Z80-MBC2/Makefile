# ============================================================
#  Z80-MBC2 IOS 固件 Makefile
#  编译: arduino-cli + MightyCore (ATmega32, 16MHz 外部晶振)
#  烧录: MightyCore 自带 avrdude + USBasp (usbasp-clone)
#  不带参数运行 make 显示帮助
# ============================================================

# ---------- 可按需覆盖的变量（例: make flash PROGRAMMER=usbasp）----------
FQBN        ?= MightyCore:avr:32:clock=16MHz_external,BOD=2v7,eeprom=keep,LTO=Os_flto,pinout=standard,bootloader=no_bootloader
PROGRAMMER  ?= usbasp-clone
MCU         ?= m32
PORT        ?= $(firstword $(wildcard /dev/cu.usbserial* /dev/cu.wchusbserial*) /dev/cu.usbserial)
BAUD        ?= 115200

# 熔丝（无 bootloader，JTAG 关闭 —— JTAG 脚被 Z80 总线占用，绝不能开）
LFUSE       ?= 0xaf
HFUSE       ?= 0xd7

# ---------- 串口烧录（urboot bootloader + urclock 协议）----------
# MightyCore 官方 Burn Bootloader 的熔丝（16MHz 外晶振/BOD 2.7V/保留 EEPROM/JTAG 关）
# 注意 hfuse 0xc7 的 CKOPT=0（振荡器满摆幅, 16MHz 的数据手册推荐值）; BOOTRST 保持 1,
# urboot 是"向量补丁"式 bootloader, 上传时由 urclock 改写复位向量, 不占用 BOOT 熔丝
BL_LFUSE    ?= 0xbf
BL_HFUSE    ?= 0xc7
UPLOAD_BAUD ?= 115200
MC_DIR      := $(HOME)/Library/Arduino15/packages/MightyCore/hardware/avr/3.1.0
URBOOT_HEX  := $(MC_DIR)/bootloaders/urboot/atmega32/watchdog_1_s/autobaud/uart0_rxd0_txd1/led+b0/urboot_atmega32_pr_ee_ce.hex

# ---------- 路径 ----------
SKETCH_DIR  := $(CURDIR)
BUILD_DIR   := $(SKETCH_DIR)/build
SKETCH_NAME := S220718-R290823_IOS-Z80-MBC2
HEX         := $(BUILD_DIR)/$(SKETCH_NAME).ino.hex
SRC         := $(wildcard *.ino *.cpp *.h)

MIGHTYCORE  := $(HOME)/Library/Arduino15/packages/MightyCore/tools/avrdude/8.0-arduino.1
AVRDUDE     := $(MIGHTYCORE)/bin/avrdude
AVRCONF     := $(MIGHTYCORE)/etc/avrdude.conf
AVR         := "$(AVRDUDE)" -C "$(AVRCONF)" -c $(PROGRAMMER) -p $(MCU)

.DEFAULT_GOAL := help
.PHONY: help build probe fuse-read fuse flash verify deploy bootloader flash-serial monitor clean

# ---------- 帮助 ----------
help:
	@echo ""
	@echo "Z80-MBC2 IOS 固件构建与烧录"
	@echo "================================================================"
	@echo "编译:"
	@echo "  make build      编译固件, 输出 build/$(SKETCH_NAME).ino.hex"
	@echo "  make clean      删除 build/ 目录"
	@echo ""
	@echo "烧录（USBasp 接 J3 ICSP 口, 走 USB 直连, 不需要也不会出现串口）:"
	@echo "  make probe      0) 通信确认 (avrdude -v, 能读到芯片签名即 OK)"
	@echo "  make fuse-read  1) 读当前熔丝"
	@echo "  make fuse       2) 写熔丝 lfuse=$(LFUSE) hfuse=$(HFUSE)  ⚠ 全新芯片需闭合 USBasp 的 JP3 慢速跳线"
	@echo "  make flash      3) 编译并烧录固件  ⚠ 熔丝写完后断开 JP3"
	@echo "  make verify     4) 校验 flash 与 hex 一致"
	@echo "  make deploy     = build + flash + verify 一条龙"
	@echo ""
	@echo "串口烧录（USB-TTL 接 J2, 日常更新固件不用再拔 USBasp）:"
	@echo "  make bootloader 一次性: 用 USBasp 写熔丝 lfuse=$(BL_LFUSE) hfuse=$(BL_HFUSE) 并安装 urboot"
	@echo "                  ⚠ 会整片擦除, 装完需用 make flash-serial 重新烧固件"
	@echo "  make flash-serial  编译并经串口烧录 (urclock 协议, $(UPLOAD_BAUD)bps, 靠 J2 的 DTR 自动复位)"
	@echo "  ⚠ 两种流程别混用: make flash (ISP) 整片擦除会把 bootloader 抹掉,"
	@echo "    之后想再用串口烧录就得重跑 make bootloader"
	@echo ""
	@echo "其它:"
	@echo "  make monitor    打开串口监视器看板子输出 —— 这个走 USB-TTL(CH340) 接 J2,"
	@echo "                  与 USBasp 无关 (PORT=$(PORT), $(BAUD) 8N1)"
	@echo ""
	@echo "可覆盖变量: PROGRAMMER=$(PROGRAMMER)  MCU=$(MCU)  LFUSE/HFUSE  PORT  BAUD"
	@echo "提示: 'cannot set sck period' 警告来自老版 USBasp 固件, 无害可忽略"
	@echo "================================================================"
	@echo ""

# ---------- 编译 ----------
build: $(HEX)

$(HEX): $(SRC)
	arduino-cli compile -b "$(FQBN)" --output-dir "$(BUILD_DIR)" "$(SKETCH_DIR)"
	@echo ""
	@echo "==> 编译完成: $(HEX)"

# ---------- 烧录 ----------
probe:
	$(AVR) -v

fuse-read:
	$(AVR) -U lfuse:r:-:h -U hfuse:r:-:h

fuse:
	@echo "==> 写熔丝 lfuse=$(LFUSE) hfuse=$(HFUSE) （全新芯片出厂 1MHz, 需闭合 JP3 降低 SCK）"
	$(AVR) -U lfuse:w:$(LFUSE):m -U hfuse:w:$(HFUSE):m
	@echo "==> 回读确认:"
	$(AVR) -U lfuse:r:-:h -U hfuse:r:-:h

flash: $(HEX)
	@echo "==> 烧录 $(HEX) （熔丝已是 16MHz 的话请断开 JP3, 否则很慢）"
	$(AVR) -U flash:w:$(HEX):i

verify: $(HEX)
	$(AVR) -U flash:v:$(HEX):i

deploy: build flash verify
	@echo "==> 完成: 编译 + 烧录 + 校验"

# ---------- 串口烧录 ----------
# 一次性: 通过 USBasp 安装 urboot bootloader（照抄 MightyCore「烧录引导程序」的两步配方）
bootloader:
	@echo "==> 步骤 1/2: 整片擦除 + 写熔丝 lfuse=$(BL_LFUSE) hfuse=$(BL_HFUSE)"
	$(AVR) -e -U lock:w:0xff:m -U lfuse:w:$(BL_LFUSE):m -U hfuse:w:$(BL_HFUSE):m
	@echo "==> 步骤 2/2: 写入 urboot bootloader (384 字节, flash 顶部)"
	$(AVR) -D -U flash:w:"$(URBOOT_HEX)":i -U lock:w:0xff:m
	@echo "==> 完成: 芯片已被擦除, 现在用 make flash-serial 烧固件"

# 日常: 经 J2 串口烧录（需要先 make bootloader; DTR 经 C5 自动复位进 bootloader）
flash-serial: $(HEX)
	@test -e "$(PORT)" || { echo "错误: 找不到串口 $(PORT), 请插上 USB-TTL 或用 PORT=/dev/cu.xxx 指定"; exit 1; }
	"$(AVRDUDE)" -C "$(AVRCONF)" -c urclock -p $(MCU) -P "$(PORT)" -b $(UPLOAD_BAUD) -D -xnometadata -U flash:w:"$(HEX)":i

# ---------- 串口监视器 ----------
monitor:
	arduino-cli monitor -p "$(PORT)" -c baudrate=$(BAUD)

# ---------- 清理 ----------
clean:
	rm -rf "$(BUILD_DIR)"
