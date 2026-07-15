# Manual Setup Commands

The `manual-provisioning/Vagrantfile` only defines the 5 VMs — it doesn't provision anything automatically. Everything below was run by hand over SSH on each VM, in this order (later services depend on earlier ones being up):

```
MySQL (Database)      →  Memcache (DB Caching)  →  RabbitMQ (Broker/Queue)  →  Tomcat (Application)  →  Nginx (Web)
```

---

## 1. MySQL Setup (db01)

```bash
vagrant ssh db01

# Verify hosts entry
cat /etc/hosts

# Update OS and install
sudo dnf update -y
sudo dnf install epel-release -y
sudo dnf install git mariadb-server -y

# Start and enable
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Secure installation (set root password to admin123, remove anonymous users/test db)
sudo mysql_secure_installation

# Create app database and user
sudo mysql -u root -padmin123
```
```sql
create database accounts;
grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123';
grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
FLUSH PRIVILEGES;
exit;
```
```bash
# Download source and load schema
cd /tmp/
git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project
mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql

# Open firewall for MySQL
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo systemctl restart mariadb
```

---

## 2. Memcache Setup (mc01)

```bash
vagrant ssh mc01

sudo dnf install epel-release -y
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached

# Bind to all interfaces, not just localhost
sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached

# Open firewall
sudo firewall-cmd --add-port=11211/tcp --permanent
sudo firewall-cmd --add-port=11111/udp --permanent
sudo firewall-cmd --reload

sudo memcached -p 11211 -U 11111 -u memcached -d
```

---

## 3. RabbitMQ Setup (rmq01)

```bash
vagrant ssh rmq01

sudo dnf install epel-release -y
sudo dnf install wget -y
sudo dnf -y install centos-release-rabbitmq-38
sudo dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
sudo systemctl enable --now rabbitmq-server

# Allow non-localhost users
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
sudo systemctl restart rabbitmq-server

# Open firewall
sudo firewall-cmd --add-port=5672/tcp --permanent
sudo firewall-cmd --reload
```

---

## 4. Tomcat Setup + App Deploy (app01)

```bash
vagrant ssh app01

sudo dnf update -y
sudo dnf install epel-release -y
sudo dnf -y install java-17-openjdk java-17-openjdk-devel
sudo dnf install git wget -y

cd /tmp/
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
tar xzvf apache-tomcat-10.1.26.tar.gz

sudo useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
sudo cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/
sudo chown -R tomcat.tomcat /usr/local/tomcat
```

Create `/etc/systemd/system/tomcat.service`:
```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
User=tomcat
Group=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINE_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat

# Open firewall
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

**Build and deploy the app:**
```bash
cd /tmp/
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
sudo cp -r apache-maven-3.9.9 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"

git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project
# Edit src/main/resources/application.properties with backend server details
/usr/local/maven3.9/bin/mvn install

sudo systemctl stop tomcat
sudo rm -rf /usr/local/tomcat/webapps/ROOT*
sudo cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
sudo chown tomcat.tomcat /usr/local/tomcat/webapps -R
sudo systemctl start tomcat
```

---

## 5. Nginx Setup (web01)

```bash
vagrant ssh web01
sudo -i

apt update
apt upgrade -y
apt install nginx -y
```

Create `/etc/nginx/sites-available/vproapp`:
```nginx
upstream vproapp {
    server app01:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://vproapp;
    }
}
```

```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

---

## Notes

- The firewall commands above were **the ones that repeatedly failed to persist** across `vagrant reload` during this project (see [DEBUGGING_NOTES.md](DEBUGGING_NOTES.md)) — if you're following this yourself and hit a 502 or a connection error, check `firewall-cmd --list-all` on the relevant VM first.
- `application.properties` needs backend hostnames (`db01`, `mc01`, `rmq01`) set correctly before `mvn install` — see the repo root `application.properties.example`.
