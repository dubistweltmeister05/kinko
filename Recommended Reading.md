https://www.reddit.com/r/linux/comments/coi4dt/a_complete_guide_of_and_debunking_of_audio_on/

```
scp out/arch/arm64/boot/Image vicharak@192.168.1.69:~/
scp out/arch/arm64/boot/dts/rockchip/rk3399-vaaman-linux.dtb vicharak@192.168.1.69:~/
scp -r out/arch/arm64/boot/dts/rockchip/overlays vicharak@192.168.1.69:~/
scp out/modules_rk3399_vaaman.tar.gz vicharak@192.168.1.69:~/
```


        es8316-sound {  
 78                 compatible = "simple-audio-card";  
 79                 simple-audio-card,format = "i2s";  
 80                 simple-audio-card,name = "rockchip,es8316-codec";  
 81                 simple-audio-card,mclk-fs = <256>;  
 82                 simple-audio-card,widgets =  
 83                         "Microphone", "Mic Jack",  
 84                         "Headphone", "Headphone Jack";  
                  rockchip,routing =  
                          "Mic Jack", "MICBIAS1",  
                          "IN1P", "Mic Jack",  
                          "Headphone Jack", "HPOL",  
                          "Headphone Jack", "HPOR";  
 90                 simple-audio-card,cpu {  
 91                         sound-dai = <&i2s0>;  
 92                         system-clock-frequency = <11289600>;  
 93                 };  
 94                 simple-audio-card,codec {  
 95                         sound-dai = <&es8316>;  
 96                         system-clock-frequency = <11289600>;  
 97                 };  
 98         };

v4l2-ctl --verbose -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat='NV12' --stream-mmap=4 --set-selection=target=crop,flags=0,top=0,left=0,width=1920,height=1080 --stream-to=sample.yuv