## SPDX-License-Identifier: MIT OR Apache-2.0
##
## Copyright (c) 2018-2020 Andre Richter <andre.o.richter@gmail.com>

# Default to the RPi3
ifndef BSP
	BSP = rpi3
endif

# Default to a serial device name that is common in Linux.
ifndef DEV_SERIAL
	DEV_SERIAL = /dev/ttyUSB0
endif

# Query the host system's kernel name
UNAME_S := $(shell uname -s)

# BSP-specific arguments
ifeq ($(BSP),rpi3)
	TARGET            = aarch64-unknown-none-softfloat
	OUTPUT            = kernel8.img
	QEMU_BINARY       = qemu-system-aarch64
	QEMU_MACHINE_TYPE = raspi3
	QEMU_RELEASE_ARGS = -serial stdio -display none
	OPENOCD_ARG       = -f /openocd/tcl/interface/ftdi/olimex-arm-usb-tiny-h.cfg -f /openocd/rpi3.cfg
	JTAG_BOOT_IMAGE   = ../X1_JTAG_boot/jtag_boot_rpi3.img
	LINKER_FILE       = src/bsp/raspberrypi/link.ld
	RUSTC_MISC_ARGS   = -C target-cpu=cortex-a53
else ifeq ($(BSP),rpi4)
	TARGET            = aarch64-unknown-none-softfloat
	OUTPUT            = kernel8.img
	# QEMU_BINARY       = qemu-system-aarch64
	# QEMU_MACHINE_TYPE =
	# QEMU_RELEASE_ARGS = -serial stdio -display none
	OPENOCD_ARG       = -f /openocd/tcl/interface/ftdi/olimex-arm-usb-tiny-h.cfg -f /openocd/rpi4.cfg
	JTAG_BOOT_IMAGE   = ../X1_JTAG_boot/jtag_boot_rpi4.img
	LINKER_FILE       = src/bsp/raspberrypi/link.ld
	RUSTC_MISC_ARGS   = -C target-cpu=cortex-a72
endif

SOURCES = $(wildcard **/*.rs) $(wildcard **/*.S) $(wildcard **/*.ld)

RUSTFLAGS          = -C link-arg=-T$(LINKER_FILE) $(RUSTC_MISC_ARGS)
RUSTFLAGS_PEDANTIC = $(RUSTFLAGS) -D warnings -D missing_docs

COMPILER_ARGS = --target=$(TARGET) \
	--features bsp_$(BSP)      \
	--release

RUSTC_CMD   = cargo rustc $(COMPILER_ARGS)
DOC_CMD     = cargo doc $(COMPILER_ARGS)
CLIPPY_CMD  = cargo clippy $(COMPILER_ARGS)
OBJCOPY_CMD = cargo objcopy \
	--                  \
	--strip-all         \
	-O binary

KERNEL_ELF = target/$(TARGET)/release/kernel

DOCKER_IMAGE         = rustembedded/osdev-utils
DOCKER_CMD           = docker run -it --rm -v $(shell pwd):/work/tutorial -w /work/tutorial
DOCKER_ARG_DIR_UTILS = -v $(shell pwd)/../utils:/work/utils
DOCKER_ARG_DIR_JTAG  = -v $(shell pwd)/../X1_JTAG_boot:/work/X1_JTAG_boot
DOCKER_ARG_DEV       = --privileged -v /dev:/dev
DOCKER_ARG_NET       = --network host

DOCKER_QEMU = $(DOCKER_CMD) $(DOCKER_IMAGE)
DOCKER_GDB  = $(DOCKER_CMD) $(DOCKER_ARG_NET) $(DOCKER_IMAGE)

# Dockerize commands that require USB device passthrough only on Linux
ifeq ($(UNAME_S),Linux)
DOCKER_CMD_DEV = $(DOCKER_CMD) $(DOCKER_ARG_DEV)

DOCKER_CHAINBOOT = $(DOCKER_CMD_DEV) $(DOCKER_ARG_DIR_UTILS) $(DOCKER_IMAGE)
DOCKER_JTAGBOOT  = $(DOCKER_CMD_DEV) $(DOCKER_ARG_DIR_UTILS) $(DOCKER_ARG_DIR_JTAG) $(DOCKER_IMAGE)
DOCKER_OPENOCD   = $(DOCKER_CMD_DEV) $(DOCKER_ARG_NET) $(DOCKER_IMAGE)
else
DOCKER_OPENOCD   = echo "Not yet supported on non-Linux systems."; \#
endif

EXEC_QEMU     = $(QEMU_BINARY) -M $(QEMU_MACHINE_TYPE)
EXEC_MINIPUSH = ruby ../utils/minipush.rb

.PHONY: all doc qemu chainboot jtagboot openocd gdb gdb-opt0 clippy clean readelf objdump nm

all: clean $(OUTPUT)

$(KERNEL_ELF): $(SOURCES)
	RUSTFLAGS="$(RUSTFLAGS_PEDANTIC)" $(RUSTC_CMD)

$(OUTPUT): $(KERNEL_ELF)
	cp $< .
	$(OBJCOPY_CMD) $< $(OUTPUT)

doc:
	$(DOC_CMD) --document-private-items --open

ifeq ($(QEMU_MACHINE_TYPE),)
qemu:
	@echo "This board is not yet supported for QEMU."
else
qemu: all
	@$(DOCKER_QEMU) $(EXEC_QEMU) $(QEMU_RELEASE_ARGS) -kernel $(OUTPUT)
endif

chainboot: all
	@$(DOCKER_CHAINBOOT) $(EXEC_MINIPUSH) $(DEV_SERIAL) $(OUTPUT)

jtagboot:
	@$(DOCKER_JTAGBOOT) $(EXEC_MINIPUSH) $(DEV_SERIAL) $(JTAG_BOOT_IMAGE)

openocd:
	@$(DOCKER_OPENOCD) openocd $(OPENOCD_ARG)

define gen_gdb
	RUSTFLAGS="$(RUSTFLAGS_PEDANTIC) $1"  $(RUSTC_CMD)
	cp $(KERNEL_ELF) kernel_for_jtag
	@$(DOCKER_GDB) gdb-multiarch -q kernel_for_jtag
endef

gdb: clean $(SOURCES)
	$(call gen_gdb,-C debuginfo=2)

gdb-opt0: clean $(SOURCES)
	$(call gen_gdb,-C debuginfo=2 -C opt-level=0)

clippy:
	RUSTFLAGS="$(RUSTFLAGS_PEDANTIC)" $(CLIPPY_CMD)

clean:
	rm -rf target

readelf:
	readelf -a kernel

objdump:
	cargo objdump --target $(TARGET) -- -disassemble -no-show-raw-insn -print-imm-hex kernel

nm:
	cargo nm --target $(TARGET) -- -print-size kernel | sort
