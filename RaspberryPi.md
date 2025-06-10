# Raspberry Pi PicoをDebug probeとして使う
ターゲットはRaspberry Pi 4。PCはWindows11。</br>
WSLのUbuntuからDebug probe (Pico)を使ってRpi4をデバッグする。

## UART
1. Picoに、`debugprobe_on_pico.uf2`を入れる。</br>
https://github.com/raspberrypi/debugprobe/releases
1. UARTを配線する。
    - PicoのPin7(UART1 RX, GP5) ～ Rpi4のPin8(UART TXD0, GPIO14)
    - PicoのPin6(UART1 TX, GP4) ～ Rpi4のPin10(UART RXD0, GPIO15)
    - お互いのGND結線

1. Raspberry Pi 4で、起動時からUARTを有効にする。
    - Raspberry Pi Imagerで、Rasberry Pi OSをSDカードに書き込む。</br>
      Raspberry Pi OS (other) -> Raspberry Pi OS Lite (64-bit)で試した。
        - 注：ユーザー・パスワードをカスタムしておいた方がよさそう。</br>
        　　HDMI接続モニタで初期ユーザーの入力を求められたが、UARTのターミナルでは対処不可なため。
    - SDカードのFATボリューム内の`config.txt`に`enable_uart=1`を追加する。</br>
      とりあえず`[all]`のところに追加した。

1. WindowsのUSBをWSL(Ubuntu)から使えるようにする。</br>
https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb
https://qiita.com/motoJinC25/items/e332d731111c2a29ca25
    - https://github.com/dorssel/usbipd-win/releases から`~.msi`をダウンロードしてインストールする
    - 管理者権限のPowerShellで、以下を実行する。
      ```bash
      usbipd list                   # Debug probeを意味する「CMSIS-DAP～」のBUSIDを確認する。
      usbipd bind --busid x-xx      # 上記で確認した`BUSID`(x-xx)を共有設定する。
      ```
    再度`usbipd list`すると、Debug probeの`STATE`が`Shared`になっている。
    - WSLにDebug probeのUSBを接続する。
      ```bash
      usbipd attach --wsl --busid x-xx
      ```
      注：事前にWSLを起動しておくこと。

1. WSL(Ubuntu)で、UARTを使えるようにする。
    - minicomをインストールする。
      ```bash
      sudo apt install minicom
      ```
    - root権限なしで使えるようにする。
      ```bash
      sudo usermod -a -G dialout $USER
      ```
      いったんログアウト（exit）して再度WSL(Ubuntu)にログインする。
    - minicomでDebug probeのUSBを開く。
      ```bash
      minicom -D /dev/ttyACMx
      ```


ここまでで、PicoのDebug probeを使って、UARTでRaspberry Pi 4にログインできた。

## SWD → 撤退
1. SWDを配線する。</br>
Raspberry Pi 4 / BCM2711で、SWDデバッグできなかったので、配線の記載を削除した。

1. OpenOCD
    - WSL(Ubuntu)でOpenOCDをインストールする。
      ```bash
      sudo apt install openocd
      ```
    - OpenOCDをroot権限なしで使えるようにする。</br>
      - `lsusb -v`で、PicvoのDebug probeの`idVendor`("0x2e8a")と`idProduct`(4桁の16進数。たぶん変わる？)を確認しておく。
      - `/etc/udev/rules.d/99-picoprobe.rules`ファイルを作成し、下記を記述する。</br>
        ```text
        # Picoprobe
        ATTRS{idVendor}=="2e8a", ATTRS{idProduct}=="000d", MODE="0666"
        ```
      - udev ruleをリロードする。
        ```bash
        sudo udevadm control --reload-rules
        sudo udevadm trigger
        ```
    - OpenOCDを実行する。
      ```bash
      openocd -f interface/cmsis-dap.cfg -f target/bcm2711.cfg
      ```
      SWDでやろうとしたけど、configはJTAGを要求してる、みたいなエラーが出た。</br>
       → BCM2711が、SWDに対応してなさそう（JTAGは対応してる）なので、撤退する。</br>
       　https://forums.raspberrypi.com/viewtopic.php?t=376114
       </br>※下記が使えるのかも、だが、、、UARTシリアルデバッグで進める。</br>
       　https://github.com/phdussud/pico-dirtyJtag
       </br>※Raspberry Pi 5なら、SWDデバッグできる。ただしUARTとは排他。</br>
       　https://www.raspi.jp/2024/03/raspberry-pi-5-hardware-debbuging/
       </br>※AIの回答。
       </br>　GeminiはBCM2711/Rpi4で、SWDデバッグできると頑固に主張した。根拠として示されたリンクにSWDの記載がないにも関わらず。
       </br>　MS Copilotは、BCM2711/Rpi4でSWDデバッグできる情報は無いとの回答だった。もし情報が見つかったら共有してほしいとのこと。


# Raspberry Pi 4
## Fan電源
https://miyabee-craft.com/blog/2024/09/15/post-9748/
</br>USB-Aからとる


## 開発ツールの準備
```bash
sudo apt install build-essential git flex bison libssl-dev device-tree-compiler
sudo apt install gcc-aarch64-linux-gnu
sudo apt install crossbuild-essential-arm64
```

## uboot
https://docs.u-boot.org/en/latest/board/broadcom/raspberrypi.html
1. ソースコードを取得する。
   ```bash
   git clone git://git.denx.de/u-boot.git
   ```
1. コンフィグレーションする。
   ```bash
   make rpi_4_defconfig
   ```
   ※`configs/rpi_4_defconfig`で`.config`ファイルを作る。
1. u-boot.bin等をビルドする。
   ```bash
   export CROSS_COMPILE=aarch64-linux-gnu-
   make -j$(nproc)
   ```
1. SDカードに書き込む。
    - u-boot.binを書き込む
    - config.txtを編集する。以下の行を追加（or 編集）する。
      ```
      kernel=u-boot.bin
      ```
1. SDカードをRaspberry Pi 4に挿入して電源ONし、ubootに入る。
   - PicoProbeのDebug UARTのCOMポートをTeraTerm等で開いておく。115200kbps.
   - Raspberry Pi 4の電源をONしたら、TeraTermでEnterキー等を連打し続ける。</br>
   　→ ubootのコマンドが入力できるようになる。

## Linux kernel
https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compile-the-kernel
1. ソースコードを取得する。
   ```bash
   git clone --depth=1 https://github.com/raspberrypi/linux
   ```
1. コンフィグレーションする。
   ```bash
   cd linux
   KERNEL=kernel8
   make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
   ```
1. ビルドする。
   ```bash
   make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
   ```


## 起動シーケンスのメモ
[Qiita Raspberry PiでSDカード暗号化とセキュアブートをテスト](https://qiita.com/kmitsu76/items/933443ea5def3d73deb2)  
[Rpi4 secure boot - chain of trust](https://github.com/raspberrypi/usbboot/blob/master/docs/secure-boot-chain-of-trust-2711.pdf)

https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-boot-eeprom

## Raspberry Pi documentation
下記をWSL Ubuntu 24.04で実施した。
1. githubから文書ソースを取得してVSCodeで開く。
   ```bash
   git clone --depth=1 https://github.com/raspberrypi/documentation
   cd documentation
   code .
   ```
1. `CONTRIBUTING.md`の`Build` -> `Linux`に従い、ツールをセットアップする。
1. `CONTRIBUTING.md`の`Set up environment`に従い、Python venv環境をセットアップする。
1. `CONTRIBUTING.md`の`Build HTML`で、HTML作成、および、Webサーバを起動する。

Webブラウザで開いたところ、下記と同様のコンテンツな様子。。。  
https://www.raspberrypi.com/documentation/


# Qemu/ARM64
## buildroot
1. buildrootをダウンロードし、適当なディレクトリに展開する。
   ```bash
   wget https://buildroot.org/downloads/buildroot-2025.02.tar.gz
   tar xvzf buildroot-2025.02.tar.gz
   ```
1. Configurationする。
   ```bash
   make qemu_aarch64_virt_defconfig
   make menuconfig
   ```
   System configuration
    -> Enable root login with passwordが[*]であることを確認する。
    -> Root passwordを入れておく

1. Buildする。
   ```bash
   make BR2_JLEVEL=$(nproc)
   ```
   - WSLでWindows PATHを引き継いだため、下記エラーが発生した。Program Filesのスペース等が原因。
     > Your PATH contains spaces, TABs, and/or newline (\n) characters.
This doesn't work. Fix you PATH.

     WSLにて`/etc/wsl.conf`を編集し、下記を追加する。WSLを再起動する。再度`make BR2_JLEVEL=$(nproc)`を実行する。
     ```
     [interop]
     appendWindowsPath = false
     ```
1. 実行する。
   ```bash
   ./output/images/start-qemu.sh
   ```
   qemu (Embeded linux)の終了するには、`poweroff`コマンドを実行する。