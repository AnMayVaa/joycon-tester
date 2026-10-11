# Joy-Con & Phone Tester

A single-page web tool for testing hardware in real time.

**Live:** https://anmayvaa.github.io/joycon-tester/

## Joy-Con tab (laptop, Bluetooth)
Pair a Nintendo Switch Joy-Con (L/R) or Pro Controller with your computer, open the page in Chrome or Edge, and click **Connect Joy-Con**.

- Buttons, analog sticks (calibration, range, dead zone)
- Gyro + accelerometer with live 3D model and graphs
- HD Rumble: manual controls and 16 haptic scenes (cup with ice, marble in a box, sword clash…)
- Player lights, Home button light
- IR motion camera (Joy-Con R)
- Battery, firmware, test checklist

## Phone sensors tab (phone → laptop live link)
Open the page on your laptop, switch to **Phone sensors**, and scan the QR code with your phone. The phone streams its sensors to the laptop over a direct WebRTC link:

- Motion (accelerometer, gyro), 3D phone, bubble level, compass
- Multi-touch, GPS, microphone spectrum, live camera
- Laptop buttons trigger the phone's vibration motor and speaker

## Requirements
- Joy-Con: Chrome/Edge on desktop (WebHID)
- Phone: any modern mobile browser; internet for the initial connection

Technical references: [`JOYCON_TESTER.md`](JOYCON_TESTER.md) (Joy-Con) and [`PHONE_TESTER.md`](PHONE_TESTER.md) (phone).
