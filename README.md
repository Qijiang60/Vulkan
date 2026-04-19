# Vulkan
离线安装 Vulkan 工具链与依赖库操作指南

目标软件包列表：
vulkan-tools libvulkan-dev libx11-dev libdrm-dev liblzma-dev libbz2-dev 
pkg-config libavformat-dev libavcodec-dev libavutil-dev libswresample-dev

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📦 方法一：创建本地 APT 仓库（推荐，稳定可靠）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【在联网机上操作】

1. 确认环境一致性
   # 检查系统版本与架构（必须与离线机完全相同）
   lsb_release -a
   dpkg --print-architecture

   # 若版本不一致，可使用 Docker 模拟（示例 Ubuntu 20.04）：
   # docker run -it --name temp-offline ubuntu:20.04 bash
   # 进入容器后执行 apt update，再继续以下步骤

2. 安装必要工具
   sudo apt update
   sudo apt install -y apt-rdepends dpkg-dev

3. 创建存放目录并递归下载所有包
   mkdir -p ~/offline-vulkan-repo
   cd ~/offline-vulkan-repo

   PACKAGES="vulkan-tools libvulkan-dev libx11-dev libdrm-dev liblzma-dev libbz2-dev pkg-config libavformat-dev libavcodec-dev libavutil-dev libswresample-dev"

   # 获取完整依赖列表并下载
   DEPS=$(apt-rdepends $PACKAGES | grep -v "^ " | sort -u)
   for pkg in $DEPS; do
       apt-get download $pkg 2>/dev/null
   done

4. 生成本地仓库索引文件
   dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz

5. 打包整个目录
   cd ~
   tar -czvf offline-vulkan-repo.tar.gz offline-vulkan-repo
   md5sum offline-vulkan-repo.tar.gz   # 记录校验值

【将 offline-vulkan-repo.tar.gz 传输到离线机，假设放在 /tmp 目录】

【在离线机上操作】

1. 解压并进入目录
   cd /tmp
   tar -xzvf offline-vulkan-repo.tar.gz
   cd offline-vulkan-repo

2. 配置本地软件源
   echo "deb [trusted=yes] file:///tmp/offline-vulkan-repo ./" | sudo tee /etc/apt/sources.list.d/local-offline.list

3. 更新源并安装软件包
   sudo apt update
   sudo apt install -y vulkan-tools libvulkan-dev libx11-dev libdrm-dev liblzma-dev libbz2-dev pkg-config libavformat-dev libavcodec-dev libavutil-dev libswresample-dev

4. 验证安装
   vulkaninfo --summary
   pkg-config --modversion vulkan

5. 清理临时源（推荐）
   sudo rm /etc/apt/sources.list.d/local-offline.list
   sudo apt update


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 方法二：手动收集并修复依赖（依赖较浅时备用）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【在联网机上操作 - 首次下载】

1. 仅下载明确列出的软件包（不处理依赖）
   mkdir ~/vulkan-offline-manual
   cd ~/vulkan-offline-manual
   apt-get download vulkan-tools libvulkan-dev libx11-dev libdrm-dev liblzma-dev libbz2-dev pkg-config libavformat-dev libavcodec-dev libavutil-dev libswresample-dev

2. 打包并传输到离线机
   cd ~
   tar -czvf vulkan-offline-manual.tar.gz vulkan-offline-manual
   # 将 tar.gz 文件复制到离线机，假设放在 /tmp 目录

【在离线机上操作 - 循环安装与修复】

1. 解压并尝试安装
   cd /tmp
   tar -xzvf vulkan-offline-manual.tar.gz
   cd vulkan-offline-manual
   sudo dpkg -i *.deb

2. 记录缺失的依赖包
   # 查看因依赖缺失导致的报错，例如：
   # dpkg: dependency problems... Package libavcodec58 is not installed.
   # 记录下缺失的包名：libavcodec58 libavutil56 libswresample3 libxcb1 libdrm2 libvulkan1 ...

   # 辅助命令：列出未满足的依赖关系
   sudo apt-get install -f --dry-run | grep "^ " | awk '{print $1}' > missing.txt

【返回联网机，下载缺失的包】

1. 下载记录的所有缺失包
   cd ~/vulkan-offline-manual
   apt-get download <将缺失包名以空格分隔列出>

2. 重新打包并传回离线机
   cd ~
   tar -czvf vulkan-offline-manual.tar.gz vulkan-offline-manual
   # 再次用 U 盘传回

【在离线机上，重复以下过程直到不报错】

- 解压覆盖目录
- sudo dpkg -i *.deb
- 记录新缺失的包名 -> 回到联网机下载 -> 再次传回安装

【最终修复与验证】

1. 运行修复命令（即使未显示错误）
   sudo apt --fix-broken install

2. 验证 Vulkan 环境
   vulkaninfo | head -n 10
   ls /usr/include/vulkan/


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ 关键注意事项（两种方法均适用）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 联网机必须与离线机具有相同的 Ubuntu/Debian 版本和 CPU 架构（amd64/arm64）。
2. 虚拟包（如 libgl1）无法直接下载，需下载其具体实现包（如 libgl1-mesa-glx）。
3. 若方法二手动循环超过 3-4 轮仍不断出现新依赖，请改用方法一。

祝安装顺利！
