분석해 보니, 현재 MTP 장치가 Windows에서 **MTP 표준 드라이버(wpdmtp.inf)** 대신 **WinUSB(INF)** 로 인식되고 있습니다.
즉, USB 장치가 MTP Class(06h)로 보이긴 하지만, Windows가 이를 “휴대용 장치(MTP)”로 처리하지 않고 그냥 WinUSB 장치로 잡아버립니다.
이 경우 탐색기에 절대 안 뜹니다.

---

## 문제 원인

1. **USB Descriptor 설정 문제**

   * `bDeviceClass`, `bInterfaceClass`, `bInterfaceSubClass`, `bInterfaceProtocol` 값이 MTP용으로 맞춰져 있어야 합니다.
   * MTP는 **Still Image Class (0x06)**, SubClass=0x01, Protocol=0x01 이어야 합니다.
   * Vendor-specific Class(0xFF)로 되어 있으면 WinUSB로 잡힙니다.

2. **USB INF 매칭 실패**

   * Windows는 `usb.inf`, `wpdmtp.inf` 매칭을 통해 MTP 드라이버를 설치합니다.
   * VID/PID가 표준 MTP 장치로 등록되지 않으면 자동 설치가 안 될 수 있습니다.

3. **USB String Descriptor**

   * iProduct, iManufacturer, iSerialNumber 등이 MTP에서 요구하는 형태와 맞지 않으면 드라이버 매칭이 실패합니다.

4. **WinUSB INF 강제 설치 이력**

   * 이전에 해당 VID/PID에 대해 WinUSB를 수동으로 설치한 적이 있으면, 그 설정이 레지스트리에 남아 MTP로 안 바뀔 수 있습니다.

---

## 해결 방법

1. **STM32 MTP 디스크립터 수정**

   * `usbd_mtp.c` 또는 `usbd_desc.c`에서 다음 값 확인:

     ```c
     #define MTP_INTERFACE_CLASS     0x06  // Still Image
     #define MTP_INTERFACE_SUBCLASS  0x01
     #define MTP_INTERFACE_PROTOCOL  0x01
     ```
   * `bDeviceClass`는 0x00(Interface 지정)으로 두고, Interface Descriptor에서 위 값 지정.

2. **VID/PID 변경**

   * ST 기본 MTP 예제에 맞추거나, 표준 MTP 장치와 유사한 값 사용:

     ```
     VID = 0x0483 (STMicroelectronics)
     PID = 0x5740 ~ 0x5742 (ST 예제)
     ```

3. **Windows 드라이버 캐시 제거**

   * 관리자 CMD에서:

     ```powershell
     pnputil /enum-devices | findstr 0483
     pnputil /remove-device "DEVICE_ID"
     pnputil /enum-drivers | findstr winusb
     pnputil /delete-driver oemXX.inf /uninstall /force
     ```
   * 이후 장치 다시 연결

4. **wpdmtp.inf 수동 설치**

   * 장치 관리자 → 해당 장치 → 드라이버 업데이트 → `휴대용 장치` → `MTP USB Device` 선택

---

원하시면 제가 **현재 로그 기반으로 STM32 MTP USB Descriptor 수정 예제**를 만들어서 드릴 수 있습니다.
그러면 Windows에서 바로 "휴대용 장치 > MTP USB Device"로 잡히게 됩니다.

바로 예제 만들어 드릴까요?
