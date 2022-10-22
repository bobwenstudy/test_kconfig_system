all: main.o
	gcc main.o -o main
main.o: main.c autoconfig.h
	gcc main.c -c -o main.o
clean:
	del main.o main.exe

autoconfig.h:.config
	python ../scripts/kconfig.py Kconfig .config autoconfig.h log.txt .config
.config:
	menuconfig
menuconfig:
	menuconfig