Raspberry Pi4を使ったGPIOメモ  

# libgpiod toolでのGPIO操作
- gpioinfo
    ```bash
    gpioinfo
        gpiochip0 - 58 lines:
            line   0:     "ID_SDA"       unused   input  active-high
            line   1:     "ID_SCL"       unused   input  active-high
            line   2:      "GPIO2"       unused   input  active-high
            line   3:      "GPIO3"       unused   input  active-high
            line   4:      "GPIO4"       unused   input  active-high
            line   5:      "GPIO5"       unused   input  active-high
            line   6:      "GPIO6"       unused   input  active-high
            line   7:      "GPIO7"       unused   input  active-high
            line   8:      "GPIO8"       unused   input  active-high
            line   9:      "GPIO9"       unused   input  active-high
            line  10:     "GPIO10"       unused   input  active-high
            line  11:     "GPIO11"       unused   input  active-high
            line  12:     "GPIO12"       unused   input  active-high
            line  13:     "GPIO13"       unused   input  active-high
            line  14:     "GPIO14"       unused   input  active-high
            line  15:     "GPIO15"       unused   input  active-high
            line  16:     "GPIO16"       unused   input  active-high
            line  17:     "GPIO17"       unused   input  active-high
            line  18:     "GPIO18"       unused   input  active-high
            line  19:     "GPIO19"       unused   input  active-high
            line  20:     "GPIO20"       unused   input  active-high
            line  21:     "GPIO21"       unused   input  active-high
            line  22:     "GPIO22"       unused   input  active-high
            line  23:     "GPIO23"       unused   input  active-high
            line  24:     "GPIO24"       unused   input  active-high
            line  25:     "GPIO25"       unused   input  active-high
            line  26:     "GPIO26"       unused   input  active-high
            line  27:     "GPIO27"       unused   input  active-high
            line  28: "RGMII_MDIO"       unused   input  active-high
            line  29:  "RGMIO_MDC"       unused   input  active-high
            line  30:       "CTS0"       unused   input  active-high
            line  31:       "RTS0"       unused   input  active-high
            line  32:       "TXD0"       unused   input  active-high
            line  33:       "RXD0"       unused   input  active-high
            line  34:    "SD1_CLK"       unused   input  active-high
            line  35:    "SD1_CMD"       unused   input  active-high
            line  36:  "SD1_DATA0"       unused   input  active-high
            line  37:  "SD1_DATA1"       unused   input  active-high
            line  38:  "SD1_DATA2"       unused   input  active-high
            line  39:  "SD1_DATA3"       unused   input  active-high
            line  40:  "PWM0_MISO"       unused   input  active-high
            line  41:  "PWM1_MOSI"       unused   input  active-high
            line  42: "STATUS_LED_G_CLK" "ACT" output active-high [used]
            line  43: "SPIFLASH_CE_N" unused input active-high
            line  44:       "SDA0"       unused   input  active-high
            line  45:       "SCL0"       unused   input  active-high
            line  46: "RGMII_RXCLK" unused input active-high
            line  47: "RGMII_RXCTL" unused input active-high
            line  48: "RGMII_RXD0"       unused   input  active-high
            line  49: "RGMII_RXD1"       unused   input  active-high
            line  50: "RGMII_RXD2"       unused   input  active-high
            line  51: "RGMII_RXD3"       unused   input  active-high
            line  52: "RGMII_TXCLK" unused input active-high
            line  53: "RGMII_TXCTL" unused input active-high
            line  54: "RGMII_TXD0"       unused   input  active-high
            line  55: "RGMII_TXD1"       unused   input  active-high
            line  56: "RGMII_TXD2"       unused   input  active-high
            line  57: "RGMII_TXD3"       unused   input  active-high
        gpiochip1 - 8 lines:
            line   0:      "BT_ON"       unused  output  active-high
            line   1:      "WL_ON"       unused  output  active-high
            line   2: "PWR_LED_OFF" "PWR" output active-low [used]
            line   3: "GLOBAL_RESET" unused output active-high
            line   4: "VDD_SD_IO_SEL" "vdd-sd-io" output active-high [used]
            line   5:   "CAM_GPIO" "regulator-cam1" output active-high [used]
            line   6:  "SD_PWR_ON" "regulator-sd-vcc" output active-high [used]
            line   7:    "SD_OC_N"       unused   input  active-high
    ```
- gpiomon で、GPIO18のINPUTをモニタする。  
    Pull upしてるのでまずはRising edge割り込みが入っている？
    その後、ボタン押下＝GNDになってFalling edge割り込みが、ボタンリリースでRising edge割り込みが入った。
    ```bash
    gpiomon gpiochip0 18
        event:  RISING EDGE offset: 18 timestamp: [      36.473537607]
        [   36.475417] GPIO 18: IRQ type set to edge-both
        [   36.475428] GPIO 18: IRQ unmasked
        [   36.475433] Handling GPIO bank 0 IRQs, mask 0xfffffff
        [  103.813963] Handling GPIO bank 0 IRQs, mask 0xfffffff
        event: FALLING EDGE offset: 18 timestamp: [     103.812081790]
        [  108.076595] Handling GPIO bank 0 IRQs, mask 0xfffffff
        event:  RISING EDGE offset: 18 timestamp: [     108.074707639]
    ```
    Ctrl-Zでgpiomonをバックグラウンドにして、gpioinfoしてみる。  
    　-> gpiomonにより使われていることがわかる。  
    ※fgでgpiomonに戻り、Ctrl-Cでgpiomonを終了すると、unusedに戻る。
    ```bash
    gpioinfo
        (GPIO18のみ抜粋)
        line  18:     "GPIO18"    "gpiomon"   input  active-high [used]
    ```
- gpioset で、GPIO23のOUTPUT出力をする。
    ```bash
    gpioset gpiochip0 23=1　# Active high出力する(LEDが点灯した)
    ```
    この状態(gpiosetは終了している状態)で、gpioinfoしてみる。  
    　-> unusedにはなっているが、outputのまま(inputに戻らない)。LEDも点灯したまま。
    ```bash
    gpioinfo
        (GPIO23のみ抜粋)
        line  23:     "GPIO23"       unused  output  active-high
    ```

- UARTで使っているGPIOについて  
    GPIO14：TXD0、GPIO15：RXD0の２ピンを、UART(シリアルコンソール)で使っている。  
    しかし、gpioinfoでは、unusedになっている。
    ```bash
    gpioinfo
        (GPIO14,15のみ抜粋)
        line  14:     "GPIO14"       unused   input  active-high
        line  15:     "GPIO15"       unused   input  active-high
    ```
    sshログインして、その端末から、gpiomonでGPIO15をモニタしてみる。  
    UARTのシリアルコンソールの方は、コンソール出力はされるが、キー入力しても入力されなくなった。  
    キー入力のたびに、下記のようなログが出力され、GPIO INPUTの割り込みが発生していることがわかる。（pinctrl-bcm2835.cに追加したログ出力）
    ```bash
    (gpiomon開始直後。割り込みが有効にされ、初回(初期のピン入力状態)の割り込みが入ったログ。)
    [   64.551236] GPIO 15: IRQ type set to edge-both
    [   64.551254] GPIO 15: IRQ unmasked
    [   64.551264] Handling GPIO bank 0 IRQs, mask 0xfffffff
    (キー入力すると下記が出力された。)
    [   75.495265] Handling GPIO bank 0 IRQs, mask 0xfffffff
    [   75.495302] Handling GPIO bank 0 IRQs, mask 0xfffffff
    [   75.495337] Handling GPIO bank 0 IRQs, mask 0xfffffff
    ```
    sshの方の端末は、gpiomonとしてのログが出力された。
    ```bash
    gpiomon gpiochip0 15
        event:  RISING EDGE offset: 15 timestamp: [      64.549867572]
        event: FALLING EDGE offset: 15 timestamp: [      75.493873769]
        event: FALLING EDGE offset: 15 timestamp: [      75.493912436]
        event:  RISING EDGE offset: 15 timestamp: [      75.493933992]
    ```

    UARTで使っているのに、pinctrlの管理状態としては未使用扱いで、別用途で使えてしまった。


# libgpiodを使ったアプリケーション
## ビルド方法
1. buildrootで、`BR2_PACKAGE_HOST_ENVIRONMENT_SETUP`を有効にしてmakeしておく。
1. buildrootが生成した環境変数用スクリプトを読み込む。
   ```bash
   source path/to/buildroot/output/host/environment-setup
   ```
1. ビルドする。
   ```bash
   ${CROSS_COMPILE}gcc hoge.c -o hoge.out $(${PKG_CONFIG} --cflags libgpiod) $(${PKG_CONFIG} --libs libgpiod)
   ```
