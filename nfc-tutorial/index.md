# Setting up Ubuntu 22.04.04 (Jammy) to Utilize ACR122U

## Install Required Packages

1. Libraries and services

    1. Mandatory

        ```sh
        sudo apt-get install autoconf pkg-config
        sudo apt-get install pcscd pcsc-tools
        ```

        Blacklist conflicting services,  
        add this list into `/etc/modprobe.d/blacklist-libnfc.conf`:

        > blacklist nfc \
        > blacklist pn533 \
        > blacklist pn533_usb

        Remove the services (lots of forums recommend this):

        ```sh
        sudo modprobe -rf pn533_usb pn533 nfc
        ```

    2. Install `libnfc` (you may skip this if you're not using `mfcuk` or `mfoc`)

        ```sh
        sudo add-apt-repository universe
        sudo apt-get install libnfc-dev libnfc-bin libnfc-examples
        ```

2. ACR122U driver

    1. Download link in [ACR122U official website](https://www.acs.com.hk/en/driver/3/acr122u-usb-nfc-reader/) (ACR122U is EOL already by 2022)

    2. Unzip the package and install (this was the file name by April 2024)

        ```sh
        unzip ./ACS-Unified-PKG-Lnx-118-P.zip
        sudo dpkg -i ./ACS-Unified-PKG-Lnx-118-P/ubuntu/eoan/libacsccid1_1.1.8-1~ubuntu18.04.1_amd64.deb
        ```

## Plug and Start the Service

Plug your ACR122U, then restart `pcscd` (NFC reader service)

```sh
sudo service pcscd restart
```

## Running Downstream Application

### Use pcsc_scan

```sh
pcsc_scan
```

### Use Wakdev's NFC Tools

1. Download the .AppImage [here](https://www.wakdev.com/en/apps/nfc-tools-pc-mac.html)

2. Add execution permission and run:

    ```sh
    sudo chmod+x nfctools-linux-latest.AppImage
    ./nfctools-linux-latest.AppImage
    ```

### Use mfcuk and mfoc

#### mfcuk

Download all `.mfd` files inside [mfcuk_fix/data/tmpls_fingerprints/](https://github.com/S0c5/mfcuk_fix/tree/master/data/tmpls_fingerprints), then put under "./mfcuk/src/data/tmplsfingerprints/"

```sh
git clone https://github.com/nfc-tools/mfcuk.git
cd mfcuk
autoreconf -is
./configure
make

cd src
./mfcuk -C -R 0:A -v 2
```

#### mfoc

```sh
git clone https://github.com/nfc-tools/mfoc.git
cd mfoc
autoreconf -is
./configure
make

cd src
./mfoc -O source_dump.mfd -k {the key A you retrieved}
```

#### Debugging

If either mfcuk or mfoc returns error on `./configure`, try executing this commands ([ref](https://github.com/nfc-tools/mfcuk/issues/28)):

```sh
sudo libtoolize
sudo aclocal
sudo autoconf
sudo autoheader
sudo automake --add-missing
sudo automake
sudo autoreconf -vis
```

## Appendix, Additional Commands that May be Useful

1. Get Linux version

    ```sh
    lsb_release -a
    ```

2. Upgrading the outdated MariaDB guide

    - [NextCloud Forum](https://help.nextcloud.com/t/apt-update-fails-because-of-mariadb-repo/175500)
    - [Upgrading MariaDB repo & version](https://mariadb.org/download/?t=repo-config&d=22.04+%22jammy%22&v=10.11&r_m=nus)

3. Find installed packages

    ```sh
    dpkg -l | grep <package_name>
    ```

4. List all repositories

    ```sh
    grep ^ /etc/apt/sources.list /etc/apt/sources.list.d/*
    ```

5. Check all available NFC devices (if you install `libnfc`)

    ```sh
    nfc-list
    ```

## Guide Author

Ferdiant Joshua Muis
