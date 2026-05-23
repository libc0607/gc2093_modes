# gc2093_modes
Use GC2093 with SSC338Q/SSC30KQ(/SSC377/SSC378/...) + OpenIPC

**DISCLAIMER: The driver contains overclocking and may damage your hardware. Use it at your own risk!**  

## Quick test 
 - Copy ```sensor_gc2093_mipi*.ko``` (suffix difference see below) to VTX ```/lib/modules/4.9.84/sigmastar/```
 - Run ```fw_setenv sensor gc2093``` on VTX
 - Add ```gc2093``` to VTX ```/usr/bin/load_sigmastar``` (add it to the only line with several sensor names) 
 - Copy some IQ file (e.g. gc2053's) to VTX```/etc/sensors/``` and name it to `gc2093.bin`
 - Reboot & Enjoy

## Sensor modes 
**Use overclocked configs at your own risk!**  
For reference, pclk: 
 - < 100 MHz: Safe
 - 100 ~ 120 MHz: Overclocked, seen in commercial SDK, most can do under default voltage, does not deteriorate image quality much
 - \> 120 MHz: Overclocked, may not work on some PCBs & will impact image quality in low-light 

Set the resolution and framerate in ```/etc/majestic.yaml```:

|res_idx|Resolution|Ratio|Framerate|PCLK|FOV|Line_Length|Frame_Length|
|---|---|---|---|---|---|---|---|
|0|1920x1080|16:9|60 fps|88.875 MHz|Full|657|1127|
|1|1920x1080|16:9|80 fps|118.125 MHz|Full|657|1125|
|2|1920x1080|16:9|90 fps|132.750 MHz|Full|657|1122|
|3|1920x864|20:9|100 fps|118.125 MHz|Cropped top/bottom|657|899|
|4|1920x768|2.5:1|120 fps|126.000 MHz|Cropped top/bottom|657|799|
|5|1280x720|16:9|120 fps|119.250 MHz|Cropped center|657|750|


## Hardware 
Note: Match the MIPI length carefully when doing PCB layout. It only has 2 lanes, so quite vulnerable to in/between differential pairs mismatch than imx415/imx335.  

### Datasheet & HW Design Guides
1/2.9 inch, 2MP, 60 fps default, 2.8um x 2.8um FSI pixel  
Chip Datasheet: [GC2093_CSP_Datasheet_V1.1_20201012.pdf](https://github.com/libc0607/gc2093_modes/blob/main/hardware/GC2093_CSP_Datasheet_V1.1_20201012.pdf)  
Schematic & HW design guides: [GC2053 CSP 模组设计指南 V1.2 20181212.pdf](https://github.com/libc0607/gc2093_modes/blob/main/hardware/GC2053%20CSP%20%E6%A8%A1%E7%BB%84%E8%AE%BE%E8%AE%A1%E6%8C%87%E5%8D%97%20V1.2%2020181212.pdf) (Pin-to-Pin with GC2053)  
\* All collected from the Internet.  

### Open Source Hardware 
[Search gc2093 on OSHWHub](https://oshwhub.com/search?wd=gc2093)  
Disclaimer: Homemade sensor modules can lead to many image quality issues.  

### Sensor Modules
Some compatible sensor modules could be used in your custom VTX design, i.e. :  
**LCKFB-GC2093-200W-MIPI-V1.0-B**: [立创商城](https://item.szlcsc.com/44319754.html) / [LCSC](https://www.lcsc.com/product-detail/C42388919.html) -- Recommend.   
[Integrated LDOs & OSC](https://github.com/user-attachments/assets/738ba6d3-0871-44d0-8310-88e7695c3ec0), 14mmx14mm, f=3.0mm distortion-free lens, 35.9 CNY / 5.5 USD  
**LCKFB-GC2093-200W-MIPI-V1.0-L**: [立创商城](https://item.szlcsc.com/44319753.html) / [LCSC](https://www.lcsc.com/product-detail/C42388918.html)  

As GC2093 seems to be one of the 'standard setups' for Kendryte K230 EVBs, there are many cheap compatible sensor modules made for them. (Try searching on Taobao)  
Almost all these modules share the same 22P FPC definition, and have integrated 1V2/1V8/2V8 LDOs and a 24MHz active crystal oscillator.  

When using those modules with SigmaStar SoCs, you should: 
 - Connect 3V3, GND, SDA/SCL/RST, MIPI CLK_[N/P] & D[0:1]_[N/P]; Leave the clock output from SoC floated 
 - Use driver file with ```_24M``` suffix for these modules to get the correct framerate  

## IQ file
Use [gc2053.bin](https://github.com/OpenIPC/firmware/blob/master/general/package/sigmastar-osdrv-infinity6e/files/sensor/configs/gc2053.bin) first -- it's good enough.  
Or try to tune it with IQTool.  

I will not provide any IQ file here. People have different setups and have different problems (usually caused by different 'money-saving tips').   
The overall image quality can be affected by many factors. Lens, coating of the glass, color filter, ~~Shantou deep fried~~ brand new LDO's low PSRR, light leaking through your free green PCB from JLCPCB (...), ISP structure & tuning, temperature, video bitrate, and your display...  
So don't judge by any single factor. It's a complex system. )  

The driver will NOT decrease framerate in low light, even if a long exposure time in IQ bin is set. The maximum number of exposure lines is limited to one full frame at the current framerate.  
Some drivers use a lower FPS to have longer exposure time; This is for FPV -- so it will not. 

## Theory of Overclocking 
GC2093 has several init register arrays with comments available (see References).   
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
Line_Length * Frame_Length * Frame_Rate * 2 = (pllmp_div / 24) * mclk = pclk
```
For example, 1920x768 120fps:
```
657(Line_Length) * 799(Frame_Length) * 120.013(fps) * 2 = (0x70(pllmp_div)/24) * 27,000,000(Hz) = 126,000,000(Hz)
```
In my & 侧面(cemian)'s tests, the sensor can be safely overclocked by 30%~40% at Vcore=1.20V, then start flicking when higher than ~140MHz.    

## References 
[drv_gc2093.c in LuckfoxTECH/luckfox-pico](https://github.com/LuckfoxTECH/luckfox-pico/blob/main/sysdrv/source/mcu/rt-thread/bsp/rockchip/common/drivers/camera/drv_gc2093.c)  
[gc2093.c in kendryte/k230_sdk](https://github.com/kendryte/k230_sdk/blob/main/src/big/mpp/kernel/sensor/src/gc2093.c)  

## Build 
Find anyone who has the Pudding SDK (specifically, V042, if using with OpenIPC SSC338Q), then ask for help  

To strip the debug symbols to reduce `.ko` size, run `arm-linux-gnueabihf-9.1.0-strip --strip-debug gc2093_MIPI.ko`  
