# Apptainer — Install, Run, and Build Containers

Commands from Virtualization, Cloud, and AWS slides 63–66.

## 1. Install build dependencies on Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  libseccomp-dev \
  pkg-config \
  uidmap \
  squashfs-tools \
  cryptsetup \
  curl \
  wget \
  git
```

For RHEL/CentOS, the source slide lists:

```bash
sudo yum groupinstall -y "Development Tools"
sudo yum install -y libseccomp-devel wget git squashfs-tools cryptsetup
```

## 2. Install Go 1.22.0

```bash
cd ~/Downloads
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz

echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
go version
```

## 3. Download and build Apptainer

```bash
cd ~
git clone https://github.com/apptainer/apptainer.git
cd apptainer
git checkout v1.3.0

./mconfig
make -C builddir
sudo make -C builddir install

apptainer --version
```

## 4. Run an Alpine container

```bash
sudo apptainer run docker://alpine
```

Inside the container:

```bash
cat /etc/os-release
ls -la
exit
```

Execute a command without opening an interactive shell:

```bash
sudo apptainer exec docker://alpine echo "Hello from container"
```

Run Python inside a container:

```bash
sudo apptainer exec docker://python:3.9 \
  python -c "print('Hello Python')"
```

## 5. Build a custom `.sif` image

Create `hello.def`:

```bash
cat <<'DEF_FILE' > hello.def
Bootstrap: docker
From: alpine

%runscript
    echo "Hello, World from inside Apptainer!"
DEF_FILE
```

Build the image:

```bash
sudo apptainer build hello.sif hello.def
```

Run it:

```bash
sudo ./hello.sif
# Or:
sudo apptainer run hello.sif
```

Source reference: Virtualization, Cloud, and AWS slides 63–66.
