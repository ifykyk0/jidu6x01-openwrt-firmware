# JioRouter AX6000 par OpenWrt ka Custom Build: official Firmware Selector se step by step

*(English summary neeche hai / English summary below)*

Is guide se aap OpenWrt ki **official website** par apni pasand ke packages ke saath apna **custom firmware** bana kar download karoge, uska hash verify karoge, aur use apne **JioRouter AX6000** par flash karoge. Jaise Tailscale, AdBlock, apna timezone, aur router ka sahi model (JIDU6401) dikhana.

> ## ⚠️ Ye guide kin ke liye hai
> **Sirf unke liye jinke router par pehle se OpenWrt chal raha hai** (sysupgrade se upgrade/rebuild karna).
>
> **Jio ke stock firmware se pehli baar OpenWrt lagana is guide me nahi hai.** Wo alag aur risky kaam hai. Ek community forum thread me bataya gaya hai ki Jio AirFiber routers ka stock bootloader signed hota hai aur official OpenWrt images reject kar sakta hai, aur logon ne uske liye UART/TFTP jaise tareeke use kiye (kuch ke router brick bhi hue). Is tarah ke case me forum thread aur OpenWrt ke Table of Hardware page par pehle padho.
>
> Galat image flash karne se router **brick** ho sakta hai. Isliye Step 1 (backup) skip mat karna.

> **Last tested:** October 2026. JioRouter AX6000 (JIDU6401, LuCI me JIDU6J01 dikhta hai), OpenWrt 25.12.6, Windows PC. Doosre version me website ya LuCI ke naam thode alag ho sakte hain.
>
> Ye ek unofficial community guide hai. Jio ya OpenWrt se iska koi sambandh nahi. Apne risk par use karo.

---

## TL;DR: 10 line me poora kaam

1. Confirm karo ki router par **pehle se OpenWrt** chal raha hai (Step 0).
2. **Backup** lo: config, aur chaho to partitions (Step 1).
3. `firmware-selector.openwrt.org` kholo aur apna router chuno, **stable version** ke saath (Step 2).
4. **Customize installed packages** me apne packages end me jodo, default list mat badlo (Step 3).
5. **First boot script** me timezone aur model-detect script paste karo (Step 4).
6. **Request Build** dabao, aur "Build successful" ka intezaar karo (Step 5).
7. **SYSUPGRADE** wali file download karo, **Factory** wali nahi. SHA256 note karo (Step 6).
8. PC par file ka **hash match** karo (Step 7).
9. LuCI me **Flash image** karo, **Keep settings** tick, **Force upgrade** untick (Step 8).
10. Flash ke baad checks karo, reboot karo, aur naya backup lo (Step 9, 10).

---

## Ye kaise kaam karta hai

```
Aap: packages + first-boot script chunte ho
        |
        v
OpenWrt ka official Firmware Selector (server par build hota hai)
        |
        v
Aapko ek .bin (sysupgrade) file milti hai + uska SHA256
        |
        v
PC par hash match -> LuCI me Flash image -> router naye firmware par
```

Build OpenWrt ke apne servers par hota hai, isliye isme kisi anjaan insaan ki banayi hui binary nahi aati. Fir bhi hash verify karna zaroori hai.

---

## Zaroori shabdon ka matlab

| Shabd | Matlab |
|---|---|
| **Firmware Selector** | OpenWrt ki official website jahan se aap apne router ke liye firmware (packages ke saath) bana sakte ho |
| **Package** | Ek chhota software/feature, jaise `tailscale` ya `adblock-fast` |
| **Sysupgrade image** | Wo firmware file jo pehle se OpenWrt chala rahe router par upgrade ke liye hoti hai (`...-squashfs-sysupgrade.bin`) |
| **Factory image** | Wo file jo pehli baar install ke liye hoti hai. **Is guide me use nahi karni** |
| **First boot script** | Ek chhota script jo firmware flash ke baad pehli boot par apne aap chalta hai |
| **SHA256 / hash** | File ka "fingerprint". Download ki file ka hash website wale se match kare to file sahi aur poori hai |
| **LuCI** | Router ka web panel (`http://192.168.1.1`) |
| **SSH** | Router me commands type karne ka tareeka (Windows me PowerShell se) |
| **Keep settings** | Flash ke baad purani settings (WiFi, password, etc.) bacha ke rakhna |

---

## Kya chahiye (checklist)

- [ ] Router par **OpenWrt** chal raha ho
- [ ] **Windows PC / Laptop**, router se **LAN cable** se juda ho
- [ ] **Router ka password** pata ho
- [ ] **Bijli ka backup** (flash ke dauran bijli gayi to router brick ho sakta hai, laptop ko charger par lagao)
- [ ] 30-40 minute ka shaanti ka time

---

## SSH kaise kholte hain (jahan guide me "SSH karo" likha ho)

**Windows:**
1. **Windows key** dabao, `PowerShell` type karo, **Enter** dabao.
2. Ye likho aur Enter dabao:
   ```
   ssh root@192.168.1.1
   ```
3. Pehli baar "Are you sure you want to continue connecting" poochega: `yes` likho.
4. **Router ka password** type karo. Type karte waqt **kuch dikhta nahi**, ye normal hai. Enter dabao.
5. `root@OpenWrt:~#` dikhe to aap router ke andar ho.

Commands ko copy karke SSH window me **right-click** karke paste karo, aur Enter dabao.

> Agar **"REMOTE HOST IDENTIFICATION HAS CHANGED"** ka error aaye (naya firmware flash karne ke baad aisa hota hai), to apne PC ke PowerShell me ye chalao aur dobara SSH karo:
> ```
> ssh-keygen -R 192.168.1.1
> ```

---

## Step 0: Confirm karo ki router par pehle se OpenWrt hai

1. Browser me `http://192.168.1.1` kholo aur login karo.
2. **Status → Overview** dekho.

**Kya dikhna chahiye:**
- **Model:** `JioRouter AX6000 JIDU6J01` (ya `...6401`)
- **Firmware Version:** `OpenWrt ...` (jaise `OpenWrt 25.12.x`)

> Dhyan rakho: JIDU6J01, 6201, 6401, 6601 aur 6701, in sab ke liye **ek hi image** hoti hai. Isliye LuCI aksar `JIDU6J01` dikhata hai chahe aapka router 6401 ho. Ye normal hai.

Chahe to SSH me ye bhi dekh sakte ho:
```
ubus call system board
```
Isme `board_name` `jiorouter,ax6000-jidu6j01` dikhna chahiye.

<!-- ![Step 0: Status Overview](screenshots/fw-step-00-status.png) -->

---

## Step 1: Backup lo (flash se pehle zaroori)

### 1A. Config backup (sabke liye zaroori)
1. LuCI → **System → Backup / Flash Firmware**.
2. **Generate archive** dabao.
3. Jo `backup-....tar.gz` file download ho, use PC me kisi safe folder me rakh lo (jaise Desktop par `jio-backup`).

> Is file me WiFi password aur settings hoti hain. **Kisi ko mat bhejo, aur GitHub par kabhi upload mat karo.**

### 1B. Partition backup (advanced, par karne layak hai)
Isme router ki WiFi calibration (`Factory`), bootloader aur manufacturing data hota hai. Ye agar kharab ho jaye to wapas nahi milta, isliye iska backup sabse kimti hai.

Pehle router par SSH me dekho ki partitions ke numbers kya hain:
```
cat /proc/mtd
```
Is router par aam taur par ye dikhta hai: `mtd0 BL2`, `mtd1 u-boot-env`, `mtd2 Factory`, `mtd3 FIP`, `mtd4 ubi`, `mtd5 ubi2`, `mtd6 MFG`, `mtd7 Reserved`. **Apne router par check karo, aur numbers alag hon to apne numbers use karo.**

Phir router par ye chalao (ye sirf padhta hai, kuch badalta nahi):
```
mkdir -p /tmp/bak
for i in 0 1 2 3 6 7; do cat /dev/mtd$i > /tmp/bak/mtd$i.bin; done
cat /dev/mtd5 > /tmp/bak/mtd5-ubi2.bin
sha256sum /tmp/bak/*.bin > /tmp/bak/sha256.txt
ls -lh /tmp/bak
```

Ab **naya PowerShell window** kholo (jiska prompt `PS C:\Users\...>` ho, `root@OpenWrt` nahi) aur ye chalao:
```
mkdir $HOME\Desktop\jio-backup
scp -O -r root@192.168.1.1:/tmp/bak/* $HOME\Desktop\jio-backup\
Get-FileHash $HOME\Desktop\jio-backup\*.bin -Algorithm SHA256
```
`Get-Content $HOME\Desktop\jio-backup\sha256.txt` se router wale hash dekh lo. **Har file ka hash dono jagah same hona chahiye.**

> Ye files **/tmp me hain, jo RAM hai**. Copy karne se pehle router reboot mat karna, nahi to chali jayengi.
>
> **`MFG` aur `Reserved` partition me device ka private data hota hai** (serial, kuch credentials jaise strings). Inke backup ko aur inke `strings` output ko kisi public jagah (GitHub, forum, Telegram) par **kabhi share mat karo.** Google Drive me private rakh sakte ho.

<!-- ![Step 1: Generate archive](screenshots/fw-step-01-backup.png) -->

---

## Step 2: Firmware Selector kholo aur router chuno

1. Browser me kholo: `https://firmware-selector.openwrt.org`
2. Search box me type karo: `JioRouter AX6000` aur list me se apna model chuno (jaise **JioRouter AX6000 JIDU6401**).
3. Search box ke saamne **version dropdown** hai. Yahan **latest stable version** chuno (jaise `25.12.6`). **SNAPSHOT** mat chuno, wo roz badalne wala test build hota hai aur bug ho sakte hain.

**Kya dikhna chahiye:** **"About this build"** me:
- **Model:** `JioRouter AX6000 JIDU6J01 / ...6201 / ...6401 / ...6601 / ...6701` (ek hi build in sab ke liye)
- **Platform:** `mediatek/filogic`
- **Version:** jo aapne chuna (jaise `25.12.6`)

<!-- ![Step 2: Select device and version](screenshots/fw-step-02-select.png) -->

---

## Step 3: Packages chuno (Customize installed packages)

1. Usi page par **"Customize installed packages and/or first boot script"** par click karke kholo.
2. **Installed Packages** ka ek badaa box dikhega, jisme pehle se packages ki list hogi.

> **Sabse zaroori niyam:** jo list pehle se hai usse **mat badlo aur kuch hatao mat.** Usme router ke WiFi, network aur kernel ke zaroori packages hain. Sirf **end me apne packages ek space dekar jodo.**

Is tested build me ye list thi (aapki default list thodi alag ho sakti hai, aap sirf end ka hissa dekho):

```
apk-mbedtls base-files ca-bundle dnsmasq dropbear firewall4 fitblk fstools kmod-crypto-hw-safexcel kmod-gpio-button-hotplug kmod-leds-gpio kmod-nft-offload libc libgcc libustream-mbedtls logd mtd netifd nftables odhcp6c odhcpd-ipv6only ppp ppp-mod-pppoe procd-ujail uboot-envtools uci uclient-fetch urandom-seed urngd wpad-basic-mbedtls kmod-usb3 kmod-mt7915e kmod-mt7986-firmware mt7986-wo-firmware luci luci-app-attendedsysupgrade tailscale luci-app-tailscale-community adblock-fast luci-app-adblock-fast
```

Yahan **sirf in 4 ko end me joda gaya** hai:

| Package | Kya karta hai |
|---|---|
| `tailscale` | Bahar se router access (VPN-jaisa) |
| `luci-app-tailscale-community` | Tailscale ka panel LuCI me |
| `adblock-fast` | DNS se ads block |
| `luci-app-adblock-fast` | AdBlock ka panel LuCI me |

(`luci` router ka web panel hai aur `luci-app-attendedsysupgrade` LuCI se update karne ka option deta hai. Dono default me ho sakte hain.)

> **Purana naam galat hai:** `luci-app-tailscale` ab OpenWrt 25.12 ke official feed me nahi hai. Usse build **fail** hota hai ("Impossible package selection: missing (luci-app-tailscale)"). Uski jagah **`luci-app-tailscale-community`** likho.
>
> **Theme (jaise Argon) is build me nahi daal sakte.** Jahan tak maine dekha wo official feed me nahi hai, third-party package hai.
>
> Package ka naam galat likha to Step 5 me error aayega. Wahi error batayega ki kaunsa naam hatana hai.

<!-- ![Step 3: Installed packages box](screenshots/fw-step-03-packages.png) -->

---

## Step 4: First boot script (timezone aur model detection)

Usi page par neeche **"Script to run on first boot (uci-defaults)"** ka box hai. Isme ye **poora** paste karo (pehle se kuch likha ho to use hatakar):

```sh
#!/bin/sh
# Timezone: India
uci set system.@system[0].timezone='IST-5:30'
uci set system.@system[0].zonename='Asia/Kolkata'
uci commit system

# Model detection: MFG partition se variant padh ke LuCI me dikhao
cat > /etc/init.d/model-detect <<'EOF'
#!/bin/sh /etc/rc.common
START=99
start() {
    MTD=$(grep '"MFG"' /proc/mtd | cut -d: -f1)
    [ -n "$MTD" ] || return 0
    V=$(strings /dev/$MTD | grep -m1 -o 'JIDU[0-9A-Z]\{4\}')
    [ -n "$V" ] || return 0
    echo "JioRouter AX6000 $V" > /tmp/sysinfo/model
}
EOF
chmod +x /etc/init.d/model-detect
/etc/init.d/model-detect enable
```

**Ye kya karta hai:**
- **Timezone** ko India (`IST-5:30`) par set karta hai.
- **Model detection:** router ke `MFG` partition se aapka asli model (jaise `JIDU6401`) padhta hai (sirf padhta hai, kuch likhta nahi) aur LuCI ke Status page par wahi dikhata hai. Model na mile to chup-chaap kuch nahi karta.

> **Isme kabhi password, Tailscale key ya koi secret mat daalo.** Firmware Selector ke builds server par cache hote hain.
>
> Ye script **sirf is router (JioRouter AX6000) ke liye** hai. Dusre device par `MFG` partition nahi hoga, to script kuch nahi karegi.
>
> Pehli flash ke baad, kabhi-kabhi Status page par model purana (`JIDU6J01`) hi dikhta hai jab tak router ek baar **reboot** na ho (Step 10).

<!-- ![Step 4: First boot script box](screenshots/fw-step-04-script.png) -->

---

## Step 5: Build request karo

1. Neeche **REQUEST BUILD** button dabao.
2. 1-3 minute ruko. Page par ek progress dikhega.

**Kya dikhna chahiye:** neele/hare box me **"Build successful"** likha aayega. Phir **Download Firmware Images** wala hissa dikhega.

**Agar laal error aaye** (jaise `Impossible package selection: missing (...)`):
- Bracket ke andar wala naam galat/ab-nahi-hai wala package hai. Use **Installed Packages** me se hatao (ya sahi naam se badlo, jaise `luci-app-tailscale` → `luci-app-tailscale-community`) aur dobara **Request Build** dabao.
- `STDERR` par click karke poora error bhi dekh sakte ho.

<!-- ![Step 5: Build successful](screenshots/fw-step-05-build.png) -->

---

## Step 6: Sahi file download karo

1. **Download Firmware Images** me do tarah ki files hongi: **SYSUPGRADE** aur **FACTORY** (ya kuch isi tarah ke naam).
2. **SYSUPGRADE** wali file download karo. Naam kuch aisa hoga:
   ```
   openwrt-25.12.6-<kuch-hash>-mediatek-filogic-jiorouter_ax6000-jidu6j01-squashfs-sysupgrade.bin
   ```
3. Us file ke paas ya page par **`sha256sum:`** ke saamne jo lamba hash likha hai, use copy karke ek notepad me rakh lo.

> **Factory wali file mat lena.** Wo pehli install ke liye hoti hai, aapke chalte hue OpenWrt par upgrade ke liye nahi.
>
> Chrome me ek hi naam ki dobara download hui file ke naam me `(1)`, `(2)` jud jata hai. Isliye **nayi file ko ek saaf alag folder me** rakho (jaise `Desktop\flash`). Aapke Downloads me purani files ho to galti se wo flash na ho.

<!-- ![Step 6: Download sysupgrade and sha256](screenshots/fw-step-06-download.png) -->

---

## Step 7: Hash match karo (zaroori, 1 minute)

Ye check karta hai ki download hui file bilkul sahi hai.

1. File ko saaf folder me rakho: `Desktop\flash`.
2. **PowerShell** (`PS C:\Users\...>` wale window) me ye chalao:
   ```
   mkdir $HOME\Desktop\flash
   Get-FileHash $HOME\Desktop\flash\*.bin -Algorithm SHA256
   ```
   (Pehle file ko `Downloads` se `Desktop\flash` me copy kar do.)
3. Jo `Hash` aaye usse Step 6 me copy kiye hue `sha256sum` se compare karo.

**Kya dikhna chahiye:** dono hash **bilkul same** (PC wala capital letters me aayega, usse farak nahi padta).

> **Folder me sirf ek `.bin` file honi chahiye.** Do files hui to PowerShell do hash dikhayega, aur confusion hoga.
>
> **Hash match na ho to flash mat karo.** File dobara download karo.

---

## Step 8: Flash karo

Laptop charger par rakho, LAN cable se juda rakho, aur ye karo:

1. LuCI → **System → Backup / Flash Firmware**.
2. **Flash new firmware image** ke saamne **Flash image...** dabao.
3. `Desktop\flash` wali `.bin` file chuno aur **Upload** karo.
4. Agle screen par **checksum** dikhega. Use apne hash se match karo (shuruat ke kuch characters dekh lo).
5. **"Keep settings"** ko **tick** rakho (purani WiFi/password settings bachengi).
6. **"Force upgrade"** ko **tick mat karo.**
7. **Continue** dabao.
8. **3-4 minute ruko.** Is dauran **bijli, LAN cable ya router ko touch mat karo**, aur page ko baar-baar refresh mat karo.

> **Agar "Image check failed" ya koi warning aaye to Force mat dabana.** Ruk jao, aur check karo ki aapne **sysupgrade** wali sahi file chuni hai ya nahi. Phir bhi samajh na aaye to error ka screenshot lekar madad maango (private cheezein chhupa ke).

<!-- ![Step 8: Flash image screen](screenshots/fw-step-08-flash.png) -->

---

## Step 9: Flash ke baad check karo

`http://192.168.1.1` kholo (na khule to 5 minute aur ruko) aur login karo. Browser me **Ctrl + F5** dabao.

**Kya dikhna chahiye:**
- **Status → Overview** me **Firmware Version** naya (jaise `25.12.6`)
- Local Time me **GMT+5:30**
- **Services** menu me aapke packages (Tailscale, AdBlock-Fast)

Status page par ek popup aa sakta hai: **"Check online for firmware upgrades"**. Ye `attendedsysupgrade` ka hai.
- **No, disable checking** chuno agar nahi chahiye ki router baar-baar online update check kare.
- **Yes, enable checking** chuno agar chahiye.
Ye sirf ek setting hai, isse firmware nahi badalta. Baad me **System → Attended Sysupgrade** se badal sakte ho.

> **Tailscale ka login chala gaya ho** to use dobara karna padega. Uski poori guide yahan hai: https://github.com/ifykyk0/Tailscale
>
> **Model abhi bhi `JIDU6J01` dikhe** to Step 10 karo (reboot).

---

## Step 10: Reboot aur final backup

1. **System → Reboot → Perform reboot**.
2. 2-3 minute ruko, `http://192.168.1.1` kholo, **Ctrl + F5** dabao.
3. **Status → Overview** me **Model** me ab aapka asli model (jaise `JioRouter AX6000 JIDU6401`) dikhna chahiye.

**Model phir bhi na badle** to SSH me ye chalao:
```
ls -l /etc/init.d/model-detect
/etc/init.d/model-detect enabled && echo enabled
/etc/init.d/model-detect start
cat /tmp/sysinfo/model
```
Last command me `JIDU6401` (ya aapka model) dikhe to script sahi hai. LuCI page ko **Ctrl + F5** se refresh karo.

Phir **System → Backup / Flash Firmware → Generate archive** se naya backup le lo.

---

## Aage jab naya version aaye

1. Apni **package list** aur **first boot script** ek text file me save rakho (copy-paste ke liye).
2. Step 2 se dobara chalo. Version dropdown me naya stable version chuno, wahi packages aur script paste karo.
3. Step 1 (backup), Step 7 (hash), Step 8 (flash) dobara karo.

LuCI me **System → Attended Sysupgrade** ka option bhi hai jo upgrade ke dauran aapke packages saath le jata hai. Isse use karna chaho to pehle backup le lo, aur dhyan se padho ki wo kya karne ja raha hai.

---

## Dikkat aaye to (troubleshooting)

| Dikkat | Kya karo |
|---|---|
| Build par laal error: `Impossible package selection: missing (...)` | Bracket wala package hatao ya sahi naam rakho, phir dobara Request Build |
| Download ki file ka hash match nahi | File dobara download karo. Browser ya internet ki dikkat ho sakti hai |
| PowerShell do hash dikhata hai | `Desktop\flash` me sirf ek `.bin` rakho |
| LuCI me **Image check failed** | **Force mat karo.** Dekho ki sysupgrade file hai, aur is router (JioRouter AX6000) ki hai |
| Flash ke baad `192.168.1.1` nahi khulta | 5-6 minute ruko. LAN cable seedha laptop se jodo. Phir bhi nahi to neeche "Recovery" dekho |
| Flash ke baad WiFi ya password purana nahi | Backup restore karo: **System → Backup / Flash Firmware → Restore backup** me apni `.tar.gz` file do |
| Model `JIDU6J01` dikhta hai | Step 10 karo |
| `ssh` me **REMOTE HOST IDENTIFICATION HAS CHANGED** | `ssh-keygen -R 192.168.1.1` chalao |
| Tailscale ka login chala gaya | Tailscale guide ka Step 3 dobara karo |
| Packages (Services menu me) nahi dikhte | Step 3 me package list me sahi naam the? Naya build karo. Ya SSH me `apk update` ke baad `apk add <package>` try karo |

### Recovery (agar router bilkul na khule)
Is router ke bootloader ke menu me "Upgrade firmware" aur "Start Web failsafe" jaise options dikhe the (serial console par), par **maine recovery test nahi ki hai**. Router me boot hi na ho to aam taur par UART serial ya TFTP jaise advanced tareeke lagte hain. Aisa ho to apna **partition backup (Step 1B) sambhal ke rakho** aur OpenWrt forum par is device ka thread dekho. Isi liye Step 1 skip nahi karna.

---

## Surakshit rehne ke liye

- **Sirf official Firmware Selector** (`firmware-selector.openwrt.org`) ya `downloads.openwrt.org` se firmware lo. Kisi aur ka banaya hua build (GitHub, Telegram) bina uska code aur scripts dekhe **seedha flash mat karo.**
- **Hamesha SHA256 match karo.**
- **Force upgrade** kabhi mat karo.
- **First boot script me password, Tailscale key ya koi secret mat daalo.**
- Router ka **password set** rakho. Bina password ke router LAN par kisi ko bhi khula hai.
- **Config backup (`.tar.gz`)** aur **partition backup (`mtd*.bin`)** ko **private** rakho. Inhe GitHub, Telegram ya kisi public jagah par kabhi upload mat karo.
- **`MFG` aur `Reserved` partition ka output** kisi forum ya group me paste mat karo.
- Guide ya screenshots share karne se pehle unme se apna **MAC address, serial number, WiFi password, Tailscale IP aur login link** hata do.

---

## Wapas purane firmware par (rollback)

1. **System → Backup / Flash Firmware → Restore backup** me apni purani config `.tar.gz` file do.
2. Purana firmware chahiye to Firmware Selector me **version dropdown** se purana version chuno, wahi steps (Step 5-8) dohrao.
3. **Naye se purane version** par jaane me settings kabhi-kabhi match nahi karti. Agar dikkat aaye to **Keep settings ko untick** karke flash karo aur backup se settings dheere dheere dobara daalo.

---

## Aksar poochhe jane wale sawal

**Kya ye safe hai?** Official website se, hash match karke, aur backup lekar flash karna kaafi safe hai. Par router brick hone ka risk kabhi zero nahi hota.

**Factory ya Sysupgrade?** Chalte hue OpenWrt par **sysupgrade**. Factory nahi.

**Kya isse mera purana WiFi/password jayega?** "Keep settings" tick rakhne par settings bachti hain. Phir bhi backup zaroor lo.

**Kya ye sab 6J01/6201/6401/6601/6701 ke liye chalega?** Haan, Firmware Selector ke "About this build" me ye sab ek hi build me listed hain.

**Theme ya doosre features jo official feed me nahi hain?** Wo is guide ke dayre se bahar hain. Third-party packages ke liye alag se sochna padta hai, aur wo official signed packages jaisi suraksha nahi dete.

**Pehli baar stock Jio firmware se OpenWrt?** Is guide me nahi hai. Upar ka warning dekho.

---

## English summary

**What this is:** a step-by-step guide to build a custom OpenWrt image for the **JioRouter AX6000** (JIDU6J01/6201/6401/6601/6701) with the **official OpenWrt Firmware Selector**, verify its SHA256, and flash it with LuCI.

**Only for routers already running OpenWrt.** First-time install from the Jio stock firmware is not covered. A forum report says the stock bootloader is signed and can reject official images, and recovery may need UART or TFTP.

**Steps:** confirm OpenWrt is running, back up (config and, optionally, the `Factory`/`BL2`/`FIP`/`MFG` partitions), open `firmware-selector.openwrt.org`, pick the device and a **stable** version, append your packages (we used `tailscale luci-app-tailscale-community adblock-fast luci-app-adblock-fast`) without removing the defaults, paste the first-boot script (timezone and model detection from the `MFG` partition), request the build, download the **sysupgrade** image (not factory), match the SHA256, flash via LuCI with *Keep settings* on and *Force upgrade* off, then reboot and re-check.

**Security:** use only the official selector, always verify the hash, never force-flash, put no secrets in the first-boot script, keep backups and the `MFG`/`Reserved` partitions private.

**Tested:** JioRouter AX6000 (JIDU6401), OpenWrt 25.12.6, October 2026. Unofficial community guide, use at your own risk.

---

## Credits

OpenWrt aur uski Firmware Selector (ASU) team ko dhanyavaad. Ye guide ek real JioRouter AX6000 par kiye gaye build aur flash se bani hai.
