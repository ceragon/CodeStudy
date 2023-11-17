# 编译 OpenJdk

## 依赖安装

```shell
# 编译环境
apt install -y cmake g++ gdb zip unzip autoconf libasound2-dev libcups2-dev libfontconfig1-dev libx11-dev libxext-dev libxrender-dev libxrandr-dev libxtst-dev libxt-dev

# 安装低版本或同版本jdk，推荐 sdkman
curl -s "https://get.sdkman.io" | bash
sdk install java 21-open
```

## configure

```shell
chmod 755 configure
sudo apt install -y clang
./configure --with-debug-level=slowdebug --disable-warnings-as-errors --with-toolchain-type=clang --build=x86_64-unknown-linux-gnu --openjdk-target=x86_64-unknown-linux-gnu
```

## make

```shell
make hotspot compile-commands-hotspot
```
