# 新设备配置git

## windows新设备配置git

1. 下载与初始化

    ```powershell
    winget install --id Git.Git -e --source winget
    git config --global user.name "abai"
    git config --global user.email "shuchangshang@gmail.com"
    git config --global core.autocrlf true  
    ```

2. 配置ssh

    windows可选功能经常卡死，可以github上下载msi
    <https://github.com/PowerShell/Win32-OpenSSH/releases>

    ```powershell
    Start-Service ssh-agent
    ssh-add $HOME\.ssh\id_ed25519
    Get-Content $HOME\.ssh\id_ed25519.pub
    ```

    github上配置密钥

    ```powershell
    ssh -T git@github.com 
    ```

    之后可以正常克隆

## macos新设备配置git
