# Vagrant-Ansible
3 Deployable Ansible environments w/ webservers, databases and a loadbalancer

Op de host machine moet Virtualbox, Vagrant en Ansible geïnstalleerd zijn om gebruik te kunnen maken van deze ICE.

Maak binnen de home directory een map aan genaamd playbooks, waar de playbooks en vagrantfile geplaatst kunnen worden.

Bronze
1.	Navigeer naar de map playbooks en maak hier een map genaamd bronze aan.
2.	Rechtermuisknop binnen deze map en klik op Open new terminal here. In de terminal die opent type je: vagrant iniit ubuntu/trusty64.
In de map wordt nu een vagrantfile aangemaakt waarmee de VM die gebruikt wordt geconfigureerd kan worden.
3.	Pas de vagrantfile aan zodat hij er als volgt uit komt te zien:

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use the same key for each machine
  config.ssh.insert_key = false

  config.vm.define "Webserver" do |vagrant1|
    vagrant1.vm.box = "ubuntu/trusty64"
    vagrant1.vm.network "forwarded_port", guest: 80, host: 8080
    vagrant1.vm.network "forwarded_port", guest: 443, host: 8443
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "web-tls.yml"
    end
  end
end

4.	Maak vervolgens de bestanden ansible.cfg, hosts en web.yml aan om de webserver te kunnen configureren. Vul de bestanden aan met de volgende tekst:

Ansible.cfg
[defaults]
inventory = hosts
remote_user = vagrant
private_key_file = ~/.vagrant.d/insecure_private_key
host_key_checking = False

Hosts
Webserver ansible_host=127.0.0.1 ansible_port=2223


5.	Maak nu een map aan genaamd templates.
6.	In templates komen de config bestanden te staan voor de webserver. Maak hier het bestand defaults aan met de volgende tekst:
	server {
	listen 80 default_server;
	listen [::]:80 default_server ipv6only=on;
	
	listen 443 ssl;

	root /usr/share/nginx/html;
	index index.html index.htm;

	server_name {{ server_name }};
	ssl_certificate {{ cert_file }};
	ssl_certificate_key {{ key_file }};
	
	location / {
		try_files $uri $uri/ =404;
		}
	}
En het bestand index.html.j2 met de volgende tekst:
<html>
	<head>
	<title>Welcome to ansible</title>
	</head>
	<body>
	<h1>nginx, configured by Ansible</h1>
	<p>If you can see this, Ansible successfully installed nginx.</p>
	<p>{{ ansible_managed }}</p>
	</body>
	</html>

7.	Als dit klaar is typ je in de terminal het command: vagrant up
Hiermee wordt de virtuele machine gestart en provisioned door ansible.
Silver
1.	Navigeer naar de map playbooks en maak hier een map genaamd silver aan.
2.	Rechtermuisknop binnen deze map en klik op Open new terminal here. In de terminal die opent type je: vagrant iniit ubuntu/trusty64.
In de map wordt nu een vagrantfile aangemaakt waarmee de VM die gebruikt wordt geconfigureerd kan worden.
3.	Pas de vagrantfile aan zodat hij er als volgt uit komt te zien:

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use the same key for each machine
  config.ssh.insert_key = false

  config.vm.define "web1" do |webservers|
    webservers.vm.box = "ubuntu/trusty64"
    webservers.vm.network "forwarded_port", guest: 80, host: 8081
    webservers.vm.network "forwarded_port", guest: 443, host: 8444
    webservers.vm.network "private_network", ip:"192.168.1.3",
        virtualbox__intnet: true
    config.vm.hostname="web1"
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "site.yml"
    end
  end
  config.vm.define "web2" do |web2|
    web2.vm.box = "ubuntu/trusty64"
    web2.vm.network "forwarded_port", guest: 80, host: 8082
    web2.vm.network "forwarded_port", guest: 443, host: 8445
    web2.vm.network "private_network", ip:"192.168.1.4",
        virtualbox__intnet: true
    config.vm.hostname="web2"
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "site.yml"
    end
  end
  config.vm.define "db1" do |db1|
    db1.vm.box = "ubuntu/trusty64"
    db1.vm.network "private_network", ip:"192.168.1.5",
	virtualbox__intnet: true
    config.vm.provision "ansible" do |ansible|
	ansible.playbook = "site.yml"
    end
  end
end

4.	Maak vervolgens de bestanden ansible.cfg, hosts en site.yml aan om de webserver te kunnen configureren. Vul de bestanden aan met de volgende tekst:

Ansible.cfg
[defaults]
inventory = hosts
remote_user = vagrant
private_key_file = ~/.vagrant.d/insecure_private_key
host_key_checking = False

Hosts
[webservers]
web1 ansible_host=127.0.0.1 ansible_port=2224
web2 ansible_host=127.0.0.1 ansible_port=2225

[database]
db1 ansible_host=127.0.0.1 ansible_port=2226

5.	Maak nu een mapje aan met roles en maak hierin mapjes aan voor de database, lodabalancer en webservers. Hierin worden de .yml bestanden gezet voor het provisionen van de webservers en database server.

Webservers:
---
- apt: update_cache=yes
  environment: proxy_env

# Install the package "mysql"
- apt: name={{item}} state=present
  with_items:
   - nginx
   - php5-fpm
   - php5-mysql
  environment: proxy_env

- name: Create nginx configuration file
  template: src=default dest=/etc/nginx/sites-available/default
  notify:
  - restart nginx

- name: Add wall.php script that demonstrates php and mysql functionality
  template: src=index.php dest=/usr/share/nginx/html/index.php

- name: nginx service state
  service: name=nginx state=started enabled=yes

Database:
---
- apt: update_cache=yes
  environment: proxy_env

# Install the packages
- apt: name={{item}} state=present
  with_items:
   - python-pip
   - python-dev
   - mysql-server
   - libmysqlclient-dev
  environment: proxy_env
 
- pip: name=MySQL-python
  environment: proxy_env

- name: Create Mysql configuration file
  lineinfile: dest=/etc/mysql/my.cnf regexp='^bind-address(\s*)=' line='bind-address\1= {{ mysql_host }}'  backrefs=yes
  notify:
  - restart mysql

- name: Start Mysql Service
  service: name=mysql state=started enabled=true

- name: Create Application Database
  mysql_db: name={{ dbname }} state=present

- name: Create Application DB User
  mysql_user: name={{ dbuser }} password={{ upassword }} priv=*.*:ALL host='%' state=present

6. In het mapje van de webservers moet nog een mapje aangemaakt worden waar de configuratie bestanden komen.

Index.php.j2:

<html>
<head>
<style>
table {
    font-family: arial, sans-serif;
    border-collapse: collapse;
    width: 100%;
}

td, th {
    border: 1px solid #dddddd;
    text-align: left;
    padding: 8px;
}

tr:nth-child(even) {
    background-color: #dddddd;
}
</style>
<title>Welcome to ansible</title>
</head>
<body>
<h1>Thank you for choosing the Gold Package.</h1>
<p>If you can see this, Ansible successfully installed nginx.</p>

<p>This is Webserver: <?php echo gethostname(); ?></p>

<p>{{ ansible_managed }}</p>


<?php

// database credentials (defined in group_vars/all)
$dbname = "dbtest";
$dbuser = "dbuser";
$dbpass = "secret";
$dbhost = "192.168.1.5";

// query templates
$create_table = "CREATE TABLE IF NOT EXISTS `wall` (
   `id` int(11) unsigned NOT NULL auto_increment,
   `title` varchar(255) NOT NULL default '',
   `content` text NOT NULL default '',
   PRIMARY KEY  (`id`)
   ) ENGINE=MyISAM  DEFAULT CHARSET=utf8";
$select_wall = 'SELECT * FROM wall';

// Connect to and select database
$link = mysql_connect($dbhost, $dbuser, $dbpass)
    or die('Could not connect: ' . mysql_error());
echo "Connected successfully\n<br />\n";
mysql_select_db($dbname) or die('Could not select database');

// create table
$result = mysql_query($create_table) or die('Create Table failed: ' . mysql_error());

// handle new wall posts
if (isset($_POST["title"])) {
    $result = mysql_query("insert into wall (title, content) values ('".$_POST["title"]."', '".$_POST["content"]."')") or die('Create Table failed: ' . mysql_error());
}

// Performing SQL query
$result = mysql_query($select_wall) or die('Query failed: ' . mysql_error());

// Printing results in HTML
echo "<table>\n";
while ($line = mysql_fetch_array($result, MYSQL_ASSOC)) {
    echo "\t<tr>\n";
    foreach ($line as $col_value) {
        echo "\t\t<td>$col_value</td>\n";
    }
    echo "\t</tr>\n";
}
echo "</table>\n";

// Free resultset
mysql_free_result($result);

// Closing connection
mysql_close($link);
?>

<form method="post">
Title: <input type="text" name="title"><br />
Message: <textarea name="content"></textarea><br />
<input type="submit" value="Post message">
</form>





</body>
</html>

Defaults
server {
	listen 80 default_server;
	listen [::]:80 default_server ipv6only=on;

	root /usr/share/nginx/html;
	index index.php index.html index.htm;

	server_name localhost;

	location / {
		try_files $uri $uri/ =404;
		
	}

	error_page 404 /404.html;

	
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
		root /usr/share/nginx/html;
	}


	location ~ \.php$ {
		fastcgi_split_path_info ^(.+\.php)(/.+)$;
		fastcgi_pass unix:/var/run/php5-fpm.sock;
		fastcgi_index index.php;
		include fastcgi_params;
	}
}

7. Als dit klaar is typ je in de terminal het command: vagrant up
Hiermee worden de virtuele machines gestart en provisioned door ansible.
Gold
6.	Navigeer naar de map playbooks en maak hier een map genaamd gold aan.
7.	Rechtermuisknop binnen deze map en klik op Open new terminal here. In de terminal die opent type je: vagrant iniit ubuntu/trusty64.
In de map wordt nu een vagrantfile aangemaakt waarmee de VM die gebruikt wordt geconfigureerd kan worden.
8.	Pas de vagrantfile aan zodat hij er als volgt uit komt te zien:

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use the same key for each machine
  config.ssh.insert_key = false

  config.vm.define "web1" do |webservers|
    webservers.vm.box = "ubuntu/trusty64"
    webservers.vm.network "forwarded_port", guest: 80, host: 8081
    webservers.vm.network "forwarded_port", guest: 443, host: 8444
    webservers.vm.network "private_network", ip:"192.168.1.3",
        virtualbox__intnet: true
    config.vm.hostname="web1"
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "site.yml"
    end
  end
  config.vm.define "web2" do |web2|
    web2.vm.box = "ubuntu/trusty64"
    web2.vm.network "forwarded_port", guest: 80, host: 8082
    web2.vm.network "forwarded_port", guest: 443, host: 8445
    web2.vm.network "private_network", ip:"192.168.1.4",
        virtualbox__intnet: true
    config.vm.hostname="web2"
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "site.yml"
    end
  end
  config.vm.define "db1" do |db1|
    db1.vm.box = "ubuntu/trusty64"
    db1.vm.network "private_network", ip:"192.168.1.5",
	virtualbox__intnet: true
    config.vm.provision "ansible" do |ansible|
	ansible.playbook = "site.yml"
    end
  end
  config.vm.define "lb1" do |lb1|
    lb1.vm.box = "hashicorp-vagrant/ubuntu-16.04"
    lb1.vm.network "forwarded_port", guest: 80, host: 8080
    lb1.vm.network "forwarded_port", guest: 443, host: 8443
    lb1.vm.network "private_network", ip:"192.168.1.6",
	virtualbox__intnet: true
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "lb.yml"
    end
  end
end

9.	Maak vervolgens de bestanden ansible.cfg, hosts en site.yml aan om de webserver te kunnen configureren. Vul de bestanden aan met de volgende tekst:

Ansible.cfg
[defaults]
inventory = hosts
remote_user = vagrant
private_key_file = ~/.vagrant.d/insecure_private_key
host_key_checking = False

Hosts
[webservers]
web1 ansible_host=127.0.0.1 ansible_port=2222
web2 ansible_host=127.0.0.1 ansible_port=2200

[database]
db1 ansible_host=127.0.0.1 ansible_port=2201

[loadbalancer]
lb1 ansible_host=127.0.0.1 anisble_port=2202

10.	Maak nu een mapje aan met roles en maak hierin mapjes aan voor de database, lodabalancer en webservers. Hierin worden de .yml bestanden gezet voor het provisionen van de webservers en database server.

Webservers:
---
- apt: update_cache=yes
  environment: proxy_env

# Install the package "mysql"
- apt: name={{item}} state=present
  with_items:
   - nginx
   - php5-fpm
   - php5-mysql
  environment: proxy_env

- name: Create nginx configuration file
  template: src=default dest=/etc/nginx/sites-available/default
  notify:
  - restart nginx

- name: Add wall.php script that demonstrates php and mysql functionality
  template: src=index.php dest=/usr/share/nginx/html/index.php

- name: nginx service state
  service: name=nginx state=started enabled=yes

Database:
---
- apt: update_cache=yes
  environment: proxy_env

# Install the packages
- apt: name={{item}} state=present
  with_items:
   - python-pip
   - python-dev
   - mysql-server
   - libmysqlclient-dev
  environment: proxy_env
 
- pip: name=MySQL-python
  environment: proxy_env

- name: Create Mysql configuration file
  lineinfile: dest=/etc/mysql/my.cnf regexp='^bind-address(\s*)=' line='bind-address\1= {{ mysql_host }}'  backrefs=yes
  notify:
  - restart mysql

- name: Start Mysql Service
  service: name=mysql state=started enabled=true

- name: Create Application Database
  mysql_db: name={{ dbname }} state=present

- name: Create Application DB User
  mysql_user: name={{ dbuser }} password={{ upassword }} priv=*.*:ALL host='%' state=present

Loadbalancer
- name: Configure the loadbalancer
  hosts: lb1
  sudo: True
  tasks:
     - name: enable ufw
       service: name=ufw state=started

     - name: allow http traffic
       ufw: rule=allow port=80 proto=tcp

     - name: install nginx
       apt: name=nginx update_cache=yes
   
     - name: start nginx
       service: name=nginx state=started

     - name: copy configuration file
       copy: src=templates/default dest=/etc/nginx/sites-available/default

     - name: enable configuration
       file: >
         dest=/etc/nginx/sites-enabled/default
         src=/etc/nginx/sites-available/default

     - name: restart nginx
       service: name=nginx state=restarted

6. In het mapje van de webservers moet nog een mapje aangemaakt worden waar de configuratie bestanden komen.

Index.php.j2:

<html>
<head>
<style>
table {
    font-family: arial, sans-serif;
    border-collapse: collapse;
    width: 100%;
}

td, th {
    border: 1px solid #dddddd;
    text-align: left;
    padding: 8px;
}

tr:nth-child(even) {
    background-color: #dddddd;
}
</style>
<title>Welcome to ansible</title>
</head>
<body>
<h1>Thank you for choosing the Gold Package.</h1>
<p>If you can see this, Ansible successfully installed nginx.</p>

<p>This is Webserver: <?php echo gethostname(); ?></p>

<p>{{ ansible_managed }}</p>


<?php

// database credentials (defined in group_vars/all)
$dbname = "dbtest";
$dbuser = "dbuser";
$dbpass = "secret";
$dbhost = "192.168.1.5";

// query templates
$create_table = "CREATE TABLE IF NOT EXISTS `wall` (
   `id` int(11) unsigned NOT NULL auto_increment,
   `title` varchar(255) NOT NULL default '',
   `content` text NOT NULL default '',
   PRIMARY KEY  (`id`)
   ) ENGINE=MyISAM  DEFAULT CHARSET=utf8";
$select_wall = 'SELECT * FROM wall';

// Connect to and select database
$link = mysql_connect($dbhost, $dbuser, $dbpass)
    or die('Could not connect: ' . mysql_error());
echo "Connected successfully\n<br />\n";
mysql_select_db($dbname) or die('Could not select database');

// create table
$result = mysql_query($create_table) or die('Create Table failed: ' . mysql_error());

// handle new wall posts
if (isset($_POST["title"])) {
    $result = mysql_query("insert into wall (title, content) values ('".$_POST["title"]."', '".$_POST["content"]."')") or die('Create Table failed: ' . mysql_error());
}

// Performing SQL query
$result = mysql_query($select_wall) or die('Query failed: ' . mysql_error());

// Printing results in HTML
echo "<table>\n";
while ($line = mysql_fetch_array($result, MYSQL_ASSOC)) {
    echo "\t<tr>\n";
    foreach ($line as $col_value) {
        echo "\t\t<td>$col_value</td>\n";
    }
    echo "\t</tr>\n";
}
echo "</table>\n";

// Free resultset
mysql_free_result($result);

// Closing connection
mysql_close($link);
?>

<form method="post">
Title: <input type="text" name="title"><br />
Message: <textarea name="content"></textarea><br />
<input type="submit" value="Post message">
</form>





</body>
</html>

Defaults
server {
	listen 80 default_server;
	listen [::]:80 default_server ipv6only=on;

	root /usr/share/nginx/html;
	index index.php index.html index.htm;

	server_name localhost;

	location / {
		try_files $uri $uri/ =404;
		
	}

	error_page 404 /404.html;

	
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
		root /usr/share/nginx/html;
	}


	location ~ \.php$ {
		fastcgi_split_path_info ^(.+\.php)(/.+)$;
		fastcgi_pass unix:/var/run/php5-fpm.sock;
		fastcgi_index index.php;
		include fastcgi_params;
	}
}

7. Als dit klaar is typ je in de terminal het command: vagrant up
Hiermee worden de virtuele machines gestart en provisioned door ansible.
