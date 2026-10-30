## 无法翻墙的服务器，通过本地电脑的vpn实现翻墙

原理：在远程服务器上设置全局代理，使用本地网络转发

服务器程序 → 服务器 127.0.0.1:20000 → SSH 转发 → 本地代理/VPN → 目标网站


本地电脑上运行：
```shell
ssh -N -R 20000:127.0.0.1:7890 zch_zhou28@10.129.166.78
```

远程服务器上运行：
```shell
export http_proxy="socks5h://127.0.0.1:20000"
export https_proxy="socks5h://127.0.0.1:20000"
```
注意这里设置的端口号2000，要与上面本地电脑设置的端口号一致。

如果要取消代理：
```shell
unset http_proxy
unset https_proxy
```

走代理需要socks，但是如果出现缺少PySocks的情况，可以先绕过代理，通过清华镜像源安装：
```shell
env -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
    -u http_proxy -u https_proxy -u all_proxy \
    python -m pip install PySocks \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
```
安装后检查：
```shell
python -c "import socks; print('SOCKS support is ready')"
```
成功之后即可在python环境中使用SOCKS代理。