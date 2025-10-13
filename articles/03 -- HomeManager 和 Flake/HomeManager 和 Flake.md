# HomeManager 和 Flake

[^tag]: nix



**nixos安装**见：[[NixOS 安装]]



![image](https://pic.sl.al/gdrive/pic/2025-08-31/hash_96aa4ca3_a402dc18111f_Screenshot_2025-07-16_15-41-28.png)

# Nix package manager



## 1. nix search



从 nixpkgs 查找软件包



```shell
nix search nixpkgs <package> <keywords> [-option]
```



`-e`: --exclude，排除

*eg*: `nix sesarch nixpkgs helm kubernetes -e plugin`



>   [!note]
>
>   archlinux 上的准备工作：
>
>   ```shell
>   sh <(curl --proto '=https' --tlsv1.2 -L https://nixos.org/nix/install) --daemon
>   sudo mkdir -p /etc/nix
>   echo "experimental-features = nix-command flakes" | sudo tee -a /etc/nix/nix.conf
>   sudo systemctl restart nix-daemon
>   ```
>
>   原因是 `nix-command` 实验性功能默认关闭



## 2. nix run



仅运行而不安装某个软件包（单次使用）



```
nix run nixpkgs#<package> -- [-option]
```



`[-option]`: 为运行的该软件包传递参数

*eg*: `nix run nixpkgs#cowsay -- --help`



## 3. nix shell



新建一个包含指定软件包环境的 shell，退出该 shell 后失效（多次使用）



```
nix shell nixpkgs#<package>
```



*eg*: `nix shell nixpkgs#cowsay`



## 4. nix profile



在当前 profile 安装指定软件包。不推荐，尽量在配置文件中声明



```
nix profile install nixpkgs#<packages>
```



*eg*: `nix profile install nixpkgs#cowsay`



列出通过profile安装的软件包



```
nix profile list
0 flake:nixpkgs#legacyPackages.x86_64-linux.<packages> github:...
1 ...
2 ...
```



移除软件包



```
nix profile remove <index>
```



## 5. nix develop



为不同的项目生成开发环境

>   *首先应进入到项目目录并创建 `flake.nix`*



```
nix develop
```





# Home Manager



在其他发行版上安装Nix包管理仅能将home-manager作为**独立软件包**安装！

在NixOS上既可将home-manager作为**模块**安装又可以作为**独立软件包**安装。



>   应将全局使用的软件包以及配置项写入 `configuration.nix`，其他所有配置写入 `home.nix`
>
>   >   提权使用homemanager管理的软件包：`sudo <command> [-option]`



## 1. NixOS



### 1.1 module



优点：集中管理， 所有配置都在 `configuration.nix` 中，系统和用户配置一致性



适用范围：



*   单用户NixOS并且希望集中管理
*   系统和用户配置完全同步



添加 home-manager 通道



```
sudo nix-channel --add https://github.com/nix-community/home-manager/archive/master.tar.gz home-manager
or nix-channel --add https://github.com/nix-community/home-manager/archive/release-25.05.tar.gz home-manager
sudo nix-channel --update
```



在 configuration.nix 中编辑以下内容



```nix
imports = [
  <home-manager/nixos>
  ...
];

...
  users.users.eve.isNormalUser = true;
  home-manager.users.eve = { pkgs, ... }: {
    home.packages = [  ];
    programs.bash.enable = true;
  
    # The state version is required and should stay at the version you
    # originally installed.
    home.stateVersion = "25.05";
  };
```



重建系统



```
sudo nixos-rebuild switch
```



配置文件路径：`/etc/nixos/home.nix`

更新用户配置：`sudo nixos-rebuild switch`



### 1.2 standalone



优点：用户级重建，更改用户配置无需重建整个系统；可移植性



适用范围：



*   多用户NixOS
*   需要较高灵活性



添加 home-manager 通道



[channel_link](https://nix-community.github.io/home-manager/index.xhtml#sec-install-standalone)



```
nix-channel --add <channel_link>

nix-channel --update
```



安装 home-manager



```
nix-shell '<home-manager>' -A install
```



配置文件路径：`~/.config/home-manager/home.nix`

用户配置更新：`home-manager switch`



## 2. other release



>   [!important]
>
>   该部分只能作为独立软件包进行安装



### installation



开启实验性功能

```shell
mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" > ~/.config/nix/nix.conf
sudo systemctl restart nix-daemon
```



#### 2.1 auto



```
nix run home-manager/master -- init --switch
```



生成默认配置在 `~/.config/home-manager/home.nix`



#### 2.2 manual



首先创建目录结构



```
/home/arch/.config/home-manager
├── flake.nix
└── home.nix
```



在 home.nix 中写入



```nix
{ config, pkgs, ...}:

{

  imports = [
  
  ]

  users.users.<username> = {
    name = "<username>";
    home = "/Users/<username>";
  };
  home-manager.users.eve = { pkgs, ... }: {
    home.packages = [  ];
    programs.bash.enable = true;

    # The state version is required and should stay at the version you
    # originally installed.
    home.stateVersion = "25.05";
  };
}
```



在 flake.nix 中写入



```nix
{
  description = "Home Manager configuration of arch";

  inputs = {
    # Specify the source of Home Manager and Nixpkgs.
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { 
    nixpkgs, 
    home-manager,
    ...
  }:let
    system = "x86_64-linux";
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    homeConfigurations = {
      "<username>" = home-manager.lib.homeManagerConfiguration {
        inherit pkgs;
        # Specify your home configuration modules here, for example,
        # the path to your home.nix.
        modules = [ 
          ./home.nix 
        ];
        # Optionally use extraSpecialArgs
        # to pass through arguments to home.nix
      };
    };
  };
}
```



运行以下命令安装 home-manager



```
nix run nixpkgs#home-manager --switch --flake nix/#USER
```



## 3. use home-manager



home-manager 安装软件包：



在 home.nix 的 `home.packages` 中写入软件包名再使用如下命令即可安装



```
home-manager switch --flake nix/#$USER
```



home-manager 管理软件包：



当前目录下任意层级创建 `<packages>.nix`，写入对应配置后再引入 home.nix



```nix
...
  imports = [
    <packages>.nix
  ]
```



最后重新运行 `home-manager switch` 即可



# Flake



编辑 Nix 配置文件（如果不存在则创建）：



```shell
sudo mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" | sudo tee -a ~/.config/nix/nix.conf
```



然后重启 Nix 守护进程：



```shell
sudo systemctl restart nix-daemon
```
