对于有root权限的安卓手机（小米手机基于安卓6的miui11为例）。
VX登录情况下，断网。（VX版本为7.0.17）
准备centos7主机或虚拟机，有互联网连接，建议内存4G起步，硬盘20G起步，多多益善。
准备好你的VX的数据库，数据库密码，资源文件。
数据库在:
/data/data/com.tencent.mm/MicroMsg/32位的id/EnMicroMsg.db
数据库密码为:
将IMEI与uin拼接起来，打开https://md5jiami.51240.com/ 计算拼接出MD5值，使用32位小写，取出前7位。这前7位即为数据库密码。
**********
IMEI获取：手机输入*#06#查看手机的IMEI，如果有多个IMEI可以都记录下来尝试。
uin获取：打开/data/data/com.tencent.mm/shared_prefs/auth_info_key_prefs.xml其中 <int name="_auth_uin" value="-8888888888" />的uin是-8888888888，包含负号。
**********
资源文件：
/data/data/com.tencent.mm/MicroMsg/32位的id/下的
avatar（头像文件夹）
image2（图片文件夹）
/storage/emulated/0/Android/data/com.tencent.mm/MicroMsg/32位的id/下的
avatar（头像文件夹，和上面的合并，使资源完整）
emoji（表情包文件夹）
image2（图片文件夹，和上面的合并，使资源完整）
sfs（如果没有请建立一个空的同名文件夹）
video（视频文件夹）
voice2（语音文件夹）
整理文件：
建立文件夹resource，将avatar,emoji,image2,sfs,video,voice2这6个文件夹放入resource文件夹中，将resource文件夹打包为resource.zip备用。
EnMicroMsg.db单独放置备用。

下载以下3个文件到电脑备用：
sox-plugins-freeworld-14.4.1-3.el7.x86_64.rpm
https://download1.rpmfusion.org/free/el/updates/7/x86_64/s/sox-plugins-freeworld-14.4.1-3.el7.x86_64.rpm

sqlcipher-2.2.1.tar.gz
https://github.com/sqlcipher/sqlcipher/archive/v2.2.1.tar.gz

wechat-dump-master.zip （下载下来后改名为wechat-dump-master.zip）
https://github.com/ppwwyyxx/wechat-dump/archive/dfdcd4ba0fcc265a04371036ce33e9d188e0406c.zip

打开centos7主机，我用的SecureCRT通过ssh连接到centos7主机的。

---------------------------------------------------------------
【可选项】把repo源改为国内的阿里云的repo源，下载安装软件更快：
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
---------------------------------------------------------------

安装软件
yum install -y sqlite sqlite-devel openssl openssl-devel gcc gcc-c++ sox lrzsz unzip

建立并进入/data文件夹
mkdir /data
cd /data

将电脑上的以下5个文件传入centos7主机上的/data文件夹下
直接拖动文件到SecureCRT的窗口即可
resource.zip
EnMicroMsg.db
sox-plugins-freeworld-14.4.1-3.el7.x86_64.rpm
sqlcipher-2.2.1.tar.gz
wechat-dump-master.zip 

继续下列操作
yum install -y sox-plugins-freeworld-14.4.1-3.el7.x86_64.rpm
tar xvf sqlcipher-2.2.1.tar.gz
cd sqlcipher-2.2.1/
./configure --enable-tempstore=yes CFLAGS="-DSQLITE_HAS_CODEC" LDFLAGS="-lcrypto" --prefix=/usr/local/sqlcipher
make && make install
ln -s /usr/local/sqlcipher/bin/sqlcipher /usr/bin/sqlcipher

解密数据库文件，下面命令行里的密码需要替换成前面计算出的7位数据库密码
cd /data
sqlcipher EnMicroMsg.db 'PRAGMA key="这里换成7位数据库密码";PRAGMA cipher_use_hmac = off;ATTACH DATABASE "decrypted_database.db" AS decrypted_database KEY "";SELECT sqlcipher_export("decrypted_database");DETACH DATABASE decrypted_database;'
看下decrypted_database.db的文件大小，若为0则密码不对，解密失败
ll -t
！！！只有解密成功才能继续后面的操作！！！

安装所需的Python3.6
yum install -y https://repo.ius.io/ius-release-el7.rpm https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install -y python36
yum install -y python36-pip

---------------------------------------------------------------
【可选项】修改pip源为国内阿里云pip源加快下载安装速度
vim /etc/pip.conf
按键盘上小写的i键
复制下面两行#中间的内容（不含上下两行#字符）
############################
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
############################
在SecureCRT的窗口点一下鼠标右键就可以粘贴，如有提示确定即可
按键盘上的esc键，输入:wq，然后回车
---------------------------------------------------------------

安装Python所需模块
pip3 install requests
pip3 install numpy

继续下列操作搭建环境
cd /data
unzip wechat-dump-master.zip
cd wechat-dump-master/
./third-party/compile_silk.sh
pip3 install -r requirements.txt

复制解密后的数据库文件到指定文件夹下
cp /data/decrypted_database.db /data/wechat-dump-master/decrypted.db

移动资源文件到指定文件夹下并解压，并删除zip压缩包节省硬盘空间
mv /data/resource.zip /data/wechat-dump-master/
unzip resource.zip
rm -f resource.zip

【至此所有准备工作都已经完成了，可以开始把聊天记录导出html了】
进入指定文件夹才能正常进行后续操作
cd /data/wechat-dump-master

导出所有聊天的文字内容到当前文件夹下的output_dir文件夹里：
./dump-msg.py decrypted.db output_dir

列出所有聊天（第一列为名字，第二列为ID）：
./list-chats.py decrypted.db

根据output_dir文件夹里的内容，统计每个聊天的大概行数、字符数：
./count-message.sh output_dir

将某个聊天导出html文件方便在浏览器上查看（需解密的数据库文件、resource文件夹里的文件）：
./dump-html.py "这里换成聊天的ID"
