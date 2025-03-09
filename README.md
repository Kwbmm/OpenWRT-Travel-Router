# OpenWRT-Travel-Router
This repository contains a step-by-step guide on how to setup a Raspberry Pi 3 with 2 WiFi cards. One card connects to the desired WiFi, the other hosts a WiFI network for clients to join.

# Initial Setup

Once OpenWRT is flashed on the SD card and the RPI is started, it require direct connection to your computer through the LAN cable. This is because OWRT is setup to create a network on the ethernet port with IP 192.168.1.1 : this IP and network are likely used by your existing router, so you won't be able to connect to it if you plug the ethernet cable into the PI and the existing router.

To fix this, you have 2 options:

  - Connect the RPI to your laptop via ethernet
  - Connect a screen to the RPI and change some configs

We'll go for the second one.

## Fixing the ethernet

Log into the console by connecting keyboard and a screen to the Pi, then modify the lan config into `/etc/config/network` as follows:

```
config interface 'loopback'
	option device 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config globals 'globals'
	option ula_prefix 'fd9b:9d35:ae65::/48'
	option packet_steering '1'

config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'eth0'

config interface 'lan'
	option device 'br-lan'
	option proto 'dhcp'
```

Then make sure SSH is enabled, by checking `/etc/config/dropbear` config:

```
config dropbear main
        option enable '1'
        option PasswordAuth 'on'
        option RootPasswordAuth 'on'
        option Port         '22'
#       option BannerFile   '/etc/banner'
```

Once you reboot, the PI should be able to join your existing network. You should be able to SSH into it with `root@<ip_addr>`

# Travel Router Setup

This setup comes from ChatGPT and the following question I asked

> I have a raspberry pi with OpenWRT installed. The RPI has 2 wifi cards: the built-in one (radio0) and an external one (radio1).
It is also connected to my home router through the LAN port, with IP 192.168.1.193 .
>
> I can currently access the web interface through it's LAN IP 192.168.1.3 .
>
> I want to setup OpenWRT such that:
>  - There is a wifi network provided by radio1 that has a gateway set on 10.71.71.1 and all other clients can connect to it and get a DHCP address in the range 10.71.71.50 - 10.71.71.100 . The wifi should be called 'Travel' .
>  - I want to be able to access the web interface to change the configuration from the network just created. So going to 10.71.71.1 should open the web interface of openwrt.

**Chat GPT answer**

To set up your Raspberry Pi with OpenWRT to create a Wi-Fi network using `radio1` with the specified configurations, follow these steps:

### Step 1: Access OpenWRT Web Interface
1. Open your web browser and go to `http://192.168.1.193` to access the OpenWRT web interface.

### Step 2: Configure `radio1` for the 'Travel' Wi-Fi Network
1. **Navigate to Network > Wireless**.
2. Find the `radio1` interface (the external Wi-Fi card) and click on **Edit**.
3. Set the following parameters:
   - **ESSID**: `Travel`
   - **Mode**: Access Point
   - **Network**: Create a new network (e.g., `Travel`).

4. **Save & Apply** the changes.

### Step 3: Create a New Network for the 'Travel' Wi-Fi
1. **Navigate to Network > Interfaces**.
2. Click on **Add new interface**.
3. Set the following parameters:
   - **Name**: `Travel`
   - **Protocol**: Static address
   - **Interface**: Select the new network you created for `radio1`.

4. Click **Submit**.

### Step 4: Configure the Static IP Address
1. Set the following parameters:
   - **IPv4 address**: `10.71.71.1`
   - **IPv4 netmask**: `255.255.255.0`
   - **IPv4 gateway**: Leave this blank (as this will be the gateway).
   - **DNS server**: You can set this to your home router's IP (e.g., `192.168.1.1`) or any public DNS server (e.g., `8.8.8.8`).

2. **Save & Apply** the changes.

### Step 5: Configure DHCP for the 'Travel' Network
1. **Navigate to Network > DHCP and DNS**.
2. Scroll down to the **DHCP Server** section and find the `Travel` interface.
3. Enable the DHCP server and set the following parameters:
   - **Start**: `50`
   - **Limit**: `51` (this will give you a range of 50 addresses, from 10.71.71.50 to 10.71.71.100)
   - **Lease time**: Set as desired (e.g., `12 hours`).

4. **Save & Apply** the changes.

### Step 6: Configure Firewall
1. **Navigate to Network > Firewall**.
2. Click on **Add** to create a new zone for the `Travel` network.
3. Set the following parameters:
   - **Name**: `Travel`
   - **Input**: Accept
   - **Output**: Accept
   - **Forward**: Reject
   - **Covered networks**: Select the `Travel` interface.

4. **Save & Apply** the changes.

### Step 7: Allow Access to the Web Interface
1. In the same Firewall settings, go to the **Traffic Rules** section.
2. Click on **Add** to create a new rule.
3. Set the following parameters:
   - **Name**: Allow access to web interface
   - **Source zone**: `Travel`
   - **Destination zone**: `Device`
   - **Destination port**: `80` (for HTTP) and `443` (for HTTPS, if enabled).
   - **Action**: Accept

4. **Save & Apply** the changes.

### Step 8: Finalize and Test
1. **Reboot** your Raspberry Pi to ensure all settings are applied correctly.
2. Connect a device to the `Travel` Wi-Fi network.
3. Check if you can access the OpenWRT web interface by navigating to `http://10.71.71.1`.

### Troubleshooting
- If you cannot access the web interface, double-check the firewall rules and ensure that the DHCP server is functioning correctly.
- Ensure that the Wi-Fi network is broadcasting and that devices can connect to it.

By following these steps, you should have a functioning Wi-Fi network on your Raspberry Pi with OpenWRT, allowing you to access the web interface from the new network.

---

Then I asked

> Now, I want radio0 to connect to a wifi network and forward internet connection to the clients connected the Travel network thorugh radio1

**Chat GPT answer**

To configure your Raspberry Pi with OpenWRT so that `radio0` connects to an existing Wi-Fi network and forwards the internet connection to clients connected to the `Travel` network via `radio1`, follow these steps:

### Step 1: Connect `radio0` to the Existing Wi-Fi Network

1. **Navigate to Network > Wireless**.
2. Find the `radio0` interface (the built-in Wi-Fi card) and click on **Edit**.
3. Set the following parameters:
   - **Mode**: Client
   - **ESSID**: Enter the SSID of the Wi-Fi network you want to connect to.
   - **Network**: Create a new network (e.g., `WAN`).

4. **Save & Apply** the changes.

### Step 2: Configure the WAN Interface

1. **Navigate to Network > Interfaces**.
2. Click on **Add new interface**.
3. Set the following parameters:
   - **Name**: `WAN`
   - **Protocol**: DHCP client (if the existing Wi-Fi network provides DHCP).
   - **Interface**: Select the new network you created for `radio0`.

4. **Save & Apply** the changes.

### Step 3: Configure Firewall Zones

1. **Navigate to Network > Firewall**.
2. You should see the existing zones for `Travel` and `WAN`. If not, create a new zone for the `WAN` interface:
   - Click on **Add**.
   - Set the following parameters:
     - **Name**: `WAN`
     - **Input**: Reject
     - **Output**: Accept
     - **Forward**: Reject
     - **Covered networks**: Select the `WAN` interface.

3. **Save & Apply** the changes.

### Step 4: Allow Forwarding from `Travel` to `WAN`

1. In the Firewall settings, go to the **Traffic Rules** section.
2. Click on **Add** to create a new rule.
3. Set the following parameters:
   - **Name**: Allow forwarding from Travel to WAN
   - **Source zone**: `Travel`
   - **Destination zone**: `WAN`
   - **Action**: Accept

4. **Save & Apply** the changes.

### Step 5: Configure NAT (Network Address Translation)

1. Still in the Firewall settings, go to the **General Settings** tab.
2. Find the `WAN` zone and check the box for **Masquerading**. This will enable NAT for the `WAN` zone, allowing clients on the `Travel` network to access the internet.

3. **Save & Apply** the changes.

### Step 6: Test the Configuration

1. **Reboot** your Raspberry Pi to ensure all settings are applied correctly.
2. Connect a device to the `Travel` Wi-Fi network.
3. Check if you can access the internet from the device connected to the `Travel` network.

### Additional Notes

- If you encounter any issues, check the status of the `WAN` interface to ensure it is connected to the existing Wi-Fi network.
- You can also check the logs in the OpenWRT web interface under **Status > System Log** for any errors or connection issues.
- Make sure that the existing Wi-Fi network allows clients to connect and provides internet access.

If you have any questions or need further assistance, feel free to ask!

---

Once this is done, make sure to unplug the ethernet cable (you won't be able to go to the internet otherwise). Then reboot


# TODO

  - AdGuard setup
  - Wifi cards used + driver install
  - Resizing of disk (if necessary)
