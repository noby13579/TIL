# Raspberry Pi PicoをDebug probeとして使う
ターゲットはRaspberry Pi 4。PCはWindows11。</br>
WSLのUbuntuからDebug probe (Pico)を使ってRpi4をデバッグする。


https://logikara.blog/pico_debug_arduino/
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
    - 管理者権限のPowerShellで、以下を実行する。</br>
    `usbipd list` ← Debug probeを意味する「CMSIS-DAP～」のBUSIDを確認する。</br>
    `usbipd bind --busid x-xx` ← 上記で確認した`BUSID`(x-xx)を共有設定する。</br>
    再度`usbipd list`すると、Debug probeの`STATE`が`Shared`になっている。
    - WSLにDebug probeのUSBを接続する。</br>
    `usbipd attach --wsl --busid x-xx`</br>
    注：事前にWSLを起動しておくこと。

1. WSL(Ubuntu)で、UARTを使えるようにする。
    - minicomをインストールする。</br>
    `sudo apt install minicom`
    - root権限なしで使えるようにする。</br>
    `sudo usermod -a -G dialout $USER`</br>
    いったんログアウト（exit）して再度WSL(Ubuntu)にログインする。
    - minicomでDebug probeのUSBを開く。
    `minicom -D /dev/ttyACMx`


ここまでで、PicoのDebug probeを使って、UARTでRaspberry Pi 4にログインできた。

## SWD
1. SWDを配線する。
    - PicoのPin5(SWCLK, GP3) ～ Rpi4のPin15(SWCLK, GPIO22)
    - PicoのPin4(SWDIO, GP2) ～ Rpi4のPin22(SWDIO, GPIO25)
    - お互いのGND結線（UARTで配線済）

1. OpenOCD
    - WSL(Ubuntu)でOpenOCDをインストールする。</br>
    `sudo apt install openocd`
    - OpenOCDをroot権限なしで使えるようにする。</br>
      - `lsusb -v`で、PicvoのDebug probeの`idVendor`("0x2e8a")と`idProduct`(4桁の16進数。たぶん変わる？)を確認しておく。
      - `/etc/udev/rules.d/99-picoprobe.rules`ファイルを作成し、下記を記述する。</br>
      `# Picoprobe`</br>
      `ATTRS{idVendor}=="2e8a", ATTRS{idProduct}=="000d", MODE="0666"`
      - udev ruleをリロードする。</br>
      `sudo udevadm control --reload-rules`</br>
      `sudo udevadm trigger`
    - OpenOCDを実行する。</br>
      `openocd -f interface/cmsis-dap.cfg -f target/bcm2711.cfg`
      - SWDでやろうとしたけど、configはJTAGを要求してる、みたいなエラーが出た。


# Raspberry Pi 4
## Fan電源
https://miyabee-craft.com/blog/2024/09/15/post-9748/
</br>USB-Aからとる