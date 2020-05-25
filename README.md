# usb_redirection

## USB menu
USB redirection을 위한, CDS Client의 USB Menu Bar 작업에 대한 설명입니다. MVC 패턴을 유지하기 위하여 아래처럼 작성되었습니다.
먼저 MVC 패턴에 대해 간략하게 설명을 하고, Menu Bar에 대한 설명을 하겠습니다.
각 소 제목들은 코드 이름에 해당합니다. 해당 코드를 보면서, 읽으시면 더 알기 쉽습니다.

### CDS Client의 Menu Bar MVC 패턴
* USB가 뽑히는 상황을 위의 그림으로 설명드리도록 하겠습니다.
  * 유저가 USB를 뽑으면, 해당 netlink socket이 알림을 받습니다.
    * __사용자 => MODEL(MOdel 코드)__
  * 해당 알림을 처리하기 위하여, Controller에 USB 메뉴 목록 수정 요청을 합니다. 이 때, Model에서 업데이트된 usb 장치의 정보들을 Controller에게 넘겨 줍니다.
    * __MODEL(Model 코드) => CONTROLLER(Controller 코드)__
  * 모든 처리가 끝나면, VIEW에 사용자가 볼 화면을 업데이트 합니다. 
    * __CONTROLLER(Controller 코드) => VIEW(Window 코드)__


### Window
* 메뉴바를 위하여 views::SimpleMenuController와 views::MenuBar를 필요로 합니다.
  * 메뉴바의 생성이나, 메뉴 아이템들을 추가하는 방법은 app/basic/texteditor와 app/basic/imageviewer의 window 코드들을 참고하였습니다.
* Window::createAndSetMenuBar()에서 메뉴 생성 및 아이템 추가를 해줍니다.
  * menu의 아이템 추가법은, 크게 2개의 argument를 요구합니다.
    * 하나는 ID
    * 다른 하나는 메뉴 항목에 적힐 이름입니다.
  * USB 메뉴는 checklist로 생성하였으며, check 여부를 저장하도록 하였습니다.
  * 생성된 메뉴는 appendMenu로 메뉴바에 추가해주면 됩니다. 현재, 메뉴의 id는 임의의 값으로 설정하였습니다. (메뉴에 대한 handler는 동작이 되지 않는 것으로 보입니다. 따라서 임의의 값으로 id값을 주면 됩니다.)
* 각 메뉴 아이템의 클릭 동작들은 executeCommand를 보시면 됩니다. 이는 SimpleMenuController로부터 overrides한 함수입니다.
  * __중요한 점은, 항상 command 동작을 처리 후, focus를 View에게 돌려주어야만 합니다.__
    * 이 부분은 menuClosed에서 일괄처리하기에 따로 처리해 줄 필요 없습니다.

### Model
* USB 정보를 추출하고, USB 환경의 변경(새로운 USB 추가 및 꽂혀있던 USB의 제거)을 모니터링하는 netlink socket을 생성합니다. 이 작업은 setUsbInfo()에서 이루어집니다.
* USB 정보는 메뉴바의 체크리스트 체크여부, 이름, bus id값입니다. include/cds_cli_common.h에 정리되어있습니다.
 typedef struct {
   bool check;
   char drive_letter; // 메뉴 아이템에 쓰일 이름
   char busid[5];
 } usb_info;
* addUsbInfo, removeUsbInfo는 각각 USB가 새로 꽂힌 경우, 꽂혀있던 USB가 뽑히는 경우 USB 정보를 업데이트하는 함수입니다.

### Controller
* setSocketFd()에서 netlink socket fd를 Event loop에 추가하여 줍니다.
* Controller에서 Window의 refreshUSBMenuItems()를 호출하여 메뉴바의 목록을 갱신하여 주는데, 이를 위해 cds_client 코드에서 Controller에 Window 오브젝트를 세팅해주는 작업을 추가하였습니다.

### View
* onFdReadEvent에서 netlink socket의 read 요청이 있는경우(USB 연결 상태 변경), Controller의 onUsbChange()를 호출합니다.
  * Controller의 onUsbChange()에서는 usb가 뽑히거나 새로 추가된 경우, Window의 refreshUsbMenuItems()를 호출하도록 합니다.
  * USB redirection의 bind, unbind로 인해 생기는 add, remove 메세지는 무시하도록 합니다. (실제로 Hardware에서 뽑히거나 추가된 경우가 아니기 때문)

## Udev 사용법
* udev 사용법은 udev wiki와 usbip_list.c(list-devices 함수)코드를 참고하였습니다. 현재는 USB block device만을 모니터링하도록 되어있습니다.
* USB subsystem에서 block device를 먼저 찾습니다. 이는 usb를 찾게되면, busid를 가져올 수 있지만 drive letter(sdX)의 X 값을 가져올 수 없기 때문입니다.
* block device들 중, disk device만을 선택하였으며 ID_BUS값이 "usb"인 device만을 찾았습니다.
* 각종 필요한 정보들은, key-value 형식으로 저장되어있습니다. 따라서 udev_list_entry_get_by_name으로 원하는 정보들을 가져올 수 있습니다.
* bus id는 device의 path를 파싱하여 사용하였습니다.
