# 环境
| 系统      | 版本  |
|  ----     | ----  |
| 操作系统  | CentOS7 |
|postgresql|9.6|
| kong      | 1.5.1 |
|konga       |0.14.9 |
|nodejs     | 12.22.0 |

# 安装脚本
```
#安装 PostgreSQL
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql96-server
/usr/pgsql-9.6/bin/postgresql96-setup initdb
cd /var/lib/pgsql/9.6/data/
cat > pg_hba.conf <<EOF
local   all             postgres                                peer
host    all             all             127.0.0.1/32            trust
host    all             all             192.168.15.0/24         trust
EOF
sed -i "s@#listen_addresses = 'localhost'@listen_addresses = '*'@g" postgresql.conf
systemctl enable postgresql-9.6
systemctl start postgresql-9.6
systemctl status postgresql-9.6

# 授权Kong数据库
su - postgres << EOF
psql << XOF
CREATE USER kong with password 'kong'; CREATE DATABASE kong OWNER kong;
XOF
EOF

# 安装kong
mkdir /soft -p
cd /soft
yum install -y wget
wget -O 'kong.rpm' https://bintray.com/kong/kong-rpm/download_file?file_path=centos/7/kong-1.5.1.el7.amd64.rpm
yum localinstall -y --nogpgcheck kong.rpm
cp /etc/kong/kong.conf.default /etc/kong/kong.conf
sed -i "s@#database = postgres@database = postgres@" /etc/kong/kong.conf
sed -i "s@#pg_host = 127.0.0.1@pg_host = 127.0.0.1@" /etc/kong/kong.conf
sed -i "s@#pg_port = 5432@pg_port = 5432@" /etc/kong/kong.conf
sed -i "s@#pg_timeout = 5000@pg_timeout = 5000@" /etc/kong/kong.conf
sed -i "s@#pg_user = kong@pg_user = kong@" /etc/kong/kong.conf
sed -i "s@#pg_password =@pg_password = kong@" /etc/kong/kong.conf
sed -i "s@#pg_database = kong@pg_database = kong@" /etc/kong/kong.conf
sed -i "s@#pg_ssl = off@pg_ssl = off@" /etc/kong/kong.conf

kong migrations bootstrap -c /etc/kong/kong.conf

# 修改文件描述符打开数
echo "* soft nofile 65535"  >> /etc/security/limits.conf
echo "* hard nofile 65535"  >> /etc/security/limits.conf
kong start -c /etc/kong/kong.conf

# nodejs
cd /soft
wget https://nodejs.org/dist/latest-v12.x/node-v12.22.0-linux-x64.tar.gz
tar -xf node-v12.22.0-linux-x64.tar.gz -C /opt
ln -sv node-v12.22.0-linux-x64 node
ln -sv /opt/node/bin/* /usr/bin/
npm install -g gulp
npm install -g bower
npm install -g sails

# konga
yum install -y git
git clone https://github.com/pantsel/konga.git /opt/konga
adduser kong
chown -R kong.kong /opt/konga
su - kong  << EOF
cd /opt/konga && npm i
EOF

# 创建postgresql数据库Konga
su - postgres << EOF
psql << XOF
CREATE USER konga with password 'konga'; CREATE DATABASE konga OWNER konga;grant all privileges on database konga to konga;
XOF
EOF

cd /opt/konga
cat > /opt/konga/.env <<EOF
PORT=1337
NODE_ENV=production
KONGA_HOOK_TIMEOUT=120000
DB_ADAPTER=postgres
DB_URI=postgresql://konga:konga@localhost:5432/konga
KONGA_LOG_LEVEL=warn
TOKEN_SECRET=some_secret_token
EOF
node ./bin/konga.js prepare --adapter postgres --uri postgresql://konga:konga@127.0.0.1:5432/konga

npm run production
```