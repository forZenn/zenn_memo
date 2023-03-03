---
title: "hyper-v + vagrant環境構築"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["powershell", "windows", "vagrant", "hyper-v"]
published: true
published_at: 2023-02-25 19:00 
---

virtualboxよりもhyper-vの方が断然早いので、hyper-vを使いたい。
virtualboxはwindowsやらmacやらlinuxやらいろんなOSが
ある環境で無い限りそんなメリット無いしね。

体系的にまとまっているサイトが無かったので書く。
hyper-vはwindows pro以上のエディションじゃないと使えないからそれは注意ね。

## hypber-vのインストール

管理者権限でpowershellを立ち上げて、下記のコマンドを実行してください。
Hyper-vがインストールされるので、再起動すると使えるようになります。

```powershell
Enable-windowsOptionalFeature -online -FeatureName Microsoft-Hyper-V-Hypervisor -All
```

## Hyper-V Administratorsグループに入れる

hyper-vを使うためにhypber-v Administratorグループに入れる必要があります。
少々お行儀が悪いですが、最初にwindowsをインストールしたユーザーだったりしてAdministratorグループにすでに入っているなら常に管理者権限で動かす事もできます。

### グループの確認

```powershell
# Hypber-V Administratorグループに入っているユーザーリスト取得
Get-LocalGroupMember "Hyper-V Administrators"

# Administratorsグループに入っているユーザーリスト取得
Get-LocalGroupMember Administrators
```

### グループに入れる

```powershell
$user = "追加したいユーザー名"
# Hypber-V Administratorグループに追加
Add-LocalGroupMember -Group "Hyper-V Administrars" -Member $user
```

## vagrantのインストール

最近はwingetで入る。

```powershell
winget install Hashicorp.Vagrant
```

## Vagrant file

hyper-vで動かすための最低限の設定は下のようになります。

```ruby:Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
    # The most common configuration options are documented and commented below.
    # For a complete reference, please see the online documentation at
    # https://docs.vagrantup.com.
  
    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://vagrantcloud.com/search.
    config.vm.box = "base"

    # dont set ssh key, if you create vagrant box from this container.
    # this config is being used, firtst time you create and launch container.
    # config.ssh.insert_key = false

    # Disable automatic box update checking. If you disable this, then
    # boxes will only be checked for updates when the user runs
    # `vagrant box outdated`. This is not recommended.
    # config.vm.box_check_update = false
  
    # Create a forwarded port mapping which allows access to a specific port
    # within the machine from a port on the host machine. In the example below,
    # accessing "localhost:8080" will access port 80 on the guest machine.
    # NOTE: This will enable public access to the opened port
    # config.vm.network "forwarded_port", guest: 3000, host: 3000
    # config.vm.network "forwarded_port", guest: 1521, host: 1521
  
    # Create a forwarded port mapping which allows access to a specific port
    # within the machine from a port on the host machine and only allow access
    # via 127.0.0.1 to disable public access
    # config.vm.network "forwarded_port", guest: 3000, host: 3000, host_ip: "127.0.0.1"
  
    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    # config.vm.network "private_network", ip: "192.168.33.10"
  
    # Create a public network, which generally matched to bridged network.
    # Bridged networks make the machine appear as another physical device on
    # your network.
    # config.vm.network "public_network"
  
    # Share an additional folder to the guest VM. The first argument is
    # the path on the host to the actual folder. The second argument is
    # the path on the guest to mount the folder. And the optional third
    # argument is a set of non-required options.

    # Share an additional folder to the guest VM. The first argument is
    # the path on the host to the actual folder. The second argument is
    # the path on the guest to mount the folder. And the optional third
    # argument is a set of non-required options.
    # config.vm.synced_folder "../data", "/vagrant_data"

    # Provider-specific configuration so you can fine-tune various
    # backing providers for Vagrant. These expose provider-specific options.
    # Example for VirtualBox:
    #
    config.vm.provider "virtualbox" do |vb|
      # Display the VirtualBox GUI when booting the machine
      vb.gui = true

      # Customize the amount of memory on the VM:
        vb.memory = "8192"
    end

    # 上のvirtual boxと同じ設定をするなら下のようになる
    config.vm.provider "hyperv" do |hv|
      # hyper-vでゲストーホスト間でのファイル共有が必要な場合は必須。
      config.vm.synced_folder ".", "/vagrant", type: "smb"
      hv.maxmemory = "8192"
    end
    #
    # View the documentation for the provider you are using for more
    # information on available options.
  
    # Enable provisioning with a shell script. Additional provisioners such as
    # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
    # documentation for more information about their specific syntax and use.
    # config.vm.provision "shell", inline: <<-SHELL
    #   apt-get update
    #   apt-get install -y apache2
    # SHELL
end
```

注意点としては
1. hyper-vにvirtualboxのvb.guiに相当するものはない -> 画面が必要ならvagrant up 後にhyper-v Managerから画面を立ち上げる必要がある。つまりウィンドウの立ち上げのみ手動。
2. virtual boxの場合はvagrantと自動的にファイル共有がなされるが、hyper-vはsambaの設定を書いたうえで、vagrant up時にwindowsのユーザー名とパスワードを入力する必要がある(環境変数に入れてもよい。)。

