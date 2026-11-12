# Puppet — Practical Manifests

Installation and manifests from Chef-Puppet-Ansible slides 22–35.

## 1. Install Puppet Server and Agent on Ubuntu

The repository package shown in the slides is the Puppet 8 **Focal** package (Ubuntu 20.04). Use it only on the matching OS release.

```bash
wget https://apt.puppet.com/puppet8-release-focal.deb
sudo dpkg -i puppet8-release-focal.deb
sudo apt-get update

sudo apt-get install -y puppetserver
sudo systemctl start puppetserver
exec bash -l
puppetserver --version

sudo apt-get install -y puppet-agent
sudo /opt/puppetlabs/bin/puppet resource service puppet ensure=running enable=true
```

Add Puppet to the current shell `PATH` if `puppet` is not found:

```bash
export PATH=/opt/puppetlabs/bin:$PATH
```

Optional persistent PATH setup:

```bash
echo 'export PATH=/opt/puppetlabs/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 2. Create a user

```bash
mkdir -p /var/tmp/demorecipes
cd /var/tmp/demorecipes
nano puppetuser.pp
```

`puppetuser.pp`:

```puppet
user { 'ditissuser':
  ensure => present,
}
```

Validate, dry-run, apply, and verify:

```bash
puppet parser validate puppetuser.pp
puppet apply puppetuser.pp --noop
sudo puppet apply puppetuser.pp
id ditissuser
```

Repeat the apply to verify that the user remains present:

```bash
sudo puppet apply puppetuser.pp
id ditissuser
```

Delete and recreate the user:

```bash
sudo userdel ditissuser
id ditissuser
sudo puppet apply puppetuser.pp
id ditissuser
```

## 3. Delete users

`delete_users.pp`:

```puppet
user { 'ditiss':
  ensure => absent,
}

user { 'ditissuser':
  ensure => absent,
}
```

Run:

```bash
puppet parser validate delete_users.pp
puppet apply delete_users.pp --noop
sudo puppet apply delete_users.pp
```

## 4. Change user properties

Replace `puppetuser.pp` with:

```puppet
user { 'ditissuser':
  ensure => present,
  uid    => '7777',
  shell  => '/bin/sh',
}
```

Run:

```bash
puppet parser validate puppetuser.pp
puppet apply puppetuser.pp --noop
sudo puppet apply puppetuser.pp
grep -i ditissuser /etc/passwd
```

## 5. Create a group

`puppetgroup.pp`:

```puppet
group { 'ditiss':
  ensure => present,
}
```

Run:

```bash
puppet parser validate puppetgroup.pp
puppet apply puppetgroup.pp --noop
sudo puppet apply puppetgroup.pp
grep '^ditiss:' /etc/group
```

## 6. Create a file with ownership and permissions

`demofile.pp`:

```puppet
file { '/var/tmp/testfile':
  ensure => present,
  owner  => 'ditissuser',
  group  => 'ditiss',
  mode   => '0644',
}
```

Run:

```bash
puppet parser validate demofile.pp
puppet apply demofile.pp --noop
sudo puppet apply demofile.pp
ls -l /var/tmp/testfile
```

Add content by changing the manifest to:

```puppet
file { '/var/tmp/testfile':
  ensure  => present,
  owner   => 'ditissuser',
  group   => 'ditiss',
  mode    => '0644',
  content => "Hello and welcome to C-DAC DITISS\n",
}
```

Run and verify:

```bash
puppet parser validate demofile.pp
puppet apply demofile.pp --noop
sudo puppet apply demofile.pp
cat /var/tmp/testfile
```

> The slide's file example uses mode `0777`; that gives all local users write access. The example uses `0644` for a normal text file. Create the `ditiss` group before applying a manifest that sets it as the file's group.

## 7. Install and run Apache

`apache.pp`:

```puppet
package { 'apache2':
  ensure => installed,
}

service { 'apache2':
  ensure  => running,
  enable  => true,
  require => Package['apache2'],
}

file { '/var/www/html':
  ensure => directory,
  owner  => 'root',
  group  => 'www-data',
  mode   => '0755',
}

file { '/var/www/html/index.html':
  ensure  => file,
  content => '<html><body><h1>Welcome to Apache!</h1></body></html>',
  owner   => 'root',
  group   => 'www-data',
  mode    => '0644',
  require => File['/var/www/html'],
}
```

Run:

```bash
puppet parser validate apache.pp
puppet apply apache.pp --noop
sudo puppet apply apache.pp
curl http://localhost
```

If the command is not found, use the full executable path:

```bash
sudo /opt/puppetlabs/bin/puppet apply apache.pp
```

## 8. Start the NFS service

Check service status:

```bash
systemctl status nfs
```

`demoservice.pp`:

```puppet
service { 'nfs':
  ensure => running,
}
```

Run:

```bash
puppet parser validate demoservice.pp
puppet apply demoservice.pp --noop
sudo puppet apply demoservice.pp
systemctl status nfs
```

The source slide does not include an NFS package-install manifest. Install the appropriate NFS server package for your Ubuntu release before applying this service resource if it is not already installed.

## 9. Create multiple resources

`demomultiresource.pp`:

```puppet
file { '/var/tmp/testdir':
  ensure => directory,
}

file { '/var/tmp/testdir/testfile':
  ensure => present,
}
```

Run:

```bash
puppet parser validate demomultiresource.pp
puppet apply demomultiresource.pp --noop
sudo puppet apply demomultiresource.pp
ls -l /var/tmp/testdir
```

Source reference: Chef-Puppet-Ansible slides 22–35. The source uses both `/var/tmp` and `/var/temp` in different places; this guide consistently uses `/var/tmp`.
