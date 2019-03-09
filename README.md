Proof of concept [WebUSB] based Interface Device (IFD) handler for USB based Chip Card Interface Device(s) ([CCID]).

[WebUSB]: https://wicg.github.io/webusb/
[CCID]: http://www.usb.org/developers/docs/devclass_docs/DWG_Smart-Card_CCID_Rev110.pdf

### Demo ###
To run the demo, host a (simple) web server and access it with a WebUSB compatible Browser e.g. Chromium/Chrome >= 61 and <=67.0.3382, and a CCID/reader (see requirements).

```
                            +- Chromium/Chrome ----------+      +--> WebSocketServer.py
                            |----------------------------|      |    (send GET CHALLENGE and
                            |                            |      |     output the cards result)
                            |     demo.html       +--WebSocket--+
                            |                     |      |      +--> WebSocketServerVICC.py
                            +---------------------|------+      |    (make card available via PC/SC
+-----+    +--------+       |                     |      |      |     on server in virtual reader)
|Smart|<-->|USB CCID| <--WebUSB----> ifd.js <-----+      |      |
|Card |    |Reader  |       |                            |      +--> WebSocketServerPACE.py
+-----+    +--------+       +----------------------------+           (generate APDUs to
                                                                      establish a PACE channel)
```

#### Requirements ####
If your CCID is WebUSB compatible you are good to go.

Making a traditional, non-WebUSB CCID available to WebUSB requires operating system specific actions. As of April 2018 (Chromium Build > 546309, version 67.0.3382.0) [those readers are blocked for security reasons by their respective USB interface class (0x0B)](https://chromium.googlesource.com/chromium/src/+/537313cf17960ed943eb82cc696a781ef8769331). If they use a different interface class, not one of [0x01,0x03,0x08,0x0B,0x0E,0x10,0xE0], they can still be made available using instructions below. Otherwise, for debugging, you can use an old Chromium Snapshot e.g. [546309 for Linux x64](https://commondatastorage.googleapis.com/chromium-browser-snapshots/index.html?prefix=Linux_x64/546309/).
To use non-WebUSB readers in production, it may be a better idea to provide a proxy WebUSB driver, which forwards messages to the hardware, and does necessary security (origin) checks. Or opt for a browser extension (not plugin) approach to access respective native components.

- For Windows [Zadig](http://zadig.akeo.ie/) is recommended to load the generic WinUSB driver for your CCID.
- For Linux, the user's browser needs write access to the usb device. This can be done by creating a custom udev rule. See the following example rule created in `/etc/udev/rules.d/50-Identiv-4700F.rules` and adding your user to the `plugdev` group in `etc/group`. A generic rule for (specific) WebUSB devices could be added to the [udev project].
```
SUBSYSTEM=="usb", ATTR{idVendor}=="04e6", ATTR{idProduct}=="5720", GROUP="plugdev"
```
Vendor Id and Product Id of your CCID can be identified using `lsusb` command.

[udev project]: http://linux-hotplug.sourceforge.net/

#### Usage ####
Once your WebUSB device is available for the browser, in user space, you can follow the quick start instructions to get the demos up and running. You need to enable SSL/TLS encryption, if you want to host the server on a different machine. SSL/TLS encryption is left off for debugging.

1. Start web server (`python3 HttpServer.py`)
2. open `http://localhost:8000/demo.html` in Chromium/Chrome
3. Click "connect reader" and choose your USB CCID smart card reader

##### Roll the Dice #####
4. Insert smart card that accepts GET CHALLENGE (`00 84 00 00 00 00 01`)
5. Clicking the square sends GET CHALLENGE; the card's response is translated to the appropriate number of pips on the dice's face

##### WebSocket example #####
4. Install SimpleWebSocketServer manually or using `setup.py`, see `WebSocketServer.py`.
5. Insert smart card that accepts GET CHALLENGE (`00 84 00 00 00 00 01`)
6. `python3 WebSocketServer.py`
7. Clicking "request remote CAPDU" below the headline "remote WebSocket only" connects to WebSocketServer.py. The response is shown in the log.

##### Relay from Browser to PC/SC #####
4. Install [virtualsmartcard](http://frankmorgner.github.io/vsmartcard/virtualsmartcard/README.html). The Python dependencies can be installed using `setup.py` or manually, see `WebSockerServerVICC.py`.
5. Insert smart card
6. `python3 WebSocketServerVICC.py`
7. Start virtual smart card reader on localhost:35963 (i.e. start PCSC-Lite, or SCardSvr.exe with the vpcd driver)
8. Clicking "request remote CAPDU" in the section "remote vicc, vpcd, app" connects to WebSocketServerVICC.py, which relayes the card to the virtual smart card reader
9. Accessing the card in the virtual reader via PC/SC is relayed through the WebSocket and through the browser; the website's log shows the command and response APDUs.

Note: I included `vicc-vpcdHost.py` an example vpcd and app. You need Virtualsmartcards' Python files, either from the install above or manually downloaded, see `setup.py`.

##### Remote verify CAN with PACE #####
4. Install required python packages manually or using `setup.py`, see `WebSocketServerPACE.py`.
5. Insert smart card that is capable of performing PACE (e.g. German ID card)
6. `python3 WebSocketServerPACE.py`
7. In the section "Remote nPA PACE", enter the card's CAN and click "send". The browser connects to WebSocketServerPACE.py, which verifies the CAN with PACE; the responses are shown in the log.

Note: It is a bad idea to hand out your authentication token and its secret to a website, as in never give out your email password. In case of the nPA, EAC (Extended Access Control) is required to access the passport's data. But an attacker can use your entered PIN to authenticate on your behalf at a legitimate party. He can also do Pin-Management. Without technical understanding it is hard for users to distinguish a secure input method from an insecure. Maybe in the future browsers will provide this themselves. For now, a browser extension does the job. Or you own an advanced CCID, that provides a hardware PIN pad and transaction display.

### Usage in standalone applications based on electron ###
[Electron] provides a Chromium based framework to build native applications for Linux, Mac and Windows. If Chromium >= 61 and <= 67 is used, it supports WebUSB. Once started (`npm start`), `navigator.usb` should be available in the included developer console (Ctrl+Shift+I).

[electron]: https://electronjs.org/
[quick-start example]: https://github.com/electron/electron-quick-start

### Security considerations ###
Be aware that WebUSB forwards your local USB device into the browser context whose security context is the origin (protocol, host, port) of the visited website. This means once you allow a USB device using the WebUSB Browser dialog, the website's scripts can run arbitrary USB commands on your local device. In the case of a WebUSB device that is designed to operate under these conditions it's fine.
Classic USB devices on the other hand were not intended to be used remote as with WebUSB, and (function) abstraction layers like driver, library and application aren't present anymore. As a result the web application host serving the web application can use USB functions of the device, which may be flash(newFirmware) or deleteSomethingImportant().
In case of authentication, a website may use your (allowed) USB token as man-in-the-middle to authenticate on your behalf somewhere else. This can be recognized by the user, if the hardware token itself displays transaction information, and denied, if it has hardware based confirmation.

### TODO ###
This is currently a side project, I occasionally work on in my spare time.
- feature complete CCID?
- add ISO 7816-4 APDU helpers?
- build a working standalone release using an old electron version?
- implement PACE with webcrypto?
  + Adding elliptic curve points function is not exposed. I can reimplement this, but the cryptographic functions should be provided by the browser. Question: How can a user distinguish safe crypto use from possibly unsafe, self implemented crypto?
- software-engineer the project further: add automated testing, add automated doc generation?

### Acknowledgements ###
Since this project was only possible because of my work on my diploma thesis, a special thanks goes out to my two advisors for all the consultations, time, and effort. During that time I was supported by my alma mater, Humboldt-Universität zu Berlin (Humboldt University of Berlin), and Bundesdruckerei GmbH ("Federal Printing Office").
