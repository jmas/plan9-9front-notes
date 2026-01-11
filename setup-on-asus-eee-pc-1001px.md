# Installation Plan 9 / 9front on Asus Eee PC 1001px

## References
- [Hardware section Q&A 9front](https://fqa.9front.org/fqa3.html#3.2)

## Q&A

Q: Why old hardware?
Because old hardware means that old bugs are fixed and I have less issues with drivers.

Q: Potential issues?
- Could be issues with Wi-Fi. Solution: use Ethernet connection and buy few popular Wi-Fi modules for testing.
- Could be issues with mouse/trackpad. Solution: test external USB mouse (I have Logitech G203) and buy popular one for testing. 

Q: [9front install on Asus Eee PC x101ch](https://www.reddit.com/r/plan9/comments/m8d2iu/comment/grkwrcc/)
```
ook, had the same problem with the latest version as well(8357), headed at cat-v, got it solved!
bashed the spacebar to prevent the bootloader from automatically run the boot command, so I get [...] bootfile=/386/9pc, and then a prompt with >
*acpi=, and then boot
no bootargs - vgasize640x480x8 - monitorvesa
now when it asks for mouseport, either ps2 or ps2intellimouse DO WORK!
```

## Hardware modules

- [Buy Wi-Fi module #1 - Unknown - 30 UAH](https://shop.zapara.com.ua/ua/p1829289372-adapter-snyatyj-noutbuka.html)
- [Buy Wi-Fi module #2 - AzureWave AR5B95 - 55 UAH](https://prom.ua/ua/p1935857911-modul-azurewave-ar5b95.html)
