# Chef — Practical Recipes

Commands and recipes from Chef-Puppet-Ansible slides 8–20.

## 1. Install Chef Workstation on Ubuntu

1. Visit [Chef downloads](https://www.chef.io/downloads).
2. Open the Community / open-source download section and select Chef Workstation for Ubuntu AMD64.
3. After downloading the `.deb` package:

```bash
cd ~/Downloads
sudo dpkg -i chef-workstation-*.deb
sudo apt-get -f install
chef --version
cookstyle --version
```

## 2. Create a user with a Chef recipe

```bash
sudo mkdir -p /var/tmp/demorecipes
sudo chown "$USER":"$USER" /var/tmp/demorecipes
cd /var/tmp/demorecipes
nano demouser.rb
```

`demouser.rb`:

```ruby
user 'ditiss' do
  shell '/bin/bash'
  uid '9999'
  action :create
end
```

Check whether the user exists and lint the recipe:

```bash
id ditiss
cookstyle demouser.rb
```

Dry run:

```bash
sudo chef-client --local-mode demouser.rb --why-run
```

Apply:

```bash
sudo chef-client --local-mode demouser.rb
id ditiss
grep -i ditiss /etc/passwd
```

Run the recipe again to check that it does not recreate an existing user:

```bash
sudo chef-client --local-mode demouser.rb
```

Delete and recreate the lab user:

```bash
sudo userdel -r ditiss
id ditiss
sudo chef-client --local-mode demouser.rb
id ditiss
```

## 3. Create a file with Chef

`demofile.rb`:

```ruby
file '/var/tmp/testfile' do
  owner 'ditiss'
  group 'ditiss'
  mode '0644'
  action :create
end
```

Validate, dry-run, apply, and verify:

```bash
cookstyle demofile.rb
sudo chef-client --local-mode demofile.rb --why-run
sudo chef-client --local-mode demofile.rb
ls -l /var/tmp/testfile
```

> The original slide uses mode `777`. That grants every local user read, write, and execute permission. `0644` is used here for a regular text file; use the source's `0777` only when a lab explicitly requires it.

## 4. Add and update file content

Replace the file resource in `demofile.rb` with:

```ruby
file '/var/tmp/testfile' do
  owner 'ditiss'
  group 'ditiss'
  mode '0644'
  content "Hello and Welcome to this DITISS Course on Chef\n"
  action :create
end
```

Run and verify:

```bash
cookstyle demofile.rb
sudo chef-client --local-mode demofile.rb --why-run
sudo chef-client --local-mode demofile.rb
cat /var/tmp/testfile
```

To update it, change the content line:

```ruby
content "Updated Content: Welcome to the advanced DevOps and Automation module!\n"
```

Then run:

```bash
cookstyle demofile.rb
sudo chef-client --local-mode demofile.rb --why-run
sudo chef-client --local-mode demofile.rb
cat /var/tmp/testfile
```

## 5. Delete the file with Chef

Replace the resource with:

```ruby
file '/var/tmp/testfile' do
  action :delete
end
```

Run and verify:

```bash
cookstyle demofile.rb
sudo chef-client --local-mode demofile.rb --why-run
sudo chef-client --local-mode demofile.rb
ls -l /var/tmp/testfile
```

## 6. Install the OpenSSH client using Chef

Check whether SSH is present:

```bash
which ssh
```

`demosshpkg.rb`:

```ruby
package 'openssh-client' do
  action :install
end
```

Validate, dry-run, and apply:

```bash
cookstyle demosshpkg.rb
sudo chef-client --local-mode demosshpkg.rb --why-run
sudo chef-client --local-mode demosshpkg.rb
which ssh
```

## 7. Create a directory

`directory_recipe.rb`:

```ruby
directory '/var/tmp/testdir' do
  mode '0755'
  owner 'ditiss'
  recursive true
  action :create
end
```

Run:

```bash
cookstyle directory_recipe.rb
sudo chef-client --local-mode directory_recipe.rb --why-run
sudo chef-client --local-mode directory_recipe.rb
ls -ld /var/tmp/testdir
```

## 8. Create a directory and a file in it

`multiconfig.rb`:

```ruby
directory '/var/tmp/newdir' do
  owner 'root'
  group 'root'
  mode '0755'
  action :create
end

file '/var/tmp/newdir/testfile' do
  mode '0644'
  content "Hello again\n"
  action :create
end
```

Run:

```bash
cookstyle multiconfig.rb
sudo chef-client --local-mode multiconfig.rb --why-run
sudo chef-client --local-mode multiconfig.rb
cat /var/tmp/newdir/testfile
```

## 9. Install and enable NGINX

`nginx.rb`:

```ruby
package 'nginx' do
  action :install
end

service 'nginx' do
  supports status: true, restart: true, reload: true
  action [:enable, :start]
end
```

Run:

```bash
cookstyle nginx.rb
sudo chef-client --local-mode nginx.rb --why-run
sudo chef-client --local-mode nginx.rb
systemctl status nginx
```

Open `http://localhost:80`.

Stop NGINX when finished:

```bash
sudo systemctl stop nginx
```

## 10. Install Apache and configure a custom page

Check whether Apache exists:

```bash
which apache2
```

`demoapachepkg.rb`:

```ruby
package 'apache2' do
  action :install
end

file '/var/www/html/index.html' do
  content 'Welcome to DITISS'
  mode '0644'
  action :create
  notifies :restart, 'service[apache2]', :delayed
end

service 'apache2' do
  action [:enable, :start]
end
```

Run:

```bash
cookstyle demoapachepkg.rb
sudo chef-client --local-mode demoapachepkg.rb --why-run
sudo chef-client --local-mode demoapachepkg.rb
which apache2
curl http://localhost
```

If Apache cannot start because port 80 is already occupied:

```bash
sudo ss -tulpn | grep ':80'
sudo systemctl stop nginx
sudo systemctl start apache2
```

Source reference: Chef-Puppet-Ansible slides 8–20. Smart/curly quotes from the slide text have been normalized to standard Ruby quotes.
