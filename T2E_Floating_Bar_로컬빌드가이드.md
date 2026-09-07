# T2E Floating Bar — 로컬 빌드/개조 가이드

원본 `com.tools.lgv30.floatingbar` (LG V30 Floating Bar) APK를 로컬 PC에서 직접 개조·리빌드하는 전체 절차입니다.
지금까지 적용한 모든 변경(안드로이드 16 호환, 아이콘, billing 제거, 프리미엄 해제, 앱 이름, 플로팅바 상주)을 그대로 재현할 수 있습니다.

> ⚠️ 개인 소장/학습용 개조 가이드입니다. 재배포·상업적 이용은 원저작권/스토어 정책을 확인하세요.

---

## 0. 준비물 (도구 설치)

| 도구 | 버전 | 용도 |
|------|------|------|
| JDK | 17+ (21 권장) | apktool/서명 실행, keytool |
| apktool | 2.11.1 | APK 디코드/리빌드 |
| uber-apk-signer | 1.3.0 | zipalign + v1/v2/v3 서명 |
| Python + Pillow, numpy | 3.x | 아이콘 투명 처리 |
| Android SDK platform-tools | 최신 | `adb`로 폰 설치 |

```bash
# 작업 폴더
mkdir t2e && cd t2e && mkdir tools

# apktool
curl -L -o tools/apktool.jar \
  https://github.com/iBotPeaches/Apktool/releases/download/v2.11.1/apktool_2.11.1.jar
# uber-apk-signer
curl -L -o tools/uber-apk-signer.jar \
  https://github.com/patrickfav/uber-apk-signer/releases/download/v1.3.0/uber-apk-signer-1.3.0.jar
# python 라이브러리
pip install Pillow numpy pyaxmlparser

# 원본 APK를 이 폴더에 orig.apk 로 둡니다.
```

---

## 1. 디코드

```bash
java -jar tools/apktool.jar d -f -o dec orig.apk
```
`dec/` 안에 `AndroidManifest.xml`, `apktool.yml`, `smali/`, `res/` 가 생성됩니다.

---

## 2. `apktool.yml` — targetSdk 상향 (설치 경고 제거)

`dec/apktool.yml` 에서:
```yaml
sdkInfo:
  minSdkVersion: 21
  targetSdkVersion: 36   # 26 → 36 으로 변경
```
> targetSdk를 올려야 "이전 버전용으로 만들어졌다" 설치 경고가 사라집니다. 대신 아래 3·5·6 호환 처리가 필수입니다.

---

## 3. `AndroidManifest.xml` 변경

### 3-1. 권한/속성
- `com.android.vending.BILLING` 권한 **줄 삭제** (billing 제거)
- INTERNET 아래에 추가:
```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
```
- `<application ...>` 태그에 `android:usesCleartextTraffic="true"` 추가(날씨 http 대비), 그리고
  `android:label="@string/app_name"` → `android:label="T2E Floating bar"` (앱 이름)
- `<application` 바로 앞에 앱목록 조회용 queries 추가:
```xml
<queries>
    <intent>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent>
</queries>
```

### 3-2. `android:exported` 명시 (Android 12+ 설치 필수)
인텐트필터가 있거나 시스템이 바인딩하는 컴포넌트는 `exported="true"`, 나머지는 `"false"`.
- `true`: SplashActivity, AccessibilityActionService, EnableTileService, RemoteControllerService, StartUpBootReceiver, AdminReceiver
- `false`: MainActivity, SettingActivity, RearrangePageActivity, UnlockActivity, CustomDialogActivity, RequestPermissionActivity, CustomThemeActivity, SupportActivity, WebviewActivity, ScreenCaptureActivity, MainService

각 컴포넌트 태그의 `android:name` 앞에 `android:exported="true|false"` 를 넣으면 됩니다. 예:
```xml
<activity android:exported="true" android:name="com.tools.lgv30.floatingbar.activity.SplashActivity" ...>
```

### 3-3. MainService 를 포그라운드 서비스로 (홈화면 상주)
```xml
<!-- 기존 -->
<service android:exported="false" android:name="com.tools.lgv30.floatingbar.service.MainService"/>
<!-- 변경 -->
<service android:exported="false" android:foregroundServiceType="specialUse"
         android:name="com.tools.lgv30.floatingbar.service.MainService">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="floating_bar_overlay"/>
</service>
```

---

## 4. 헬퍼 클래스 추가 — `Cc.smali`

`dec/smali/com/tools/lgv30/floatingbar/Cc.smali` 파일을 **새로 생성**하고 아래 내용을 넣습니다.
(리시버 등록 플래그, 포그라운드 시작/승격을 안전하게 감싸는 유틸)

```smali
.class public Lcom/tools/lgv30/floatingbar/Cc;
.super Ljava/lang/Object;
.source "Cc.java"

.method public constructor <init>()V
    .locals 0
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V
    return-void
.end method

# RECEIVER_NOT_EXPORTED(0x4) 로 registerReceiver (API33+)
.method public static reg(Landroid/content/Context;Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;)Landroid/content/Intent;
    .locals 2
    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I
    const/16 v1, 0x21
    if-lt v0, v1, :cond_0
    const/4 v0, 0x4
    invoke-virtual {p0, p1, p2, v0}, Landroid/content/Context;->registerReceiver(Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;I)Landroid/content/Intent;
    move-result-object v0
    return-object v0
    :cond_0
    invoke-virtual {p0, p1, p2}, Landroid/content/Context;->registerReceiver(Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;)Landroid/content/Intent;
    move-result-object v0
    return-object v0
.end method

# RECEIVER_EXPORTED(0x2) — 결제 브로드캐스트용
.method public static regExp(Landroid/content/Context;Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;)Landroid/content/Intent;
    .locals 2
    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I
    const/16 v1, 0x21
    if-lt v0, v1, :cond_0
    const/4 v0, 0x2
    invoke-virtual {p0, p1, p2, v0}, Landroid/content/Context;->registerReceiver(Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;I)Landroid/content/Intent;
    move-result-object v0
    return-object v0
    :cond_0
    invoke-virtual {p0, p1, p2}, Landroid/content/Context;->registerReceiver(Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;)Landroid/content/Intent;
    move-result-object v0
    return-object v0
.end method

# 서비스 시작 (크래시 방지 - 일반 startService)
.method public static fgStart(Landroid/content/Context;Landroid/content/Intent;)V
    .locals 1
    :try_start_0
    invoke-virtual {p0, p1}, Landroid/content/Context;->startService(Landroid/content/Intent;)Landroid/content/ComponentName;
    :try_end_0
    .catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0
    return-void
    :catch_0
    move-exception v0
    return-void
.end method

# 포그라운드 승격 (API26+, 실패해도 크래시 없음)
.method public static fgPromote(Landroid/app/Service;)V
    .locals 5
    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I
    const/16 v1, 0x1a
    if-ge v0, v1, :cond_0
    return-void
    :cond_0
    :try_start_0
    const-string v0, "titan_fb"
    new-instance v1, Landroid/app/NotificationChannel;
    const-string v2, "Floating Bar"
    const/4 v3, 0x2
    invoke-direct {v1, v0, v2, v3}, Landroid/app/NotificationChannel;-><init>(Ljava/lang/String;Ljava/lang/CharSequence;I)V
    const-string v2, "notification"
    invoke-virtual {p0, v2}, Landroid/app/Service;->getSystemService(Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v2
    check-cast v2, Landroid/app/NotificationManager;
    invoke-virtual {v2, v1}, Landroid/app/NotificationManager;->createNotificationChannel(Landroid/app/NotificationChannel;)V
    new-instance v1, Landroid/app/Notification$Builder;
    invoke-direct {v1, p0, v0}, Landroid/app/Notification$Builder;-><init>(Landroid/content/Context;Ljava/lang/String;)V
    const v0, 0x7f080095   # R.drawable.ic_tile — 본인 APK의 실제 ID로 확인
    invoke-virtual {v1, v0}, Landroid/app/Notification$Builder;->setSmallIcon(I)Landroid/app/Notification$Builder;
    move-result-object v0
    const-string v1, "Floating Bar"
    invoke-virtual {v0, v1}, Landroid/app/Notification$Builder;->setContentTitle(Ljava/lang/CharSequence;)Landroid/app/Notification$Builder;
    move-result-object v0
    invoke-virtual {v0}, Landroid/app/Notification$Builder;->build()Landroid/app/Notification;
    move-result-object v0
    const/16 v1, 0x3e9
    invoke-virtual {p0, v1, v0}, Landroid/app/Service;->startForeground(ILandroid/app/Notification;)V
    :try_end_0
    .catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0
    return-void
    :catch_0
    move-exception v0
    return-void
.end method
```
> `0x7f080095` 는 `ic_tile` 아이콘의 리소스 ID입니다. `dec/res/values/public.xml` 에서
> `<public type="drawable" name="ic_tile" id="0x..."/>` 값을 확인해 본인 APK에 맞게 넣으세요.

---

## 5. `registerReceiver` 호출을 헬퍼로 교체 (Android 14+ 크래시 방지)

아래 파일들에서 `invoke-virtual {...}, L...;->registerReceiver(Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;)Landroid/content/Intent;`
를 `invoke-static {같은 레지스터}, Lcom/tools/lgv30/floatingbar/Cc;->reg(...)` 로 바꿉니다 (레지스터 목록 그대로).

- **`reg` 로 교체 (9곳):** `a/d$a.smali`, `activity/MainActivity.smali`, `activity/UnlockActivity.smali`, `customview/Lgv30BarView.smali`, `customview/g.smali`, `customview/k.smali`, `fragment/b.smali`, `service/MainService.smali`, `service/RemoteControllerService.smali`
- **`regExp` 로 교체 (2곳, 결제 PURCHASES_UPDATED):** `activity/MainActivity$10.smali`, `activity/SupportActivity$3.smali`

한 방에 처리하는 sed 예시 (reg 그룹):
```bash
cd dec
PAT='s|invoke-virtual (\{[^}]*\}), L[^;]+;->registerReceiver\(Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;\)Landroid/content/Intent;|invoke-static \1, Lcom/tools/lgv30/floatingbar/Cc;->reg(Landroid/content/Context;Landroid/content/BroadcastReceiver;Landroid/content/IntentFilter;)Landroid/content/Intent;|'
sed -i -E "$PAT" \
  'smali/com/tools/lgv30/floatingbar/a/d$a.smali' \
  smali/com/tools/lgv30/floatingbar/activity/MainActivity.smali \
  smali/com/tools/lgv30/floatingbar/activity/UnlockActivity.smali \
  smali/com/tools/lgv30/floatingbar/customview/Lgv30BarView.smali \
  smali/com/tools/lgv30/floatingbar/customview/g.smali \
  smali/com/tools/lgv30/floatingbar/customview/k.smali \
  smali/com/tools/lgv30/floatingbar/fragment/b.smali \
  smali/com/tools/lgv30/floatingbar/service/MainService.smali \
  smali/com/tools/lgv30/floatingbar/service/RemoteControllerService.smali
# 결제 2곳은 위 PAT에서 ->reg( 를 ->regExp( 로 바꿔 동일 적용
```
> `Cc.smali` 자체의 registerReceiver(내부 구현)는 바꾸지 마세요.

---

## 6. 크래시/기능 관련 smali 패치

### 6-1. `PendingIntent` 불변 플래그 (Android 12+ 크래시 방지)
`activity/ScreenCaptureActivity.smali` 의 `a(...ScreenCaptureActivity;Ljava/lang/String;)V` 메서드:
- `.locals 6` → `.locals 7`
- `invoke-static {p0, v4, v1, v4}, Landroid/app/PendingIntent;->getActivity(...)` 바로 앞에 `const v6, 0x4000000` 추가
- 그 invoke의 마지막 인자 `v4` → `v6` 로 변경 (FLAG_IMMUTABLE)

### 6-2. 날씨 API http → https (평문 차단 대응)
`e/k.smali`:
```
const-string v0, "http://api.openweathermap.org/data/2.5/weather?appid="
→ const-string v0, "https://api.openweathermap.org/data/2.5/weather?appid="
```

### 6-3. 프리미엄 잠금 해제 — `p()` 항상 true
`e/c.smali` 의 `.method public static p(Landroid/content/Context;)Z` 본문 전체를 아래로 교체:
```smali
    .locals 1
    const/4 v0, 0x1
    return v0
```

### 6-4. billing 연결 무력화 (bind/unbind try-catch)
`b/c.smali`
- `bindService(...)Z` 호출을 `:try_start_z0 ... :try_end_z0 / .catch Exception :catch_z0` 로 감싸고, catch에서 `goto :goto_0`
- `unbindService(...)V` 호출도 `:try_start_z1 ... :try_end_z1 / .catch :catch_z1` 로 감싸고, 정상 흐름은 `goto :cond_1`, catch는 `move-exception` 후 `:cond_1` 로 진입
> 이렇게 하면 BILLING 권한이 없어도 결제 서비스 연결 시도에서 크래시가 나지 않습니다.

### 6-5. MainService 포그라운드 승격 + 시작 방식
- `service/MainService.smali` 의 `onStartCommand` 첫 줄(`.locals` 다음)에 삽입:
```smali
    invoke-static {p0}, Lcom/tools/lgv30/floatingbar/Cc;->fgPromote(Landroid/app/Service;)V
```
- `e/c.smali` 의 MainService 시작 2곳:
```
invoke-virtual {p0, v0}, Landroid/content/Context;->startService(Landroid/content/Intent;)Landroid/content/ComponentName;
→ invoke-static {p0, v0}, Lcom/tools/lgv30/floatingbar/Cc;->fgStart(Landroid/content/Context;Landroid/content/Intent;)V
```

---

## 7. 리소스 변경

### 7-1. "업그레이드 프로" 메뉴 숨김
`res/layout/activity_main.xml` 의 `@id/button_upgrade_pro` RelativeLayout 에
`android:visibility="gone"` 추가 (뷰는 남겨야 NPE 없음).

### 7-2. "(Premium)" 표기 제거 / 앱 이름
`res/values/strings.xml`, `res/values-ko/strings.xml`:
- `theme_custom`: `Custom Theme (Premium)` → `Custom Theme` (ko: `사용자 테마`)
- `app_name`: → `T2E Floating bar` (앱 내 제목 일관성)

### 7-3. 앱 아이콘 투명 처리 (배경 제거)
아이콘 원본 이미지를 `icon.png` 로 두고:
```python
from PIL import Image, ImageDraw, ImageFilter
import numpy as np
src = Image.open('icon.png').convert('RGB'); w,h = src.size
arr = np.asarray(src)
sent=(255,0,255)
work = src.copy()
for s in [(0,0),(w-1,0),(0,h-1),(w-1,h-1),(w//2,0),(w//2,h-1),(0,h//2),(w-1,h//2)]:
    ImageDraw.floodfill(work, s, sent, thresh=120)   # 모서리에서 흰 배경 채우기
wa=np.asarray(work); bg=(wa[:,:,0]==255)&(wa[:,:,1]==0)&(wa[:,:,2]==255)
alpha=np.where(bg,0,255).astype('uint8')
alpha=Image.fromarray(alpha,'L').filter(ImageFilter.GaussianBlur(2.0))  # 가장자리 페더
out=src.convert('RGBA'); out.putalpha(alpha)
for d,sz in {"mdpi":200,"hdpi":300,"xhdpi":400,"xxhdpi":600,"xxxhdpi":800}.items():
    out.resize((sz,sz), Image.LANCZOS).save(f"dec/res/drawable-{d}/ic_launcher.png")
```
> 밀도별 크기는 원본 `ic_launcher.png` 크기에 맞추세요(이 앱은 200~800). 마스크로 글자가 잘리므로 어댑티브 아이콘 대신 이 방식(풀 스퀘어 + 투명 배경)을 씁니다.

---

## 8. 리빌드 · 서명 · 설치

```bash
# 리빌드
java -jar tools/apktool.jar b dec -o out.apk

# 서명 키 생성(최초 1회) — 비번은 원하는 값으로
keytool -genkeypair -v -keystore t2e.jks -alias t2e -keyalg RSA -keysize 2048 \
  -validity 10000 -storepass CHANGEME -keypass CHANGEME \
  -dname "CN=T2E, OU=Mod, O=T2E, C=NA"

# zipalign + v1/v2/v3 서명
java -jar tools/uber-apk-signer.jar --apks out.apk \
  --ks t2e.jks --ksAlias t2e --ksPass CHANGEME --ksKeyPass CHANGEME --allowResign -o signed

# 폰 설치 (USB 디버깅 ON)
adb uninstall com.tools.lgv30.floatingbar   # 서명이 다르면 기존앱 먼저 삭제
adb install -r "signed/out-aligned-signed.apk"
```
> 같은 `t2e.jks` 를 계속 쓰면 다음 업데이트는 삭제 없이 덮어쓰기 설치됩니다(키를 잘 보관).

---

## 9. 검증(선택)

```bash
python3 - <<'PY'
from pyaxmlparser import APK
a=APK("signed/out-aligned-signed.apk")
print("label:", a.get_app_name(), "| target:", a.get_target_sdk_version())
print("BILLING:", "com.android.vending.BILLING" in a.get_permissions())
PY
```

---

## 10. 트러블슈팅

| 증상 | 원인/해결 |
|------|-----------|
| 설치 시 "이전 버전용" 경고 | targetSdk가 낮음 → 2번(36) |
| 설치 자체가 거부(Android 12+) | `android:exported` 누락 → 3-2 |
| 앱 실행/토글 즉시 크래시 | registerReceiver 플래그(5) 또는 PendingIntent(6-1), FGS 승격 실패(6-5의 try-catch) 확인. `adb logcat | grep -i "FATAL\|floatingbar"` |
| 날씨 안 뜸 | http→https(6-2) + cleartext(3-1) |
| 앱 목록/바로가기 비어있음 | `<queries>`(3-1) |
| 홈에서 바 사라짐 | MainService 포그라운드화(3-3 + 6-5) |
| Play 프로텍트 차단 | 기기측 검사 — "무시하고 설치" 또는 Play 프로텍트 스캔 일시 끄기. 접근성/알림접근 서비스를 매니페스트에서 빼면 신호↓ |

---

## 요약: 변경 파일 목록
- `apktool.yml` (targetSdk 36)
- `AndroidManifest.xml` (exported, 권한, queries, cleartext, label, MainService FGS)
- `smali/.../Cc.smali` (신규)
- `smali/.../e/c.smali` (p()=true, fgStart)
- `smali/.../e/k.smali` (https)
- `smali/.../b/c.smali` (billing try-catch)
- `smali/.../service/MainService.smali` (fgPromote)
- `smali/.../activity/ScreenCaptureActivity.smali` (PendingIntent)
- registerReceiver 11곳(reg/regExp)
- `res/layout/activity_main.xml` (업그레이드 프로 gone)
- `res/values*/strings.xml` (app_name, theme_custom)
- `res/drawable-*/ic_launcher.png` (투명 아이콘)
