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



### Version 4: `sec-charger` 중지 후 `late-init`


#### 가설 및 구현


여기까지 진행했으면, 일단 본 실험이 이루고자 하는 목표의 50%는 이룬 것이다. 어쨋든 태블릿이 충전기를 꼽았더니 켜졌잖아? 머리가 아프니 일단 밖에 나가서 오토바이를 타고 한 바퀴를 돌고 왔다. 


charger session에서 늦게 `late-init`을 호출하지 말고, PID 1이 boot mode를 읽기 전부터 `androidboot.mode=charger`를 `unknown`으로 보이게 하면 Android init이 처음부터 normal boot 쪽으로 갈 수 있을 것이다.


내 새로운 가설을 검증하기 위해, Magiskinit의 boot config parsing 단계에 다음 로직을 추가했다.


```
- `/proc/cmdline`을 읽는다.
- `/proc/bootconfig`를 읽는다.
- 내용 중 `androidboot.mode=charger`를 `androidboot.mode=unknown`으로 바꾼 shadow file을 `/dev` 아래 만든다.
- shadow file을 원래 proc file 위에 bind mount한다.
- Magisk와 Android init은 shadowed proc file을 읽는다.
```

패치의 핵심은 다음과 같다:

```
changed |= tabactive3_replace_all(str, "androidboot.mode=charger", "androidboot.mode=unknown");
...
if (xmount(shadow, target, nullptr, MS_BIND, nullptr) != 0) {
    PLOGE("TabActive3 autoboot: bind mount %s over %s", shadow, target);
    return content;
}
```

아, 또한 이 버전에서는 10초간의 Debounce Logic을 임의로 추가했던 부분을 제거했다. 아니 원래도 Charger Mode에서 Kernel을 불러오는 시간은 (최소한 한국인식 속도 개념으로는) 아주 답답하기 짝이 없기 때문이다. 아니 바빠 죽겠는데 언제 10초를 세고 있는가? 최소한 내 차에 있는 인포테인먼트가 그렇게 행동한다면 그것은 제거 대상 1순위이다. 


#### 결과: 성공한 부분과 실패한 부분


USB 전원 인가 후 화면상으로는 회색 글씨의 배터리 로딩 직후 Samsung 로고로 넘어갔다. Android property도 다음처럼 정상 boot에 가까워졌다.


```
ro.boot.mode = unknown
ro.bootmode = unknown
sys.boot_completed = 1
/proc/cmdline contains androidboot.mode=unknown
```


그런데도, 여전히 터치와 S펜이 동작하지 않았다. 수집 스냅샷 기준으로는:

```
boot sha256 = 1023780bb4401b2563d76c15fe89cb45d111e7816c69642cc753be7cf70aaa4c
ro.boot.mode = unknown
ro.bootmode = unknown
sys.boot_completed = 1
sec_touchscreen = absent
sec_touchpad = absent
sec_e-pen = absent
/sys/class/sec/tsp = absent
/sys/bus/spi/drivers/himax_tp = absent
```

반면 같은 버전 4의 이미지에서 ADB reboot를 시도하자, 거기서 올라온 normal reboot session에서는 너무 당연하게도 touch와 pen이 존재했고 정상적으로 기능했다. 이 차이는 문제가 SystemUI나 launcher가 아니라 kernel input device registration 단계에 있음을 의미했다.


#### 결정적 관찰?

```
/proc/cmdline: androidboot.mode=unknown
device tree chosen bootargs: androidboot.mode=charger
```

즉 Magiskinit이 userspace-visible proc file은 바꿨지만, kernel이 boot 초기에 이미 읽었거나 보관한 원본 bootargs는 바뀌지 않았다. touch/pen driver가 kernel 내부의 LPM flag를 보고 skip된다면 해당 버전의 방식만으로는 너무 늦게 동작해서 의도대로 동작하지 않는 것이다.



### Version 5: 워크어라운드.

#### 가설 및 구현


난 여기서 약간의 머리를 쓰기로 했다. 나는 방금 이 방식으로 부팅을 시도하면 터치 드라이버를 정상적으로 불러오지 않지만, 최소한 Android의 핵심 기능은 정상적으로 올라온다는 사실을 알았지. 그리고 거기서 재부팅을 하면 원래 Android 정상 부팅 상태로 다시 올라올 수 있다는 사실도 알고 있다.


잠깐, 그렇다면 이렇게 '나사 빠진' 부팅을 한번 시킨 뒤, 그것이 완료되지마자 정상 재부팅을 강제로 수행해버리면 되잖아? Android가 충분히 일찍 뜬 직후 정상 reboot를 한 번 요청하면, 두 번째 boot는 normal reboot reason으로 들어와 touch/pen이 정상 등록될 것이다.


물론 나도 알고 있다. 이것은 근본적으로 드라이버를 Charger 모드에서 불러오고자 하는 시도는 아니다. 그렇지만 일단 작동하게 하는 것도 나쁘지 않다고 생각했다. charger-origin을 감지하면 750 ms 뒤 다음 명령을 실행했다.

```
setprop sys.powerctl reboot,tabactive3-autoboot-v5
```


charger-origin signal은 userspace shadow가 아니라 device tree chosen bootargs를 읽어 판별했다.


```
/sys/firmware/devicetree/base/chosen/bootargs contains androidboot.mode=charger
```

반복 reboot를 막기 위해 guard file을 사용했다.

```
/cache/lpm/tabactive3-v5-autoreboot.done
```


당연히 이것은 매우 훌륭한 Fallback이었고 매우 정상적으로 동작했다. 그렇지만 알다시피 한국인들의 시간관념상 부팅이 될 때까지 기다리고 한번 더 재부팅을 하는 과정을 또 기다리는 것은 너무나도 고통스럽다. 특히 내 성질머리가 아주 급하다는 것이 중요했다. 뭐.. 차를 예열할 떄까지 켜진다 라고 생각하면 이해는 되는 부분이겠지만, 그는 루팅이 된 태블릿에서 부팅 전 쓸데없는 경고문을 한번 더 봐야 하고 그것을 스킵할려면 전원 버튼을 결국 수동으로 눌러줘야 한다는 사실에서 생각이 바뀌엇다. 


본 해결책은

- 너무 당연하게도, Android 시스템의 boot가 두 번 일어난다.
- unlocked/custom warning이 두 번 보이며 난 이것이 아주 불쾌했다.
- 차량 시동 후 usable UI까지 시간이 (아주 많이) 늘어난다.
- 문제의 root cause를 해결한 것이 아니라 normal reboot로 회피한 것이다.


이 때문에 다음 버전에서는 다시 kernel root cause를 찾기로 시도했다.



## 커널의 Root Cause 분석


Android property와 `/proc/cmdline`은 normal-like인데 왜 touch가 죽는가?


가능성은 세 가지였다.

1. Android InputDispatcher나 SystemUI가 touch를 막는다.
2. vendor service가 charger-origin을 보고 input device를 disable한다.
3. kernel built-in driver가 boot 초기에 LPM이면 아예 등록하지 않는다.

실제 증거는 3번을 가리켰다.


### 입력 장치의 존재 여부?


4번째 버전의 실패 스냅샷에는 `/proc/bus/input/devices`에 `sec_touchscreen`, `sec_touchpad`, `sec_e-pen`이 없었다. `/sys/class/sec/tsp`도 없었다. 이것은 단순히 UI가 입력을 무시하는 상황이 아니라 아예 처음부터 input device가 등록되지 않은 상황이다.

한편, 노멀 부트 스냅샷에는 다음이 존재했다.


```
N: Name="sec_touchscreen"
S: Sysfs=/devices/platform/13ae0000.spi/spi_master/spi21/spi21.0/input/input1
H: Handlers=event1

N: Name="sec_touchpad"
H: Handlers=event2

N: Name="sec_e-pen"
S: Sysfs=/devices/platform/10910000.hsi2c/i2c-7/7-0056/input/input3
H: Handlers=event3
```

`last_lpm.log`에는 Samsung LPM이 touch/pen sysfs를 찾지 못하는 로그가 반복됐다.

```
LPM Start
LPM-SET setInputDeviceLpmMode
LPM-Input No /sys/class/sec/tsp/input/enabled
LPM-Input No /sys/class/sec/sec_epen/input/enabled
KERNEL DRIVER MODULE has loaded!!! count 0
```

이는 LPM userspace가 touch를 꺼서 문제가 생긴 것이 아니라, 애초에 해당 sysfs가 만들어지지 않았음을 보여준다.


그리고.. 무엇보다도 결정적으로, 커널 Image에서 다음 문자열을 찾았다.



```
[HXTP][ERROR] %s %s: Do not load driver due to : lpm %d
himax_common_init
%s: %s: Do not load driver due to : lpm %d
wacom_i2c_init
lpcharge
bootmode_setup
```

그니까, 대놓고 'LPM 부트' 이기 떄문에 himax 터치패드 드라이버와 와콤 드라이버를 안 불러오겠다는 것이다. 


### 디스어셈블리

Init Wrapper를 디스어셈블리하여 관찰하기로 했다. 



다음 함수가 `androidboot.mode` 값을 `"charger"` 문자열과 비교하고, 일치하면 전역 lpcharge를 1로 저장했다.

```
0xb1a85c:
  compare boot mode with "charger"
  if equal:
      w2 = 1
      store w2 to lpcharge_global
  print "Low power charging mode: %d"
```

해석하면, Boot Mode를 "charger" 변수와 비교하겠다는 것이다. 만약 같으면 w2 의 값을 '1' 로 지정하고, 그 1을 'lpcharge_global' 에 저장한다는 것이다. 즉 '사람 말로' 번역하면 충전기로 부팅 시 충전 상태 플래그를 1로 저장한다는 개념이다. 


Himax 에서는

```
0x14d0454:
  ldr w8, [lpcharge_global] 
  cmp w8, #0x1
  b.ne normal_himax_init
  print "Do not load driver due to : lpm"
  return -19
```

w8 레지스터의 값을 앞서 설명한 lpcharge_global과 비교한다. 앞서 설명했듯 이것이 1일 경우 충전 모드이고 0일 경우 충전 모드가 아니다.
이것을 '1' 이라는 값과 비교하고, 이것이 1이 아닐 경우 (= 충전 모드가 아닐 경우) himax 드라이버 init 코드로 분기한다.


와콤 드라이버 에서는


```
0x14d04e8:
  ldr w3, [lpcharge_global]
  cbz w3, normal_wacom_init
  print "Do not load driver due to : lpm"
  return 0
```
동일하다. w3 레지스터의 값을 앞서 설명한 lpcharge_global과 비교한다. 앞서 설명했듯 이것이 1일 경우 충전 모드이고 0일 경우 충전 모드가 아니다. c


아.. cbz라는 명령이 뭔지 몰라 검색해봤다. ARM64 어셈블리 인스트럭션에서 이것은 비교해서 0일경우 분기하란 뜻이란다.
즉 w3이 0이라면 (= 충전 모드가 아닐 경우) 와콤 드라이버 init 코드로 분기한다. 


이쯤 되면 정답이 어느 정도 나온 것으로 보인다.



## 최종 시도

### 가설 및 시도

touch/pen driver를 각각 patch하는 것은 내 능력 밖이고, 복잡할 것이라고 판단했다. 그러나 본 어셈블리 코드를 본다면 해결책은 아주 간단해 보인다. 그 driver들이 공통으로 보는 `lpcharge` global flag가 처음부터 0으로 남게 하도록 패치를 '낑겨 넣으면' 된다.


charger-origin boot라는 사실은 bootloader/DT bootargs에 남아도 크게 상관이 없을 것이다. 다른 안드로이드 시스템의 동작에 잇어서 이것은 큰 영향을 끼치지 않는 것 같다. 그러나 kernel의 low-power-charging policy flag가 0이면 Himax와 Wacom init은 normal path를 탄다.


커널 raw Image offset `0xb1a884`의 ARM64 명령 한 개를 바꿨다.


```
original bytes: e2030032
original asm:   orr w2, wzr, #0x1

patched bytes:  e2031f2a
patched asm:    mov w2, wzr
```

ARM64에서 wzr은, 항상 0인 zero register이다. 즉 오리지널 명령은 "W2의 값을 0과 1로 OR 연산한다" 라면, 패치한 명령의 의미는 "W2를 0으로 설정해 버린다" 라는 의미 되시겠다.


이 결과 정상 동작이 성공하였으며, Flash 후 동작은 매우 원할하게 작동했다.  실제 bootloader 입력은 charger-origin 그대로인데, Android userspace는 normal-like로 보고, kernel driver도 `lpcharge=0` 때문에 LPM skip path로 빠지지 않는다.



## 장기 안정성 검증?


이 좁고 간단한 바이너리 패치로 우리는 모든 문제를 해결한 것처럼 보였으나, 실제로는 아직 안정성 테스트를 해야 할 부분이 많이 남아있을 것이다. 특히 다음과 같은 상황 말이다...


1. 배터리 잔량이 낮을 때 USB insertion boot.
2. 충전기가 약하거나 차량 전원이 순간적으로 흔들릴 때.
3. 완전 방전 근처에서 부팅 시도.
4. 여러 번 반복한 power-off -> USB insertion cycle.
5. boot 후 장시간 충전 중 발열 관리.
6. sleep/wake, screen off charging, cable unplug behavior.
7. ETC...

일단 1-3의의 경우, 배터리가 부족할 떄 전원이 켜졌다 꺼졋다를 빠르게 반복하나, 충전기의 성능이 괜찮은 것을 사용할 경우 안정적으로 부팅하여 충전이 가능했다. 나머지는 추후 검증해 나갈 것이지만, 일단 지금까지로써는 Green Flag이다.


## 롤백

이 프로젝트는 `boot` partition만 수정했다. 따라서 Download Mode와 Odin AP slot으로 기존 boot-only tar를 다시 넣는 rollback 전략이 유지된다. `param`, `up_param`, bootloader, vbmeta는 건드리지 않았다.


## 추후 다른 기기에 이 방식을 적용한다면??

비슷한 버전을 가진 삼성 루트 디바이스에 적용하는 절차라면,

1. 현재 boot partition을 반드시 dump하고 hash를 기록한다.
2. 대상 firmware build와 bootloader revision을 기록한다.
3. Magisk version을 고정한다.
4. power-off USB insertion boot에서 `/proc/cmdline`, DT chosen bootargs, `getprop ro.bootmode`를 수집한다.
5. touch/pen/input device가 charger-origin boot에서 빠지는지 확인한다.
6. kernel config에서 input driver가 module인지 built-in인지 확인한다.
7. kernel Image strings에서 다음 류의 문구를 찾는다.

```text
Do not load driver due to : lpm
lpcharge
bootmode_setup
charger
```

8. 해당 문자열을 참조하는 init wrapper를 disassemble한다.
9. driver별 skip branch를 patch할지, 공통 `lpcharge` setter를 patch할지 결정한다.
10. boot image를 repack하고 fresh unpack으로 hash와 bytes를 검증한다.
11. Odin AP boot-only tar를 만든다.
12. 첫 boot 후 ADB로 input device와 sysfs를 검증한다.



이 방법은 Bootloader Unlock이 된 기기에 한정해서 동작하며, Magisk Root 또는 Boot Image Repack이 가능한 기기에서 동작한다. 


## 적용 방법

boot.img를 odin에서 설치한다. 반드시 다른 파일을 지워놓고, 일반적인 표준 ODIN 사용법을 따른다. Re-partition 설정이 '반드시 꺼져 있는지' 확인할 것.


`0xb1a884` offset은 `SM-T575N / T575NKOSFEYI1`의 이 boot image에만 맞는다. 다른 펌웨어, 다른 모델, 심지어 같은 모델의 다른 보안 패치 레벨에도 그대로 쓰면 안 된다.


다른 기기에 적용할 때는 "같은 의미의 코드"를 새로 찾아야 한다. offset을 복사하는 것이 아니라 분석 절차를 복사해야 한다.


나는 이 이미지로 인해 발생하는 손상에 대해 책임지지 않을 것이며 모든 행위의 책임은 본인에게 있다. 그렇지만 해당 조건이 갖춰진 태블릿에서 boot.img 수정은 상대적으로 안전한 방식으로 생각하고 있다.

