# gc2093_modes
Use GC2093 with SSC338Q + OpenIPC

**DISCLAIMER: The driver contains overclocking and may damage your hardware. Use it at your own risk!**  

## Quick test 
 - Copy ```sensor_gc2093_mipi.ko``` to VTX ```/lib/modules/4.9.84/sigmastar/```
 - Run ```fw_setenv sensor gc2093``` on VTX
 - Add ```gc2093``` to VTX ```/usr/bin/load_sigmastar``` (add it to the only line with several sensor names) 
 - Copy some IQ file (e.g. gc2053's) to VTX```/etc/sensors/```
 - Reboot & Enjoy

## Sensor modes 
Set the resolution and framerate in ```/etc/majestic.yaml```:
### 1920x1080 60fps 
Safe.  
16:9, full size, vts=1127, pclk=88.875MHz  
### 1920x768 120fps 
Overclocked. Use it at your own risk.   
2.5:1, cropped at top & bottom, vts=799, pclk=126.000MHz  
### 1920x1080 90fps 
Overclocked. Use it at your own risk.  
16:9, full size, vts=1122, pclk=132.750MHz  

## Hardware 
1/2.9 inch, 2MP, 60fps, 2.8um x 2.8um pixel  
Datasheet: [GC2093_CSP_Datasheet_V1.1_20201012.pdf](https://github.com/libc0607/gc2093_modes/blob/main/hardware/GC2093_CSP_Datasheet_V1.1_20201012.pdf)  
Schematic & HW design guides: [GC2053 CSP 模组设计指南 V1.2 20181212.pdf](https://github.com/libc0607/gc2093_modes/blob/main/hardware/GC2053%20CSP%20%E6%A8%A1%E7%BB%84%E8%AE%BE%E8%AE%A1%E6%8C%87%E5%8D%97%20V1.2%2020181212.pdf) (Pin-to-Pin with GC2053)  
\* All collected from Internet.  

Note: Match the MIPI length carefully when doing PCB layout. It only has 2 lanes, so quite vulnerable to in/between differential pairs mismatch than imx415/imx335.   

## IQ file
Use gc2053.bin first -- it's good enough.  
Or try to tune it with IQTool.  

## Theory
GC2093 has several init register arrays available (see References).   
So, by some guess and calculation, it is possible to overclock the sensor & get higher framerates.

From the datasheet, we have: 
```
Line_Length[11:0] = {reg_0x0005[3:0], reg_0x0006[7:0]};
Frame_Length[13:0] = {reg_0x0041[5:0], reg_0x0042[7:0]};
```
From the GC2053 driver, guess GC2093 has: 
```
pllmp_div[7:0] = reg_0x03f8[7:0];
```
On hardware, we have a 27MHz clock, from the SoC to the sensor:  
```
#define Preview_MCLK_SPEED  CUS_CMU_CLK_27MHZ

// mclk
handle->mclk = Preview_MCLK_SPEED;

sensor_if->MCLK(idx, 1, handle->mclk);
```
So, we can find an equation. When other pll-related registers not touched, we have:  
```
Line_Length * Frame_Length * Frame_Rate = (pllmp_div / 48) * mclk
```
For example, 1920x768 120fps:
```
657(Line_Length) * 799(Frame_Length) * 120.013(fps) = (0x70(pllmp_div)/48) * 27,000,000(Hz)
```
In my & 侧面(cemian)'s tests, the sensor can be safely overclocked by 30%~40% at Vcore=1.20V, then start flicking when higher than ~140MHz.    

## References 
[drv_gc2093.c in LuckfoxTECH/luckfox-pico](https://github.com/LuckfoxTECH/luckfox-pico/blob/main/sysdrv/source/mcu/rt-thread/bsp/rockchip/common/drivers/camera/drv_gc2093.c)  
[gc2093.c in kendryte/k230_sdk](https://github.com/kendryte/k230_sdk/blob/main/src/big/mpp/kernel/sensor/src/gc2093.c)  

## Build 
Find anyone who has the Pudding SDK, then ask for help 
