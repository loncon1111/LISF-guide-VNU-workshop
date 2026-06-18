# Running LISF v7.5.4 via Ubuntu in Docker on macOS (Apple Silicon)

A complete step-by-step guide based on what we worked through in VNU Workshop. Aimed at config development and small-domain testing
## Prerequisites

* Docker Desktop installed on macOS, version 4.25 or newer
* ~10 GB free disk space (Docker VM + extracted AppImage + dependencies)
* For Apple Silicon: enable **Settings → General → "Use Rosetta for x86_64/amd64 emulation", then Apply & Restart**

## Step 1 - Have the LISF AppImage
On macOS, grab the v7.5.4 release that bundles all three components (LDT/LIS/LVT) in one AppImage, having the name of LISF-x86_64.AppImage from:
https://github.com/NASA-LIS/LISF/releases/download/v7.5.4-public/LISF-x86_64.AppImage


### Note: For regular Ubuntu (or other Linux platforms):
```bash
chmod 755 ./LISF-x86_64.AppImage
```

```bash
mkdir LISF-x86_64
cd LISF-x86_64
../LISF-x86_64.AppImage --appimage-extract
```
```bash
export LISF_ROOT=${PWD}
```
### Everything is different for Mac

## Step 2 - Launch an Ubuntu container (docker)

```bash
cd ~/Downloads
docker run --platform=linux/amd64 -it \
  -v "$PWD":/work -w /work \
  --name lisf ubuntu:22.04 bash
```

Flag-by-flag:

* ```--platform=linux/amd64``` - forces x86_64 emulation (LISF is x86_64-only)
* ```-v "$PWD":/work``` - mounts your Downloads folder into the container at /work
* ```--name lisf``` - names the container so you can restart it later
ubuntu:22.04 - glibc 2.35, comfortably above LISF's 2.22 floor

## Step 3 - Extract the AppImage payload (could be run on regular Ubuntu also if LISF_x86_64.AppImage extracted fail)

Need to be run on **root**

```bash
# Inside the container
apt-get update && apt-get install -y squashfs-tools grep

# Find the real squashfs offset (there may be false 'hsqs' matches in the ELF launcher)
for off in $(grep -abo 'hsqs' /work/LISF-x86_64.AppImage | cut -d: -f1); do
  if unsquashfs -s -o "$off" /work/LISF-x86_64.AppImage 2>&1 | grep -q "valid SQUASHFS"; then
    SIZE=$(unsquashfs -s -o "$off" /work/LISF-x86_64.AppImage 2>&1 | awk '/Filesystem size/ {print $3}')
    echo "candidate offset: $off  filesystem size: $SIZE bytes"
  fi
done
```

Pick the offset whose ```Filesystem``` size is close to the AppImage's total size (~452 million bytes for v7.5.4). Then extract into the container's own filesystem:

```bash
export LISF_ROOT=/root/squashfs-root
unsquashfs -d /root/squashfs-root -o <OFFSET> /work/LISF-x86_64.AppImage
ls /root/squashfs-root/usr/bin/    # should list LDT, LIS, LVT
```
**Ex:** 

## Step 4 - Install dependencies (can happen at regular Ubuntu virtual too)
If you don't encounter problems like
```bash
 error while loading shared libraries: libxxx.xx.x: cannot open shared object file: No such file or directory
```
it's good to go, but if you ain't lucky enough, then make sure all these libs are available

```bash
apt-get install -y \
  libjpeg-turbo8 \
  libgfortran5 \
  libgomp1 \
  zlib1g \
  libcurl4 \
  libxml2 \
  libxext6 \
  libxrender1 \
  libice6 \
  libsm6 \
  libfontconfig1
```
If any binary still complains about a specific ```.so```, this is the diagnostic loop:

```bash
# Find which package provides a missing library
apt-get install -y apt-file && apt-file update
apt-file search libfoo.so.N
# then apt-get install -y <package>
```
Or use `ldd` against the AppRun-prepared environment to see all unresolved libs at once:
```bash
# pull the exact LD_LIBRARY_PATH AppRun would set:
LIBPATH=$(bash -c 'source <(grep -E "^(export )?LD_LIBRARY_PATH|^LISF_LIBS_" /root/squashfs-root/AppRun); echo $LD_LIBRARY_PATH')
LD_LIBRARY_PATH="$LIBPATH" ldd /root/squashfs-root/usr/bin/LIS | grep "not found"
```
That prints every missing ```.so``` in one go, so you can ```apt-get``` install them together rather than chasing one error per run.

***NOTE 1: Specify libtiff.so.5***
Sometimes, it says there is no such thing call libtiff5, then
```bash
apt-get install -y libtiff6
```
then rename ```libtiff.so.6``` with ```libtiff.so.5```
```bash
cd /usr/lib/x86_64-linux-gnu/
ln -fs ./libtiff.so.6 ./libtiff.so.5
cd $LISF_ROOT
```
***NOTE 2: Specify libmpifort.so.12***
Install MPICH
```bash
apt-get install -y mpich
```
Ubuntu 22.04 ships MPICH 4.0, which keeps the same libmpi*.so.12 SONAMEs as 3.4 and is ABI-compatible for the interfaces ESMF uses, so this works without rebuilding anything.
If you are still not that lucky (Ubuntu version is too new), then we shall make a symlink to rename 
```bash
cd /usr/lib/x86_64-linux-gnu/
ln -fs ./libmpichfort.so.12 ./libmpifort.so.12
ln -fs ./libmpichxx.so.12 ./libmpixx.so.12
ln -fs ./libmpichcc.so.12 ./libmpicc.so.12
cd $LISF_ROOT
```

## Step 5 - Run the smoke tests

## Step 6 - Commit the container so you never redo this:
```bash
# from a separate macOS terminal, while the container is running
docker ps                              # grab the container ID
docker commit <container_id> lisf:7.5.4
```
Then future sessions
```bash
docker run --platform=linux/amd64 -it \
  -v "$PWD":/work -w /work --name lisf lisf:7.5.4 bash
```
