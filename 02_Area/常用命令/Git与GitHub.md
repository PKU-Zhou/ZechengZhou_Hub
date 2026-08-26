# GitHub配置SSH Key
我们在往github上push项目的时候，如果走https的方式，每次都需要输入账号密码，非常麻烦。

而采用ssh的方式，就不再需要输入，只需要在github自己账号下配置一个ssh key即可。

SSH登录安全性由非对称加密 保证，产生密钥时，一次产生两个密钥，一个公钥放在远程主机（也就是github网站），一个私钥放在本地。 在git中一般命名为id_rsa.pub, id_rsa。 因此我们要做的就是将id_rsa.pub添加到github网站上。

1. 检查本地是否已经存在SSH Key
```bash
cd ~/.ssh
ls
#看是否存在 id_rsa 和 id_rsa.pub文件，如果存在，说明已经有SSH Key
```
2. 生成密钥和公钥
```bash
ssh-keygen -t rsa -C "zch.zhou@outlook.com"
# 执行后一直回车即可
```

3. 获取公钥并添加到GitHub
```bash
cd ~/.ssh
cat id_rsa.pub
```
![[Pasted image 20260825201352.png]]

4. 将密钥添加到ssh-agent中，为ssh client指定使用的密钥文件
 ```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
 ```
 
5. 配置IDE中的ssh config文件
不使用默认的22端口
```bash
Host github.com
    HostName ssh.github.com
    User git
    Port 445
    IdentityFile ~/.ssh/id_rsa
    IdentitiesOnly yes
    ProxyCommand nc -X 5 -x 127.0.0.1:20000 %h %p
```
6. 验证是否添加成功
```bash
ssh -T git@github.com
```
显示如下信息即表示设置成功：
```shell
Hi PKU-Zhou! You've successfully authenticated, but GitHub does not provide shell access.
```