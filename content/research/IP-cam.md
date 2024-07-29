+++
date = "2024-01-31"
title = "IP Cam"
slug = "IP-cam"
+++



### Summary



### Getting the firmware 
Is not possible to download any updates for CIP-37210AT camera from Smartwares website, and Wayback Machine hasn't archived any interesting stuff either. 

![Alt text](images/image-1.png)

So, in order to get our hands on the firmware, we have two ways: intercept network traffic by mitm-proxy the app request or hook the designated function via Frida.

By launching the application on rooted emulator, one of the first requests seems to really fit the bill. 

![Alt text](images/image-2.png)


In fact, the response JSON provides the necessary URLs for downloading the latest firmware available (i.e., 3.2.0.0) for several camera models, including the **CP721IP** in question. 

![Alt text](images/image-3.png)

A quick look at the APK via JADX can also be convenient: searched for the firmware keyword, it's not hard to find locate the designated function. 

![Alt text](images/image-4.png)

Let's hook it with the following Frida rule. 

```javascript
Java.perform(() => {
  const CameraManager = Java.use('nl.homewizard.android.cameras.camera.CameraManager');

  // Hooking firmwareVersionForCameraModel method
  CameraManager.firmwareVersionForCameraModel.implementation = function (model) {
      console.log("Hooking firmwareVersionForCameraModel Method...");
      console.log("Model: " + model);
      const firmwareVersion = this.firmwareVersionForCameraModel(model);
      console.log("FirmwareVersion: " + firmwareVersion);
      return firmwareVersion;
  };
});
```

Just to unsure https://cupdate.net/x.tar.gz is the correct URL for this IP-cam. 

![Alt text](images/image-5.png)

Untar the archive and it contains a single bash script called "update.sh".  
The interesting part for our purpose is the following. 

![Alt text](images/image-6.png)

The variable ```$TAGURL``` holds the final URL and, for the CP721IP camera model, is build in this fashion.

```bash
$TARURL="http://provisioning.homewizard.com/cameras/3.2/C721IP_3.2.00.tar.gz"
```


### Modifying the firmware
The download archive "C721IP_3.2.00.tar.gz" contains the following files.

```bash
├── app
│   └── bin
│       └── homewizard.sh
├── bin
│   ├── camcli
│   ├── snapshotserver
│   └── unabtoCamera
├── etc
│   ├── admin_default
│   ├── applyFirmwareUpdate.sh
│   ├── diagnose.sh
│   ├── init.d
│   │   ├── 8188_ap.sh
│   │   ├── 8188_sta.sh
│   │   ├── alive.sh
│   │   ├── boot.sh
│   │   ├── clearVideoSymlinks.sh
│   │   ├── network.sh
│   │   ├── runUnabto.sh
│   │   ├── snapshot.sh
│   │   └── waakhond.sh
│   ├── initWifiAP.sh
│   ├── mansnap.sh
│   ├── rtsp_user.sh
│   ├── scanWifi.sh
│   ├── setWifiSD.sh
│   ├── set_name.sh
│   ├── swap_webserver.sh
│   ├── udhcpd.conf
│   ├── watchAPMode.sh
│   ├── watchErrorMode.sh
│   └── wifiTestPrep.sh
├── mnt
│   ├── mtd
│   └── music
│       └── beep.wav
└── usr
    └── share
        └── udhcpc
            └── default.script
```
























