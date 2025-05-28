# BIOS (GMKtec EVO-X1 64GB)
- iGPUのRAM容量を増やした。UMA Frame buffer size -> 32G</br>
https://chimolog.co/gmktec-evo-x1/

# WSL
1. WSLと仮想マシンプラットフォームの有効化</br>
 https://qiita.com/mizutoki79/items/c8fcb26a03957805b9b3
</br>管理者権限のPowerShellで下記を実行する
</br>`Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux`
</br>`Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform`
   
1. `wsl.exe --install`
1. `wsl.exe --update`
1. `wsl.exe --set-default-version 2`
1. Linux distributionを入れる</br>
`wsl.exe --list --online`</br>
`wsl.exe --install Ubuntu-24.04`</br>
※wslインストール時に入ってたUbuntuを削除する：`wsl.exe --unregister Ubuntu`

# Git
- Git for Windows: https://git-scm.com/downloads/win
- Github Desktop: https://desktop.github.com/download/

# VSCode
- VSCode: https://code.visualstudio.com/docs/?dv=win64user