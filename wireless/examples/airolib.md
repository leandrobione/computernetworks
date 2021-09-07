crear archivo `essid.txt` con los essid wlan

crear archivo wordlist `pass.txt`

## airolib-ng
comandos

```
airolib-ng rainbowtable --import essid essid.txt`

airolib-ng` rainbowtable --import passwd pass.txt`

airolib-ng rainbowtable --stat

airolib-ng rainbowtable --batch

airolib-ng rainbowtable --sql 'select * from essid'

airolib-ng rainbowtable --sql 'select * from passwd'

airolib-ng rainbowtable --sql 'select * from pmk'

airolib-ng rainbowtable --clean all 
```

## comparations
without rainbowtable `aircrack-ng -w pass.txt SCAN.cap` 

with `aircrack-ng -r rainbowtable SCAN.cap`


