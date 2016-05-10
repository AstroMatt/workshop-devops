# Puppet

## Instalacja Puppet

	$ sudo apt-get install puppet

## Konfiguracja
Zaglądnij do katalogu `/etc/puppet`.
Co się tam znajduje?

Przejdź do katalogu `/etc/puppet/manifests`.

## Facter
Przyjrzyj się wynikom poleceń:

    $ facter
    $ facter ipaddress
    $ facter lsbdistdescription

Co zauważyłeś? Jak można wykorzystać te informacje?

## Zarządzanie użytkownikami i grupami
- Stwórz plik `/etc/puppet/manifests/www-data.pp`:
- Upewnij się że instnieje grupa www-data i ma gid=33
- Upewnij się, że istnieje user www-data i należy do grupy www-data i ma uid=33

```puppet
group { 'www-data':
	ensure => 'present',
	gid    => '33',
}

user { 'www-data':
	ensure           => 'present',
	comment          => 'www-data',
	groups           => ['www-data'],
	home             => '/var/www',
	password         => '*',
	password_max_age => '99999',
	password_min_age => '0',
	shell            => '/usr/sbin/nologin',
	uid              => '33',
}
```

## Podstawowa konfiguracja maszyny
- Zainstaluj Puppeta
- Zrób by Puppet wykonał polecenie `apt-get update`
- Upewnij się, że następujące paczki są zainstalowane:
    - `nmap`
    - `htop`
    - `git`

```puppet
exec { 'apt-get update':
  command => '/usr/bin/apt-get update';
}

Exec['apt-get update'] -> Package <| |>

package { ['htop', 'nmap', 'git']:
  ensure => present;
}
```

Alternatywnie instalację paczek można zapisać nie jako listę a osobne zasoby:

```puppet
exec { 'apt-get update':
  command => '/usr/bin/apt-get update',
}

Exec['apt-get update'] -> Package <| |>

package { 'htop':
	ensure => 'latest',
}

package { 'nmap':
	ensure => 'latest',
}

package { 'git':
	ensure => 'latest',
}
```

Możesz też podać update listy jako wymaganie do wykonania istalacji

```puppet
exec { 'package definition update':
	command => 'apt-get update',
}

package { ['nmap', 'htop', 'git']:
	ensure	=> 'latest',
	require => Exec['package definition update'],
}
```


## Zmiana hostname
- Za pomocą manifestu Puppeta zmień hostname maszyny na `ecosystem.local`
- Upewnij się, że po wpisaniu polecenia `hostname` będzie ustawiona na odpowiednią wartość.

```puppet
file { "/etc/hostname":
        ensure  => present,
        owner   => root,
        group   => root,
        mode    => '0644',
        content => "ecosystem.local\n",
        notify  => Exec["set-hostname"],
}

exec { "set-hostname":
        command => '/bin/hostname -F /etc/hostname',
        unless  => "/usr/bin/test `hostname` = `/bin/cat /etc/hostname`",
}
```

Alternatywnie

```puppet
exec { 'set hostname':
	command => '/usr/bin/hostnamectl set-hostname ecosystem.local'
}
```


## Konfiguracja Apache2
- Za pomocą Puppet upewnij się by był użytkownik `www-data` i miał `uid=33`
- Za pomocą Puppet upewnij się by była grupa `www-data` i miała `gid=33`
- Upewnij się że katalog `/var/www` istnieje i właścicielem jego są user `www-data` i grupa `www-data` i że ma uprawnienia `rwxr-xr-x`
- Zainstaluj i skonfiguruj Apache2 wykorzystując moduł Puppet
- Z terminala wygeneruj certyfikaty self signed OpenSSL (`.cert` i `.key`) (za pomocą i umieść je w `/etc/ssl/`)
- Za pomocą Puppet Stwórz dwa vhosty
    - `insecure.example.com` na porcie 80 i z katalogiem domowym `/var/www/insecure.example.com`
    - `ssl.example.com` na porcie 443 i z katalogiem domowym `/var/www/ssl.example.com` + używanie certyfikatów SSL wcześniej wygenerowanych

```puppet
file {
  '/var/www':
    ensure => 'directory',
    owner => 'www-data',
    group => 'www-data',
    mode  => '0755',
}
```

	$ puppet module install apache
	$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout self-signed.key -out self-signed.cert
	$ cat /etc/puppet/manifests/apache.pp

```puppet
class { 'apache':
	default_vhost => false,
}

# The non-ssl virtual host
apache::vhost { 'insecure.example.com':
	servername => 'insecure.example.com',
	port       => '80',
	docroot    => '/var/www/insecure',
}

# The SSL virtual host at the same domain
apache::vhost { 'ssl.example.com':
	servername => 'ssl.example.com',
	port       => '443',
	docroot    => '/var/www/ssl',
	ssl        => true,
	ssl_cert   => '/etc/ssl/self-signed.cert',
	ssl_key    => '/etc/ssl/self-signed.key',
}
```

	$ puppet apply /etc/puppet/manifests/apache.pp
	$ ls /var/www
	$ cat /etc/apache2/sites-enabled/*


## Instalacja i konfiguracja MySQL
- zainstaluj moduł bazy MySQL wykorzystując Puppeta
- ustaw hasło roota na mypassword
- ustaw nasłuchiwanie serwera `mysqld` na `0.0.0.0`
- stwórz bazę danych `mydb` z `utf-8`
- stwórz usera `myusername` z hasłem `mypassword`
- nadaj wszystkie uprawnienia dla usera `myusername` dla bazy `mydb`
- ustaw backupowanie bazy danych do `/tmp/mysql-backup`

	$ puppet module install puppetlabs-mysql

```puppet
# Install and configure MySQL
class { "mysql::server":
	root_password => "mypassword",
	#remove_default_accounts => true,
	override_options => {
		mysqld => {
			"bind_address"  => "0.0.0.0",
		}
	},
	databases => {
	'mydb' => {
		ensure  => 'present',
		charset => 'utf8',
	},
	},
	users => {
	'myusername@%' => {
		ensure          => 'present',
		password_hash   => mysql_password("mypassword"),
	},
	},
	grants => {
	'myusername@%/mydb.*' => {
		ensure      => 'present',
		privileges  => ["all"],
		table       => "mydb.*",
		user        => "myusername@%",
	},
	},
}

# Enable MySQL Backups
class { "mysql::server::backup":
	backupuser      => "myusername",
	backuppassword  => "mypassword",
	backupdir       => "/tmp/mysql_backup",
}
```

## Instalacja Java i Tomcat
- zainstaluj Javę za pomocą Puppeta
- zainstaluj Tomcat8 za pomocą Puppeta do `/opt/tomcat8`
- Skonfiguruj dwie instancje Tomcata działające jednocześnie
    - Jedna uruchamiana na domyślnych portach
    - Druga uruchamiana na 8006 a connector z portu 8081 przekierowywał na 8443
    - Na pierwszej uruchom WAR z lokacji `/opt/tomcat8/webapps/docs/appdev/sample/sample.war`

```sh
$ puppet module install puppetlabs/java
$ puppet module install puppetlabs/tomcat
$ cat /etc/puppet/manifests/tomcat.pp
```

```puppet
class { 'java': }

tomcat::install { '/opt/tomcat8':
	source_url => 'https://www.apache.org/dist/tomcat/tomcat-8/v8.0.33/bin/apache-tomcat-8.0.33.tar.gz'
}

tomcat::instance { 'tomcat8-first':
	catalina_home => '/opt/tomcat8',
	catalina_base => '/opt/tomcat8/first',
}

tomcat::instance { 'tomcat8-second':
	catalina_home => '/opt/tomcat8',
	catalina_base => '/opt/tomcat8/second',
}

# Change the default port of the second instance server and HTTP connector
tomcat::config::server { 'tomcat8-second':
	catalina_base => '/opt/tomcat8/second',
	port          => '8006',
}

tomcat::config::server::connector { 'tomcat8-second-http':
	catalina_base         => '/opt/tomcat8/second',
	port                  => '8081',
	protocol              => 'HTTP/1.1',
	additional_attributes => {
		'redirectPort' => '8443'
	},
}
```