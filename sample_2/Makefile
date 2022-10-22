APP ?= app/test1
OUTPUT_PATH := output

INCLUDE_PATH := -I$(APP) -Idriver -I$(OUTPUT_PATH)

# define user .config setting
USER_CONFIG_SET := 
USER_CONFIG_SET += $(APP)/prj.conf

# define menuconfig .config path
DOTCONFIG_PATH := $(OUTPUT_PATH)/.config

# define user merged path
USER_RECORD_CONFIG_PATH := $(OUTPUT_PATH)/user_record.conf

# define autoconfig.h path
AUTOCONFIG_H := $(OUTPUT_PATH)/autoconfig.h

#define Kconfig path
KCONFIG_ROOT_PATH := Kconfig

#define need hardwritten
CONFIG_HANDWRITTEN := --handwritten-input-configs


#For windows work.
FIXPATH = $(subst /,\,$1)


all: $(APP)/main.o driver/driver_test.o
	gcc $^ -o $(OUTPUT_PATH)/main.exe
$(APP)/main.o: $(APP)/main.c
	gcc $< $(INCLUDE_PATH) -c -o $@
driver/driver_test.o: driver/driver_test.c $(AUTOCONFIG_H)
	gcc $< $(INCLUDE_PATH) -c -o $@

clean:
	del /q /s $(call FIXPATH, $(APP)/main.o driver/driver_test.o $(OUTPUT_PATH))

$(AUTOCONFIG_H):$(DOTCONFIG_PATH)
	python ../scripts/kconfig.py $(CONFIG_HANDWRITTEN) $(KCONFIG_ROOT_PATH) $(DOTCONFIG_PATH) $(AUTOCONFIG_H) $(OUTPUT_PATH)/log.txt $(DOTCONFIG_PATH)

$(USER_RECORD_CONFIG_PATH): $(USER_CONFIG_SET)
	@echo Using user config.
#	create user_record.conf to record current setting.
	@copy $(call FIXPATH, $^) $(call FIXPATH, $@)
#	create .config by user config setting.
	python ../scripts/kconfig.py $(CONFIG_HANDWRITTEN) $(KCONFIG_ROOT_PATH) $(DOTCONFIG_PATH) $(AUTOCONFIG_H) $(OUTPUT_PATH)/log.txt $(USER_CONFIG_SET)

export KCONFIG_CONFIG=$(DOTCONFIG_PATH)
$(DOTCONFIG_PATH):$(USER_RECORD_CONFIG_PATH)
	@echo .config updated

menuconfig:$(DOTCONFIG_PATH)
#	set KCONFIG_CONFIG=$(DOTCONFIG_PATH)
	menuconfig $(KCONFIG_ROOT_PATH)