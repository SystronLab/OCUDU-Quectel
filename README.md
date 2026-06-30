# OCUDU-Quectel

Guide to setup to setup Host (5g core , OCUDU, controller) with Robot/Client (Quectel 5g modem, robot). We are able to cotroller a mini robot over a private 5g network.

## Host

Run the 5g core

Run the gNB stack

Connect the controller to the Host

Build gamepad_interface.c with `make`

After the build is successful, run the udp client which sends the controller's commands to the udp server (running on robot) as JSON with

```bash
./gamepad_interface -p 9020 -i 10.45.0.2
```
If there is a joystick detection issue, run

```bash
ls /dev/input/by-id
```
verify that the event-joystic that shows up is the same in the code.

## Robot / Client

Insert the programmed SIM in the modem

Connect the Modem to the robot/client via usb

Install ModemManager and start it on Boot

```bash
sudo apt install modemmanager

sudo systemctl enable ModemManager
sudo systemctl start ModemManager
```

Wait for some time and List Modems. (You might have to keep trying till the modem shows up)

```bash
sudo mmcli -L
```
Check index number

Install Picocom

```bash
sudo apt install picocom
```

Identify AT Port

```bash
sudo mmcli -m 0 | grep ports
```
0 here is the index number identified above. You might have a different index number.

One of these is the AT port
```bash
ls /dev/ttyUSB*
```
Start AT Console
```bash
sudo picocom -b 115200 /dev/ttyUSB2
```

- Check AT connection:

  ```
  AT
  ```
- Check whether SIM is detected:

  ```
  AT+CPIN?
  ```
- Returns the IMSI:

  ```
  AT+CIMI
  ```
- Check if the modem sees the network (take a couple of mins):

  ```
  AT+COPS=?
  ```

- Returns APNs:
  ```
  AT+CGDCONT?
  ```

Example output:

```
+CGDCONT: 1,"IP","srsapn",...
+CGDCONT: 2,"IPV4V6","ims",...
+CGDCONT: 3,"IPV4V6","sos",...
```

> ⚠️ If you see multiple APNs (ims, sos), you'll get `DNN Not Supported OR Not Subscribed in the Slice` error in AMF logs. Remove extra APNs with:

```bash
AT+CGDCONT=2
AT+CGDCONT=3
```

Check the data interface

```bash
AT+QCFG="data_interface"
```

Set it to usb

```bash
AT+QCFG="data_interface",0,0
```

```bash
AT+QCFG="usbnet"

AT+QCFG="usbnet",2
```

Set APN with IPv4 Only

```bash
AT+CGDCONT=1,"IP","srsapn"
```

Attach to the Packet Domain

```bash
AT+CGATT=1
```

Activate PDP Context 1

```bash
AT+CGACT=1,1
```

Check if IP Address is Assigned

```bash
AT+CGPADDR=1
```

Ping Test

```bash
AT+QPING=1,"8.8.8.8"
```

Once the connection is successful and you see wwan0 as an interface, You're all set to run the script in the robot folder on boot.

If the wwan interface is still not visible then check if you have cdc-wdm and qmi_wwan modules using `modprobe`

If you want to test communication between the Host (gnb) and the receiver (UE). Use the following:

```bash
sudo mmcli -m "$MODEM_INDEX" -e
sudo mmcli -m "$MODEM_INDEX" --simple-connect="apn=srsapn"

# Assuming interface is wwan0 and IP is fixed; adjust as needed
sudo ip link set wwan0 up
sudo ip addr add 10.45.0.2/16 dev wwan0

sudo iptables -F
```
Now you can ping the Host on 10.45.0.1 and the UE on 10.45.0.2 (we have manually assigned the modem's IP address to the reciever machine by ip spoofing)
