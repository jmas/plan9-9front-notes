# Q&A

Q: [9front install on Asus Eee PC x101ch](https://www.reddit.com/r/plan9/comments/m8d2iu/comment/grkwrcc/)
```
ook, had the same problem with the latest version as well(8357), headed at cat-v, got it solved!
bashed the spacebar to prevent the bootloader from automatically run the boot command, so I get [...] bootfile=/386/9pc, and then a prompt with >
*acpi=, and then boot
no bootargs - vgasize640x480x8 - monitorvesa
now when it asks for mouseport, either ps2 or ps2intellimouse DO WORK!
```
