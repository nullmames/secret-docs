# Setup SGX

## Prepare your Hardware <a href="#prepare-your-hardware" id="prepare-your-hardware"></a>

If you're running a local machine and not a cloud-based VM -

1. Go to your BIOS menu
2. Enable SGX (Software controlled is not enough)
3. Disable Secure Boot
4. Disbale HyperV

## [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#installation)Installation <a href="#installation" id="installation"></a>

### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#for-node-runners)For Node Runners <a href="#for-node-runners" id="for-node-runners"></a>

#### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#install-sgx)Install SGX <a href="#install-sgx" id="install-sgx"></a>

Note: `sgx_linux_x64_driver_2.6.0_602374c.bin` is the latest driver as of July 13, 2020. Please check under https://download.01.org/intel-sgx/sgx-linux/ that this is still the case. If not, please send us a PR or notify us.

```
#! /bin/bash

UBUNTUVERSION=$(lsb_release -r -s | cut -d '.' -f 1)
PSW_PACKAGES='libsgx-enclave-common libsgx-urts sgx-aesm-service libsgx-uae-service autoconf libtool make'

if (($UBUNTUVERSION < 16)); then
	echo "Your version of Ubuntu is not supported. Must have Ubuntu 16.04 and up. Aborting installation script..."
	exit 1
elif (($UBUNTUVERSION < 18)); then
	DISTRO='xenial'
	OS='ubuntu16.04-server'
else
	DISTRO='bionic'
	OS='ubuntu18.04-server'
fi

echo "\n\n###############################################"
echo "#####       Installing Intel SGX driver       #####"
echo "###############################################\n\n"

# download SGX driver
wget "https://download.01.org/intel-sgx/sgx-linux/2.10/distro/${OS}/sgx_linux_x64_driver_2.6.0_602374c.bin"

# Make the driver installer executable
chmod +x ./sgx_linux_x64_driver_*.bin

# Install the driver
sudo ./sgx_linux_x64_driver_*.bin

# Remount /dev as exec, also at system startup
sudo tee /etc/systemd/system/remount-dev-exec.service >/dev/null <<EOF
[Unit]
Description=Remount /dev as exec to allow AESM service to boot and load enclaves into SGX

[Service]
Type=oneshot
ExecStart=/bin/mount -o remount,exec /dev
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable remount-dev-exec
sudo systemctl start remount-dev-exec

echo "\n\n###############################################"
echo "#####       Installing Intel SGX PSW          #####"
echo "###############################################\n\n"

# Add Intels's SGX PPA
echo "deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu $DISTRO main" |
   sudo tee /etc/apt/sources.list.d/intel-sgx.list
wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key |
   sudo apt-key add -
sudo apt update

# Install libprotobuf
if (($UBUNTUVERSION > 18)); then
   sudo apt install -y gdebi
   # Install all the additional necessary dependencies (besides the driver and the SDK)
   # for building a rust enclave
   wget -O /tmp/libprotobuf10_3.0.0-9_amd64.deb http://ftp.br.debian.org/debian/pool/main/p/protobuf/libprotobuf10_3.0.0-9_amd64.deb
   yes | sudo gdebi /tmp/libprotobuf10_3.0.0-9_amd64.deb
else
   PSW_PACKAGES+=' libprotobuf-dev'
fi

sudo apt install -y $PSW_PACKAGES
```

### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#for-enclave-developers)For Enclave Developers <a href="#for-enclave-developers" id="for-enclave-developers"></a>

#### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#prerequisites)Prerequisites <a href="#prerequisites" id="prerequisites"></a>

First, make sure you have Rust installed: https://www.rust-lang.org/tools/install

*   Once Rust is installed, install the `nightly` toolchain:

    ```
    rustup toolchain install nightly
    ```

Then you can use this script (or run the commands one-by-one), which was tested on Ubuntu 20.04 with SGX driver/sdk version 2.10 intended for Ubuntu 18.04:

#### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#install-sgx-sdk-driver)Install SGX SDK + Driver <a href="#install-sgx-sdk-driver" id="install-sgx-sdk-driver"></a>

```
#! /bin/bash

UBUNTUVERSION=$(lsb_release -r -s | cut -d '.' -f 1)

if (($UBUNTUVERSION < 16)); then
	echo "Your version of Ubuntu is not supported. Must have Ubuntu 16.04 and up. Aborting installation script..."
	exit 1
elif (($UBUNTUVERSION < 18)); then
	DISTRO='xenial'
else
	DISTRO='bionic'
fi

echo "\n\n#######################################"
echo "##### Installing missing packages #####"
echo "#######################################\n\n"

# Install needed packages for script
sudo apt install -y lynx parallel gdebi make

# Create a working directory to download and install the SDK inside
mkdir -p "$HOME/.sgxsdk"

(
   # In a new sub-shell cd into our working directory so to no pollute the
   # original shell's working directory
   cd "$HOME/.sgxsdk"

   echo "\n\n################################################"
   echo "##### Downloading Intel SGX driver and SDK #####"
   echo "################################################\n\n"

   # 1. Go to https://download.01.org/intel-sgx/sgx-linux
   # 2. Step into the latest version
   # 3. Step into `distro/$LATEST_UBUNTU_YOU_SEE_THERE`
   # 4. Download `sgx_linux_x64_driver_*.bin` and `sgx_linux_x64_sdk_*.bin`
   lynx -dump -listonly -nonumbers https://download.01.org/intel-sgx/sgx-linux/ |
      grep -P 'sgx-linux/(\d\.?)+/' |
      sort -V |
      tail -1 |
      parallel --bar --verbose lynx -dump -listonly -nonumbers "{}/distro" |
      grep -P 'ubuntu\d\d' |
      sort -V |
      tail -1 |
      parallel --bar --verbose lynx -dump -listonly -nonumbers |
      grep -P '\.bin$' |
      parallel --bar --verbose curl -OSs

   # Make the driver and SDK installers executable
   chmod +x ./sgx_linux_*.bin

   echo "\n\n###############################################"
   echo "##### Installing Intel SGX driver and SDK #####"
   echo "###############################################\n\n"

   # Install the driver
   sudo ./sgx_linux_x64_driver_*.bin

   # Remount /dev as exec, also at system startup
   sudo tee /etc/systemd/system/remount-dev-exec.service >/dev/null <<EOF
[Unit]
Description=Remount /dev as exec to allow AESM service to boot and load enclaves into SGX

[Service]
Type=oneshot
ExecStart=/bin/mount -o remount,exec /dev
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
EOF
   sudo systemctl enable remount-dev-exec
   sudo systemctl start remount-dev-exec

   # Install the SDK inside ./sgxsdk/ which is inside $HOME/.sgxsdk
   echo yes | ./sgx_linux_x64_sdk_*.bin

   # Setup the environment variables for every new shell
   echo "source '$HOME/.sgxsdk/sgxsdk/environment'" |
      tee -a "$HOME/.bashrc" "$HOME/.zshrc" > /dev/null
)

echo "\n\n##############################################"
echo "##### Installing additional dependencies #####"
echo "##############################################\n\n"

# Add Intels's SGX PPA
echo "deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu $DISTRO main" |
   sudo tee /etc/apt/sources.list.d/intel-sgx.list
wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key |
   sudo apt-key add -
sudo apt update

# Install all the additional necessary dependencies (besides the driver and the SDK)
# for building a rust enclave
wget -O /tmp/libprotobuf10_3.0.0-9_amd64.deb http://ftp.br.debian.org/debian/pool/main/p/protobuf/libprotobuf10_3.0.0-9_amd64.deb
(sleep 3 ; echo y) | sudo gdebi /tmp/libprotobuf10_3.0.0-9_amd64.deb

sudo apt install -y libsgx-enclave-common libsgx-enclave-common-dev libsgx-urts sgx-aesm-service libsgx-uae-service libsgx-launch libsgx-aesm-launch-plugin libsgx-ae-le autoconf libtool
```

Note that sometimes after a system reboot you'll need to reinstall the driver (usually after a kernel upgrade):

```
sudo $HOME/.sgxsdk/sgx_linux_x64_driver_*.bin
```

## [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#testing-your-sgx-setup)Testing your SGX setup <a href="#testing-your-sgx-setup" id="testing-your-sgx-setup"></a>

### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#for-node-runners-2)For Node Runners <a href="#for-node-runners-2" id="for-node-runners-2"></a>

#### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#run-secretd-init-enclave)Run `secretd init-enclave` <a href="#run-secretd-init-enclave" id="run-secretd-init-enclave"></a>

See https://github.com/scrtlabs/SecretNetwork/blob/master/docs/testnet/verify-sgx.md for a guide how to test your setup

### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#for-contract-developers)For Contract Developers <a href="#for-contract-developers" id="for-contract-developers"></a>

#### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#using-sgx-detect)using `sgx-detect`: <a href="#using-sgx-detect" id="using-sgx-detect"></a>

First, make sure you have Rust installed: https://www.rust-lang.org/tools/install

* Once Rust is installed, install the `nightly` toolchain:

```
rustup toolchain install nightly
```

```
sudo apt install -y libssl-dev protobuf-compiler
cargo +nightly install fortanix-sgx-tools sgxs-tools

sgx-detect
```

Should print at the end:

```
✔  Able to launch enclaves
   ✔  Debug mode
   ✔  Production mode (Intel whitelisted)

You're all set to start running SGX programs!
```

#### [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#compiling-a-hello-rust-project)Compiling a `hello-rust` project: <a href="#compiling-a-hello-rust-project" id="compiling-a-hello-rust-project"></a>

```
git clone --depth 1 -b v1.1.2 git@github.com:apache/incubator-teaclave-sgx-sdk.git

cd incubator-teaclave-sgx-sdk/samplecode/hello-rust
perl -i -pe 's/SGX_SDK \?=.+/SGX_SDK ?= \$(HOME)\/.sgxsdk\/sgxsdk/' Makefile
make
cd bin
./app
```

Should print somting similar to this:

```
[+] Init Enclave Successful 2!
This is a normal world string passed into Enclave!
This is a in-Enclave Rust string!
gd: 1 0 0 1
static: 1 eremove: 0 dyn: 0
EDMM: 0, feature: 9007268790009855
supported sgx
[+] say_something success...
```

## [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#uninstall)Uninstall <a href="#uninstall" id="uninstall"></a>

To uninstall the Intel(R) SGX Driver, run:

```
sudo /opt/intel/sgxdriver/uninstall.sh
```

The above command produces no output when it succeeds. If you want to verify that the driver has been uninstalled, you can run the following, which should print `SGX Driver NOT installed`:

```
ls /dev/isgx &>/dev/null && echo "SGX Driver installed" || echo "SGX Driver NOT installed"
```

To uninstall the SGX SDK, run:

```
sudo "$HOME"/.sgxsdk/sgxsdk/uninstall.sh
rm -rf "$HOME/.sgxsdk"
```

To uninstall the rest of the dependencies, run:

```
sudo apt purge -y libsgx-enclave-common libsgx-enclave-common-dev libsgx-urts sgx-aesm-service libsgx-uae-service libsgx-launch libsgx-aesm-launch-plugin libsgx-ae-le
```

## [#](https://docs.scrt.network/testnet/setup-sgx-testnet.html#refs)Refs <a href="#refs" id="refs"></a>

1. https://github.com/apache/incubator-teaclave-sgx-sdk/wiki/Environment-Setup
2. https://github.com/openenclave/openenclave/blob/master/docs/GettingStartedDocs/install\_oe\_sdk-Ubuntu\_18.04.md
3. https://github.com/apache/incubator-teaclave-sgx-sdk/blob/783f04c002e243d1022c5af8a982f9c2a7138f32/dockerfile/Dockerfile.1804.nightly
4. https://edp.fortanix.com/docs/installation/guide/

\
