# SAMSUNG_GALAXY_TAB_ACTIVE_AUTOBOOT
(KR) 삼성 갤럭시 탭 액티브 3 (SM-T575N) 용 자동 부팅 패치 

## 요약
알다시피 갤럭시 탭 액티브 제품군은 차량에서 사용하는 것을 상정하고 만들어진 제품이다. 특히 차량용 네비게이션 업체들이 대부분 죽을 쑤고 있는 2020년대 들어서, 탭 액티브 제품군은 적당한 화면 크기와 강력한 내구성 그리고 번인 현상이 적은 LCD 디스플레이 (번인이 없다고는 안 했다. 하.... 나도 알고 싶지 않았으니 묻지 마시라... 매우 아쉽게도 난 거지 동네 구석탱이 쪽방에 살지 강남대로에 살지 않는다.) 를 장착하고 있어, 차량용 네비게이션의 대용품으로 인기가 있는 상황이다.


자... 그렇다면 차에 탈때마다 항상 태블릿을 거치해서 충전을 한 뒤 전원 버튼을 길게 누르는 삽질을 해야 되겠는가? 그냥 연결한 채로 냅두고 시동만 켜면 알아서 켜지게 해서는 안 되는가? 
삼성 KNOX에서 비슷한 기능을 하는 사례가 있지만 안타깝게도 내 태블릿에는 본인의 또 다른 Android 프로젝트이자 안드로이드용 인포테인먼트 쉘 시스템인 'Fascia' 가 설치되어 있고 그것을 위해 이미 루팅이 된 상황이다. 그렇기 때문에, 보다 근본적인 솔루션이 필요했다. 


이 프로젝트의 목표는 완전히 꺼진 Galaxy Tab Active3에 USB 전원이 들어오면 Samsung 저전력 충전 화면에 머무르지 않고 자동으로 Android까지 부팅되게 하는 것이었다. 사용 시나리오 중 넘버원이라고 할 수 있는 것은 다름아닌 차량용 인포테인먼트다. 차량 시동 또는 ACC 전원이 들어오면 USB 전원이 태블릿에 공급되고, 사용자는 전원 버튼을 누르지 않아도 내비게이션이나 차량 UI를 곧바로 볼 수 있어야 한다.


## 연구 질문

그니까... 난 사실 이 시점에서 리눅스 및 AOSP의 내부 구조를 정확히 이해하지 못한 상황이엇다. 왜냐면 난 매우 간지나는 임베디드 시스템 엔지니어지만, 그것이 나를 리눅스 천재로 만들어주지는 않기 때문이다... 아님 말고. 그러니 이 프로젝트를 통해 리눅스 및 안드로이드의 부팅 구조를 이해할 수 있으면 그것도 아주 좋은 방향인 것 같다. 


즉, 완전히 꺼진 `SM-T575N`에 USB 전원이 들어왔을 때, 물리 전원 버튼 없이 Android 정상 boot와 사용자 동작 관점에서 완전히 동일한 상태로 뜰 수 있는가? 


여기서 "정상 boot와 거의 같은 상태"는 다음을 뜻한다.


- `sys.boot_completed=1`에 도달한다.
- touch 입력 장치가 등록된다.
- S-Pen 입력 장치가 등록된다.
- Android framework와 vendor service가 정상적으로 올라온다.
- 사용자는 Samsung LPM 충전 UI에 오래 머무르지 않는다.
- 가능한 한 custom/unlocked boot warning을 추가로 반복해서 보지 않는다.
- rollback이 가능한 boot partition 범위 안에서 수정한다.


우리는 부가적으로 다음이 또 궁금하다:
1. Samsung charger/LPM 화면은 Android와 별개인가, 아니면 같은 boot image 안의 제한 부팅 모드인가?
2. Android userspace에서 `androidboot.mode=charger`만 숨기면 정상 boot와 같아지는가?
3. touch가 죽는 원인은 SystemUI/InputDispatcher 수준인가, 커널 driver registration 수준인가?
4. boot image만 수정해서 해결할 수 있는가, 아니면 bootloader/param/up_param 같은 더 위험한 영역이 필요한가?
5. 이 방법을 앞으로 다른 rooted Samsung 기기에 적용하려면 무엇을 확인해야 하는가?


이 중 일부는 알고 있고 일부는 아직 모른다. 왜냐면 난 배달을 해서 하루하루 벌어먹는 가난한 배달기사이고 여러 태블릿을 afford할 돈이 없기 때문이다만, 추후 내가 태블릿을 교체할 경우 동일한 시도를 할 수 있게 하기 위해 여기에 Document하겠다.


뭐 그런 날이 오지 않았으면 좋겟다. 난 Active 3를 사랑한다...


## 배경 지식


### Android Boot에서 androidboot.mode


특히 당신이 EE나 CS 학과를 전공하는 학부생이지만 아직 이 분야에 대해 잘 알지 못한다면, 나는 이것을 간단하게 요약할 수 있다:
아주 간단하게 말해 '야 호스트, 너 누가 널 켰냐? 뭐 때문에 켜졋냐?' 이다.


거의 대부분의 안드로이드 기기는, 전원이 꺼진 상태에서 충전기를 연결할 경우 다음의 동작을 한다 - 
"충전 대기 중" 이미지 -> '현재 배터리 퍼센트가 나오는 이미지'" 의 동작을 하며, 사용자는 "현재 배터리 퍼센트가 나오는 화면" 을 볼 때까지 약간의 대기 시간 동안 기다려야 하지만 이것이 휴대전화를 완전히 시작하는 시간만큼 걸리지는 않는다. 


본 연구는 바로 여기서 모티브를 얻어 진행한다. 그렇다면, 이 "충전 대기 중" 이미지의 정체는 대체 무엇이란 말인가? 이게 Firmware 상에서 나오는 이미지라는 것은 알겠는데, 아무래도 이 화면을 Display Firmware 또는 PMIC가 그대로 띄우는 것이 아니고, AP를 한번 거쳐서 띄우는 것이 아닌가? 특히 요즘의 Charging Status 디스플레이는 다양하고 역동적인 애니메이션을 보유한 경우가 많으며 이것은 분명히 AP 내에서 2D 그래픽 연산 또는 어느 정도의 이미지 프로세싱이 돌아간다는 것을 의미할 것이다...


특히 Samsung의 구현 사례를 보면, 삼성의 부트로더는 부팅 원인에 따라 Kernal Command Line 또는 Device Tree Chosen Bootargs에 androidboot.mode = charger 또는 그에 국한되지 않는, 비슷한 유형의 값을 넘긴다.  그리고 android init과 magisk init은 본 값을 읽어 ro.bootmode, ro.boot.mode와 같은 property를 만든다. 일반 전원 버튼 부팅이면 대개 Android framework를 곧바로 올리지만, charger boot이면 제한된 charger service만 실행할 수 있다.


본인의 가설을 검증하기 위해, 펌웨어 파일 확인 결과, 이 장치에서는 power-off 상태에서 USB 전원만 들어오면 bootloader가 charger/LPM 경로를 선택했다. 즉 SoC와 kernel은 실제로 켜지지만, Android 전체가 아니라 Samsung `lpm` 충전 UI 중심으로 움직인다.


### Samsung의 LPM과 sec-charger는 대체 무엇을 하는가?


이것을 확인하기 위해, 실기기 dump를 확인햇다. 여기서는 Samsung charger service는 `/system/bin/lpm`이며, init class 이름은 일반 AOSP의 `charger`가 아니라 `sec-charger`였다. 기존 AOSP init에는 다음 개념이 남아 있었다.

```
# Healthd can trigger a full boot from charger mode by signaling this
# property when the power button is held.
#on property:sys.boot_from_charger_mode=1
#    class_stop charger
#    trigger late-init
```

하지만 Samsung 펌웨어에서는 이 action이 주석 처리되어 있었고, class 이름도 `charger`가 아니라 `sec-charger`였다. 우선, 이 관찰을 이용해 보기로 했다. 그래.. 뭐라도 나오겠지. 아님 말고.


### MagiskInit의 역할?


Magisk는 boot image의 ramdisk 안에서 `/init`을 가로채고, rootfs overlay를 구성한 뒤 원래 Android init으로 넘긴다. 그렇기 때문에, 일단 내가 시도한 첫 시도에서는 Magiskinit과 ramdisk overlay를 이용했다. 


Magisk 30.7의 기본 동작은 charger mode에서 overlay를 적용하지 않고 recovery/charger fallback으로 빠지는 것이었다. 특히 첫 시도에서는 이것을 실패했는데, 왜냐하면 본 환경에 설치된 Magisk 30.7의 기본 동작이 charger mode로 부팅된 상황에서는 해당 기능이 비활성화되고 overlay를 적용하지 않고 recovery / charger 용 fallback으로 떨어져서 작동하지 않았다.


다른 Magisk 버전에서는 이러한 동작을 하지 않는다. 그렇기 때문에 첫 시도에서 Magisk 버전 변경으로 만족할 수도 있었을 것이지만, 나는 이러한 서드파티 애플리케이션의 동작에 내 개조가 영향을 받지 않았으면 좋겠었다.



#### Kernel Built-In Driver와 Module 간 차이

이 장치의 kernel config에서 touch/pen 관련 설정은 다음처럼 확인됐다.

```
CONFIG_SOC_EXYNOS9810=y
CONFIG_EXYNOS_SOC_NAME="9810"
CONFIG_TOUCHSCREEN_HIMAX_CHIPSET=y
CONFIG_TOUCHSCREEN_HIMAX_COMMON=y
CONFIG_TOUCHSCREEN_HIMAX_SPI=y
CONFIG_TOUCHSCREEN_HIMAX_IC_HX83102=y
CONFIG_INPUT_WACOM=y
CONFIG_EPEN_WACOM_W9019=y
CONFIG_MODULES=y
CONFIG_KALLSYMS=y
# CONFIG_LTO is not set
```

중요한 점은 Himax touch와 Wacom pen이 module이 아니라 `=y` built-in이라는 점이다. `/proc/modules`도 비어 있었다. 따라서 charger boot에서 드라이버 init이 한번 skip되면, userspace에서 `modprobe`나 module reload로 다시 올리는 접근은 맞지 않는다.


## 실험 결과


우선, 실험 결과를 밝히기 전 초기 기준은 Magisk 30.7이 적용된 정상 root boot image였다는 점을 밝힌다.


```
original Magisk boot sha256 =
a773a5cf318b1188a189eec79251a31ea846afd42b3af595d2b36114ebade508
```


### Version 1: 오버레이만으로 시도해 보기


#### 가설


charger boot도 결국 같은 boot image의 ramdisk/init을 쓰므로, ramdisk overlay에 `on charger` action과 worker script를 추가하면 USB 전원 인가 시 자동으로 Android boot로 전환할 수 있을 것이라고 보았다.


#### 구현


v1은 기존 Magisk boot image의 kernel, DTB, header를 그대로 두고 ramdisk에 다음을 추가했다.


```
overlay.d/init.tabactive3-autoboot.rc
overlay.d/sbin/tabactive3-autoboot.sh
```

그 결과 정상 Android boot에서는 overlay 파일이 존재했지만, charger boot에서는 worker 로그가 생기지 않았다. Samsung `last_lpm.log`에는 다음 성격의 로그가 있었다. 또한 충전기 연결 시 충전 화면으로 넘어갔으며, 사용자 측면에서는 어떠한 변화가 없었다. 


내부 로그에는

```
magiskinit: Charger mode or ramdisk is recovery, abort
```


가만보면 magisk는 원래 그것이 동작하는 것대로 동작했을 뿐이다. 본 스크립트는 틀리지 않앗다. 단순히 Magisk가 충전기 부팅 모드에서 오버레이 자체를 적용하지 않아서 매지스크 관련 명령이 전혀 먹히지 않는다는 것. 


Magisk의 기타 버전에서는 이것이 정상적으로 동작한다는 언급이 있었지만, 전술했듯 내 프로젝트가 다른 서드파티 애플리케이션의 구현 상태에 따라 다르게 동작하는 것을 원치 않아 본 가설은 폐기하였다.



### Version 2: 오버레이만으로 시도해 보기

#### 가설

Magiskinit이 charger mode에서 overlay를 포기하는 조건만 제거하면, v1의 rc와 worker가 charger boot에서도 실행될 것이다.


#### 구현


Magisk v30.7 소스의 init 분기에서 charger 비교만 제거했다. recovery 감지는 유지했다.

```
 } else if cstr!("/sbin/recovery").exists()
     || cstr!("/system/bin/recovery").exists()
-    || unsafe { CStr::from_ptr(self.config.boot_mode.as_ptr()) } == c"charger"
 {
     self.recovery_or_charger();
```


해당 실험 결과 Charger Boot 모드에서 MagiskInit과 Overlay Worker가 실행되는 것을 제대로 확인할 수 있었다. 즉 V1의 가장 큰 차단점을 내가 성공적으로 풀었다고 할 수 있다.

```
magiskinit: First Stage Init
magiskinit: Second Stage Init
init: starting service 'tabactive3-autoboot'...
```

즉 두 번쨰 버전은 최종적으로 다음의 방식으로 Reboot를 요청했다:

```
setprop sys.powerctl reboot
```


그러나 실기기에서는 정상 Android로 안정적으로 이어지지 않았다. attempt marker가 남아 다음 charger session에서 반복 시도를 막았다. 젠장...


즉, 이 실험에서 우리는 charger session에서 단순히 reboot를 요청하는 방식은 bootloader/S-Boot 경로를 다시 통과해야 하며, 목적한 normal boot reason이 보장되지 않기 때문에 의도한 기능을 구현할 수 없다는 결론에 도달했다. 다음 단계에서는 reboot 없이 현재 charger session을 Android normal init sequence로 전환하는 방법을 찾기로 했다.



### Version 3: `sec-charger` 중지 후 `late-init`


#### 가설 및 구현

Android init에는 charger mode에서 full boot로 넘어가는 개념이 원래 존재한다. 그니까 '원래 안드로이드에서는 이것이 된다' 는 것이었다. 그저 Samsung 펌웨어에서 그 action이 주석 처리되어 있을 뿐이라면, overlay rc로 equivalent action을 추가할 수 있다.


Samsung charger class 이름이 `sec-charger`임을 확인하고, 다음 action을 overlay rc에 추가했다.


```
on property:sys.boot_from_charger_mode=1
    class_stop sec-charger
    trigger late-init
```

worker의 최종 동작은 reboot가 아니라 property set으로 바뀌었다.

```
setprop sys.boot_from_charger_mode 1
```


그 결과 power-off 상태에서 USB 전원 인가 후 Android까지 도달했다. 로그는 다음 성공 경로를 보였다.


```
charger trigger received
power stable; requesting in-place late-init transition
normal boot stable; attempt guard cleared
```


어... 된건가? 이제 내 기기를 장착하러 가도 되는가? 라고 생각해서, 화면을 터치해 보니 아무 기능도 동작하지 않는다. 정말이다. 터치가 동작하지 않았다. 설마 터치패널이 고장났나? 아니 그럴리가 없는데... 아니 고장나면 진짜 큰일인데. 알리익스프레스에서 부품을 살 수 있지만 그것을 사면 또 배송이 오래 걸리고, 그 동안 Fascia 개발에 차질을 빛겠지... 하...... 


#### 한계


이 버전은 일단 기능적으로는 성공했다. 태블릿의 전원이 정상적으로 들어왔다! 그렇지만 우리가 좋아해서는 안 되는 이유는, 본 시도는 본질적으로 "charger boot session을 나중에 Android로 이어 붙인 것"이다. bootloader와 kernel은 여전히 charger mode로 시작했다. 그리고 그 Bootloader와 Kernal의 Charger Mode 동작에서는 kernel built-in driver 중 일부가 불러와지지 않을 수 있었다는 것이고.....


역시 실제 내 3번쨰 실험에서는 드라이버 중 일부가 전혀 실행되지 않았으며 거기에 터치 드라이버와 와콤 펜 드라이버가 포함되어 있었다. 그러니 터치가 안 되지. 즉 이것을 실사용할 수 있다고 가정할 수 없었다.


뭐... 어쨋든 이번 시도로  "일단 Android가 올라오면 되는가?" 라는 중심 질문에 대해서는 'YES' 라는 답을 내놓을 수 있었다. 이후 질문은 "Android가 정상 boot와 같은 hardware driver 상태로 올라오는가"로 바뀌었다.




