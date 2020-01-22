```
###############################################################################
#          MAIN PART
#Makefile 是和 make 命令一起配合使用的.
#很多大型项目的编译都是通过 Makefile 来组织的, 如果没有 Makefile, 那很多项目中各种库和代码之间的依赖关系不知会多复杂.
#Makefile的组织流程的能力如此之强, 不仅可以用来编译项目, 还可以用来组织我们平时的一些日常操作. 这个需要大家发挥自己的想象力.
#


显式规则：生成目标文件
例中，edit是第一个target，它将作为最终的目标文件
edit : main.o command.o display.o insert.o               #目标文件edit，指定依赖文件
	cc -o edit main.o command.o display.o insert.o   #目标文件edit，命令(必须以Tab开头)，也就是make需要执行的命令。（任意的shell命令）
main.o : main.c defs.h
	cc -c main.c
command.o : command.c defs.h command.h
	cc -c command.c
display.o : display.c defs.h buffer.h
	cc -c display.c
insert.o : insert.c defs.h buffer.h
	cc -c insert.c



变量定义：
objects = main.o kbd.o command.o display.o  #定义一个名为 objects 的变量
MAKEFILES = foo.make a.mk b.mk c.mk e.mk    #定义环境变量，名称只能是MAKEFILES，值是其它的Makefile，用空格分隔，
从环境变量中引入的Makefile的“目标”不会起作用，如果环境变量中定义的文件发现错误，make也会不理。只要这个变量一被定义，那么当你使用make时，所有的Makefile都会受到它的影响






引用其它的Makefile：
bar=e.mk f.mk
include foo.make *.mk $(bar)【相当于】include foo.make a.mk b.mk c.mk e.mk f.mk
命令：
必须要以[Tab]键开始







#GNU make 的工作方式
#读入主Makefile (主Makefile中可以引用其他Makefile)
#读入被include的其他Makefile
#初始化文件中的变量
#推导隐晦规则, 并分析所有规则
#为所有的目标文件创建依赖关系链
#根据依赖关系, 决定哪些目标要重新生成
#执行生成命令


#默认执行 make 命令时, GNU make在当前目录下依次搜索下面3个文件 "GNUmakefile", "makefile", "Makefile",
#找到对应文件之后, 就开始执行此文件中的第一个目标(target). 如果找不到这3个文件就报错.




#变量调用： $(  )
#-rm edit $(objects)  相当于  -rm edit main.o kbd.o command.o display.o



#【:=】定义变量，只能使用前面定义好的变量
#【 =】定义变量，可以使用后面定义的变量
#【+=】变量追加值 ，
#【override】变量覆盖，
#【target】目标变量作用是使变量的作用域仅限于这个目标(target), 而不像之前例子中定义的变量, 对整个Makefile都有效

#命令

###############################################################################
CROSS_COMPILE?=/home/dev/projects/crosscompiler/gcc-arm-none-eabi-5_4-2016q2/bin/arm-none-eabi-
JLINKGDBSERVER?=/opt/JLink_Linux_V600d_x86_64/JLinkGDBServer

# Tools definitions
CC=$(CROSS_COMPILE)gcc
CXX=$(CROSS_COMPILE)g++
LD=$(CROSS_COMPILE)ld
AR=$(CROSS_COMPILE)ar
AS=$(CROSS_COMPILE)as
SIZE=$(CROSS_COMPILE)size
OBJCOPY=$(CROSS_COMPILE)objcopy

VERBOSE ?= 0

# rules verbosity
ifneq '$(VERBOSE)' '0'
P = @ true
E =
else
P = @ echo
E = @
endif

TARGET=tudigong.elf

include ../../Drivers/BSP/TuDiGong/tudigong.mk
include ../../Middlewares/mbmaster/mbmaster.mk

LIBS = -lgcc -lc -lnosys -lm

STDLIB_SRCS += ../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/ml_math_func.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/start_manager.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/hal_outputs.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/mpl.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/mlmath.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/results_holder.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/data_builder.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/storage_manager.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/message_layer.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/eMPL-hal/eMPL_outputs.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/driver/eMPL/inv_mpu_dmp_motion_driver.c \
../../Middlewares/InvenSense/motion_driver_6.12/core/driver/eMPL/inv_mpu.c

SRCS += \
./Src/main.c \
./Src/app_algo.c \
./Src/lora.c \
./Src/sensor.c \
./Src/movee_mpu_config.c \
./Src/movee_algorithms.c \
../../Drivers/CMSIS/Device/SiliconLabs/EFM32LG/Source/system_efm32lg.c

INCLUDES += -I Inc/ \
        -I../../Middlewares/InvenSense/motion_driver_6.12/core/driver/include/ \
        -I../../Middlewares/InvenSense/motion_driver_6.12/core/driver/eMPL/ \
        -I../../Middlewares/InvenSense/motion_driver_6.12/core/mllite/ \
        -I../../Middlewares/InvenSense/motion_driver_6.12/core/mpl/ \
        -I../../Middlewares/InvenSense/motion_driver_6.12/core/eMPL-hal/

LIBS +=  ../../Middlewares/InvenSense/motion_driver_6.12/libraries/arm/GCC/m3/liblibmplmpu.a -lm

LDSCRIPT = ../../Drivers/CMSIS/Device/SiliconLabs/EFM32LG/Source/GCC/efm32lg.ld

# For MPU Accelerometer
CFLAGS += -DMPU6500 -DEMPL_TARGET_AGRION -DEMPL -DMPU_DEBUG -DARM_MATH_CM3
# For EFM32 Cortex-m3
CFLAGS += -mcpu=cortex-m3 -mthumb 
#-msoft-float
# Reduce binary size
CFLAGS += -ffunction-sections -fdata-sections
CFLAGS += -std=gnu99 -O0
CFLAGS += -g -gdwarf-2 -DLOGLEVEL=LOGDEBUG -DDEBUG_EFM_USER -DASSERT_BREAKPOINT -DUSE_FULL_ASSERT
CFLAGS += $(INCLUDES)
# /!\ Linker options : nano lib
LDFLAGS +=  -Wl,-Map=$(TARGET:.elf=.map) -L. -Wl,-T$(LDSCRIPT) -O0 --specs=nano.specs -Wl,--gc-sections -u _printf_float
OBJS=$(SRCS:.c=.o) $(ASM:.S=.o)
STDLIB_OBJS=$(STDLIB_SRCS:.c=.o)
DEPS=$(SRCS:.c=.depends) $(STDLIB_SRCS:.c=.depends)

$(OBJS): EXTRA_FLAGS := -Wall -Wextra -Wunreachable-code -Wno-unused-parameter
ifneq (,$(wildcard $(CPPTESTSCAN_PATH)))
$(OBJS): CC := $(CPPTESTSCAN_PATH) $(CROSS_COMPILE)gcc
endif

all:$(TARGET)

$(TARGET): $(STDLIB_OBJS) $(OBJS)
	$P "  LD    $@"
	$E $(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^ $(LIBS)
	$P "  AXF   $(TARGET:.elf=.axf)"
	$E cp $@ $(TARGET:.elf=.axf)
	$P "  HEX   $(TARGET:.elf=.hex)"
	$E $(OBJCOPY) -O ihex $@ $(TARGET:.elf=.hex)
	$E $(SIZE) $@

%.o : %.c
	$P "  CC    $@"
	$E $(CC) $(CFLAGS) $(EXTRA_FLAGS) -c -o $@ $<

%.o : %.S
	$P "  AS    $@"
	$E $(CC) -x assembler-with-cpp $(CFLAGS) $(EXTRA_FLAGS) -c -o $@ $<

%.depends: %.c
	$P "  DEP   $@"
	$E $(CC) $(CFLAGS) -MM -MP -MT $(patsubst %.c,%.o,$<)  -o $@ $<

clean:
	$E rm -rf $(TARGET) $(TARGET:.elf=.axf) $(TARGET:.elf=.hex) $(TARGET:.elf=.dfu) testdri.bin $(OBJS) $(STDLIB_OBJS) $(TARGET:.elf=.map) $(CPPTESTBDF) $(CPPTESTWORKSPACE) $(CPPTESTREPORT)

distclean: clean
	$E rm -rf $(DEPS)

debug:$(TARGET)
	@echo "Launch GDB with :"
	@echo "${CROSS_COMPILE}gdb --eval-command=\"target remote localhost:2331\" $(TARGET)"
	@echo "-----------------"
	@$(JLINKGDBSERVER) -if SWD -device $(JLINKDEVICE)

-include $(DEPS)

.PHONY : clean debug burn doc pdf cpptest

# End of Makefile
```

