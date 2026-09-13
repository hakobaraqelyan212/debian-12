1. Host համակարգում ստեղծիր նոր նախագիծ
   mkdir -p ~/debian12-installer-project
   cd ~/debian12-installer-project

Եթե նույն անունով հին container կա՝

docker rm -f debian12-iso-builder 2>/dev/null || true
2. Ստեղծիր persistent Debian 12 container

Կատարիր host համակարգում՝

docker run -it
  --privileged
  --name debian12-iso-builder
  -e DEBIAN_FRONTEND=noninteractive
  -v "$HOME/debian12-installer-project:/build"
  debian:12 bash

Կարևոր է՝ այստեղ --rm չկա։

Այսինքն երբ container-ից դուրս գաս՝

exit

այն կկանգնի, բայց չի ջնջվի։

Հետագայում նույն container-ը նույն installed ծրագրերով բացելու համար՝

docker start -ai debian12-iso-builder
3. Container-ի ներսում տեղադրիր Debian Installer build գործիքները

Prompt-ը կլինի մոտավորապես՝

root@xxxxxxxx:/#

Կատարիր՝

apt update

հետո՝

apt install -y
  simple-cdd
  debian-cd
  reprepro
  xorriso
  isolinux
  syslinux-common
  mtools
  dosfstools
  wget
  ca-certificates
  gnupg

Ստուգիր՝

build-simple-cdd --help | head

Պետք է տեսնես build-simple-cdd-ի help-ը։

simple-cdd-ն կարող է ստեղծել basic Debian Installer CD նույնիսկ առանց custom profile-ի։

4. Անցիր նախագծի թղթապանակ
   cd /build

Ստուգիր՝

pwd

պետք է լինի՝

/build

Այս պահին ոչ մի profile, preseed կամ package list չենք ստեղծում։

5. Build արա մաքուր Debian 12 installer ISO

Քանի որ container-ում root ենք, տալիս ենք --force-root։

build-simple-cdd
  --force-root
  --dist bookworm
  --debian-mirror https://deb.debian.org/debian
  --security-mirror https://security.debian.org/debian-security
  2>&1 | tee build-stage1.log

--dist bookworm-ը պարտադիր ենք բացահայտ նշում, որպեսզի հաստատ Debian 12 կառուցվի։ build-simple-cdd-ի --dist, --debian-mirror և --security-mirror պարամետրերը հենց դրա համար են։

Առաջին build-ը կարող է բավական ժամանակ տևել և շատ package ներբեռնել։

6. Build-ից հետո գտիր ISO-ն
   find /build/images
   -maxdepth 1
   -type f
   -name '*.iso'
   -ls

Կամ՝

ls -lh /build/images/

ISO-ի կոնկրետ անունը կարող է տարբեր լինել, դրա համար հիմա անունը չենք hardcode անում։

Օրինակ կարող է ստացվել նման ֆայլ՝

/build/images/debian-12.x.x-amd64-CD-1.iso

simple-cdd-ի basic build-ի ISO-ն հենց images/ թղթապանակում է ստեղծվում։

7. Host համակարգից ISO-ն նույնպես հասանելի կլինի

Container-ից դուրս եկ՝

exit

Host-ում՝

cd ~/debian12-installer-project
ls -lh images/

Այս ISO-ն արդեն կարող ես գրել USB-ի վրա կամ փորձարկել VirtualBox-ում։

Սա Live ISO չէ։ Boot-ից հետո պետք է հայտնվի Debian Installer-ը և կարողանաս սովորական հերթականությամբ անցնել՝

Install / Graphical install
        ↓
Language
        ↓
Keyboard
        ↓
Network
        ↓
Users
        ↓
Partition disks
        ↓
Install base system
        ↓
Software selection
        ↓
GRUB
        ↓
Reboot
        ↓
HDD/SSD-ից Debian

Եթե ինչ-որ տեղ տեսնես ավտոմատ debian-live login, ուրեմն սխալ ISO ես boot արել։ Այս նախագծում live-build ընդհանրապես չենք օգտագործում։

Փուլ 1-ի նպատակը

Այս պահին ոչինչ ավտոմատ չենք փոխում։ Installer-ը պետք է լինի հնարավորինս Debian-ի default installer-ը։ Տեղադրման ժամանակ թող ինքը ցույց տա իր սովորական Software selection էջը և իր default տարբերակները։

Երբ այս ISO-ն հաջող build անես և փորձարկես, ուղարկիր build-stage1.log-ի վերջին մասը՝

tail -n 50 ~/debian12-installer-project/build-stage1.log

կամ ուղղակի ասա, որ installation-ը հասել է մինչև վերջ։ Հետո Փուլ 2-ում միայն քո ասած մեկ փոփոխությունը կանենք, առանց որևէ ուրիշ բան ավելացնելու։

1. Ստեղծիր local default.downloads

Container-ի ներսում՝

cd /build

mkdir -p profiles

Շատ կարևոր է՝ դատարկ default.downloads չստեղծենք։ Սկզբում պատճենում ենք Simple-CDD-ի իր default ցանկը՝

cp
  /usr/share/simple-cdd/profiles/default.downloads
  profiles/default.downloads

Հետո ավելացնում ենք միայն zstd՝

grep -qxF 'zstd' profiles/default.downloads
  || echo 'zstd' >> profiles/default.downloads

Ստուգիր՝

tail -n 10 profiles/default.downloads

և՝

grep -n '^zstd$' profiles/default.downloads

Պետք է ցույց տա՝

zstd

*.downloads ֆայլը նախատեսված է այն փաթեթների համար, որոնք պետք է հայտնվեն installation media-ի վրա, բայց պարտադիր չեն install արվի որպես custom profile package։

2. Ինչու պատճենեցինք default ֆայլը

default profile-ը Simple-CDD-ում հատուկ profile է և միշտ օգտագործվում է։ Local ./profiles/default.downloads ֆայլը գերակայում է /usr/share/simple-cdd/profiles/default.downloads-ին։ Դրա համար եթե ուղղակի ստեղծեինք միայն՝

zstd

կարող էինք կորցնել default ցանկի մյուս անհրաժեշտ փաթեթները։

3. Հին build output-ը մաքրիր

Քո project ֆայլերը չենք ջնջում։ Միայն նախորդ generated ISO/mirror-ը՝

cd /build

rm -rf tmp images

Ստուգիր՝

ls -la

Պետք է մոտավորապես մնա՝

profiles/
build-stage1.log
4. Նորից build արա նույն հրամանով
build-simple-cdd
  --force-root
  --dist bookworm
  --debian-mirror https://deb.debian.org/debian
  --security-mirror https://security.debian.org/debian-security
  2>&1 | tee build-stage1-zstd.log

Մենք դեռ ոչ մի customization չենք անում։ Սա շարունակում է մնալ Phase 1-ի default Debian 12 installer-ը։ zstd-ը պարզապես Bookworm Simple-CDD bug-ի workaround-ն է։

5. Build-ից հետո ստուգիր ISO-ն
   ls -lh /build/images/

Պետք է նորից ունենաս՝

debian-12-amd64-CD-1.iso

Հետո ստուգենք, որ այս անգամ zstd իսկապես media-ի մեջ է։

zgrep -i 'zstd'
  /build/images/debian-12-amd64-CD-1.list.gz

Նաև՝

find /build/tmp
  -type f
  -iname 'zstd*.deb'
  -print

Առնվազն մեկը պետք է zstd գտնի։

6. ISO-ի timestamp-ը ստուգիր

Որպեսզի հաստատ հին ISO-ն USB-ի վրա չգրես՝

stat /build/images/debian-12-amd64-CD-1.iso

Creation/Modification ժամանակը պետք է լինի նոր build-ի ժամանակը։

Եթե ուզում ես checksum էլ պահել՝

sha256sum
  /build/images/debian-12-amd64-CD-1.iso
  | tee /build/images/debian-12-amd64-CD-1.iso.sha256
7. Log-ի ճանապարհի մասին

Դու container-ի ներսում գրել էիր՝

tail -n 100 ~/debian12-installer-project/build-stage1.log

բայց container-ի ներսում ~ նշանակում է՝

/root

իսկ project-ը mount արել ենք /build։

Ուստի ճիշտ հրամանն է՝

tail -n 100 /build/build-stage1-zstd.log

կամ, եթե արդեն /build-ում ես՝

tail -n 100 build-stage1-zstd.log

Այս նոր ISO-ն փորձիր նորից։ Եթե Install the base system փուլը այս անգամ անցնի, Փուլ 1-ը կհամարենք հաջող ավարտված և հետո միայն քո ասած փոփոխությամբ կանցնենք Փուլ 2-ին։
