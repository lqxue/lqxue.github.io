---
title: 获取主卡和双卡手机IMEI，IMSI，ICCID
date: 2019-07-15
tags:
  - 双卡双待
categories: Android
---

## 首先要添加权限

```java
<uses-permission android:name="android.permission.READ_PHONE_STATE"/>
```

## 获取主卡的IMEI，IMSI，ICCID

- 如果一张手机卡那就是获取当前使用的手机卡的信息
- 如果2张手机卡,那就是获取的主卡信息,所谓主卡就是设置了默认打电话的卡

<!-- more -->

```java
/**
 * Return the phone status.
 *
 * <p>Must hold {@code <uses-permission android:name="android.permission.READ_PHONE_STATE" />}</p>
 *
 * IMEI 与你的手机是绑定关系 用于区别移动终端设备
 * IMSI 与你的手机卡是绑定关系 用于区别移动用户的有效信息 IMSI是用户的标识。
 * DeviceId就是IMEI
 * DeviceId = 99000311726612<br>
 * DeviceSoftwareVersion = 00<br>
 * Line1Number =<br>
 * NetworkCountryIso = cn<br>
 * NetworkOperator = 46003<br>
 * NetworkOperatorName = 中国电信<br>
 * NetworkType = 6<br>
 * PhoneType = 2<br>
 * SimCountryIso = cn<br>
 * SimOperator = 46003<br>
 * SimOperatorName = 中国电信<br>
 * ICCID ICCID是卡的标识，由20位数字组成
 * ICCID只是用来区别SIM卡，不作接入网络的鉴权认证。而IMSI在接入网络的时候，会到运营商的服务器中进行验证。
 * SimSerialNumber就是ICCID
 * SimSerialNumber = 89860315045710604022<br>
 * SimState = 5<br>
 * SubscriberId(IMSI) = 460030419724900<br>
 * VoiceMailNumber = *86<br>
 */
@SuppressLint("HardwareIds")
@RequiresPermission(READ_PHONE_STATE)
public static String getPhoneStatus() {
    TelephonyManager tm = getTelephonyManager();
    String str = "";
    //noinspection ConstantConditions
    str += "DeviceId(IMEI) = " + tm.getDeviceId() + "\n";
    str += "DeviceSoftwareVersion = " + tm.getDeviceSoftwareVersion() + "\n";
    str += "Line1Number = " + tm.getLine1Number() + "\n";
    str += "NetworkCountryIso = " + tm.getNetworkCountryIso() + "\n";
    str += "NetworkOperator = " + tm.getNetworkOperator() + "\n";
    str += "NetworkOperatorName = " + tm.getNetworkOperatorName() + "\n";
    str += "NetworkType = " + tm.getNetworkType() + "\n";
    str += "PhoneType = " + tm.getPhoneType() + "\n";
    str += "SimCountryIso = " + tm.getSimCountryIso() + "\n";
    str += "SimOperator = " + tm.getSimOperator() + "\n";
    str += "SimOperatorName = " + tm.getSimOperatorName() + "\n";
    str += "SimSerialNumber = " + tm.getSimSerialNumber() + "\n";
    str += "SimState = " + tm.getSimState() + "\n";
    str += "SubscriberId(IMSI) = " + tm.getSubscriberId() + "\n";
    str += "VoiceMailNumber = " + tm.getVoiceMailNumber();
    return str;
}
```

## 如果手机有多张卡

[TelephonyManager的官方源码](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/telephony/java/android/telephony/TelephonyManager.java)

其实多卡情况下主要要获得的是两个地方：getSubscriberId和getSimSerialNumber,打开上面的的源码，搜索一下这两个方法，发现这两个方法都有一个带参数(subId)的重载方法,并且这两个方法都是@hide的，hide倒是无所谓，这个可以通过反射调用，主要的问题要弄清楚他的这个参数subId是个什么东西。代码片段如下：

```java
/**
 * Returns the unique subscriber ID, for example, the IMSI for a GSM phone.
 * Return null if it is unavailable.
 *
 * <p>Requires Permission: {@link android.Manifest.permission#READ_PHONE_STATE READ_PHONE_STATE}
 * or that the calling app has carrier privileges (see {@link #hasCarrierPrivileges}).
 */
@SuppressAutoDoc // Blocked by b/72967236 - no support for carrier privileges
@RequiresPermission(android.Manifest.permission.READ_PHONE_STATE)
public String getSubscriberId() {
    return getSubscriberId(getSubId());
}

/**
 * Returns the unique subscriber ID, for example, the IMSI for a GSM phone
 * for a subscription.
 * Return null if it is unavailable.
 *
 * @param subId whose subscriber id is returned
 * @hide
 */
@RequiresPermission(android.Manifest.permission.READ_PHONE_STATE)
public String getSubscriberId(int subId) {
    try {
        IPhoneSubInfo info = getSubscriberInfo();
        if (info == null)
            return null;
        return info.getSubscriberIdForSubscriber(subId, mContext.getOpPackageName());
    } catch (RemoteException ex) {
        return null;
    } catch (NullPointerException ex) {
        // This could happen before phone restarts due to crashing
        return null;
    }
}
```

从上面的注释来来，这个subId 是subscription id的简写，既然提到subscription id，那不得不说的就是[SubscriptionManager](https://developer.android.com/reference/android/telephony/SubscriptionManager.html).

## Subscription和SubscriptionManager

一台设备可以有多张SIM卡，最典型的例子就是眼下流行的“双卡双待”。每一张SIM卡都对应一个Subscription，Subscription：谷歌翻译为“订阅”

**订阅**（Subscription）：定义了请求者与业务或业务操作之间的关联关系。只有定义了订阅关系，才能通过业务策略，控制请求者对某个业务的访问行为。

我们用谁家的SIM卡就相当于订阅(Subscription)谁家的业务，对应的SIM卡的信息就是`Subscription Information`，比如运营商名称、MNC、MCC等，多张SIM卡就有多个`Subscription Information`。

在Android中，针对上述功能的实现、管理就是SubscriptionManager，表现到软件上就是：

![](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/image/SubscriptionManager.png)

各个类的实现比较简单自行参看源码。

SubscriptionManager作用有三个：

1. 获取Subscription信息
2. 更改Subscription某些信息
3. 提供`OnSubscriptionsChangedListener`监听器，方便其他应用监听`Subscription`的状态改变

Subscription内容：

```java
/**
 * A Parcelable class for Subscription Information.
 */
public class SubscriptionInfo implements Parcelable {

    /**
     * Size of text to render on the icon.
     */
    private static final int TEXT_SIZE = 16;

    /**
     * Subscription Identifier, this is a device unique number
     * and not an index into an array
     */
    private int mId;

    /**
     * The GID for a SIM that maybe associated with this subscription, empty if unknown
     */
    private String mIccId;

    /**
     * The index of the slot that currently contains the subscription
     * and not necessarily unique and maybe INVALID_SLOT_ID if unknown
     */
    private int mSimSlotIndex;

    /**
     * The name displayed to the user that identifies this subscription
     */
    private CharSequence mDisplayName;

    /**
     * String that identifies SPN/PLMN
     * TODO : Add a new field that identifies only SPN for a sim
     */
    private CharSequence mCarrierName;

    /**
     * The source of the name, NAME_SOURCE_UNDEFINED, NAME_SOURCE_DEFAULT_SOURCE,
     * NAME_SOURCE_SIM_SOURCE or NAME_SOURCE_USER_INPUT.
     */
    private int mNameSource;

    /**
     * The color to be used for tinting the icon when displaying to the user
     */
    private int mIconTint;

    /**
     * A number presented to the user identify this subscription
     */
    private String mNumber;

    /**
     * Data roaming state, DATA_RAOMING_ENABLE, DATA_RAOMING_DISABLE
     */
    private int mDataRoaming;

    /**
     * SIM Icon bitmap
     */
    private Bitmap mIconBitmap;

    /**
     * Mobile Country Code
     */
    private int mMcc;

    /**
     * Mobile Network Code
     */
    private int mMnc;

    /**
     * ISO Country code for the subscription's provider
     */
    private String mCountryIso;

    /**
     * Whether the subscription is an embedded one.
     */
    private boolean mIsEmbedded;

    /**
     * The access rules for this subscription, if it is embedded and defines any.
     */
    @Nullable
    private UiccAccessRule[] mAccessRules;

    /**
     * The ID of the SIM card. It is the ICCID of the active profile for a UICC card and the EID
     * for an eUICC card.
     */
    private String mCardId;
}
```

看下对应的术语：

- ICCID：Integrate circuit card identity 集成电路卡识别码（固化在手机SIM卡中）。ICCID为IC卡的唯一识别号码，共有20位数字组成：

![](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/image/sim_iccid.png)

- PLMN：Public Land Mobile Network 公共陆地移动网，一般某个国家的一个运营商对应一个PLMN

- SPN：Service Provider Name 运营商名称

- MCC：Mobile Country Code，移动国家码，MCC的资源由国际电联（ITU）统一分配和管理，唯一识别移动用户所属的国家，共3位，中国为460

- MNC：Mobile Network Code 移动网络号码，用于识别移动用户所归属的移动通信网，2~3位数字组成，如中国移动系统使用00、02、04、07，中国联通GSM系统使用01、06、09

- ISO country code: 国际上不同国家的ISO编码，比如我们国家：cn

对于`SubscriptionController`来说，它是从数据库`/data/data/com.android.providers.telephony/databases/telephony.db`的siminfo表中读取的数据.
只要知道一点，这个类就是为了5.0之后，为了配合TelephonyProvider操作和处理`/data/data/com.android.providers.telephony/databases/telephony.db`这个数据库中的表的，就可以了。

```java
siminfo表的创建：

TelephonyProvider.java (g:\rk3288-android6.0f\packages\providers\telephonyprovider\src\com\android\providers\telephony)	105530	2017-04-24
private void createSimInfoTable(SQLiteDatabase db) {
	db.execSQL("CREATE TABLE " + SIMINFO_TABLE + "("
		+ SubscriptionManager.UNIQUE_KEY_SUBSCRIPTION_ID + " INTEGER PRIMARY KEY AUTOINCREMENT,"
		+ SubscriptionManager.ICC_ID + " TEXT NOT NULL,"
		+ SubscriptionManager.SIM_SLOT_INDEX + " INTEGER DEFAULT " + SubscriptionManager.SIM_NOT_INSERTED + ","
		+ SubscriptionManager.DISPLAY_NAME + " TEXT,"
		+ SubscriptionManager.CARRIER_NAME + " TEXT,"
		+ SubscriptionManager.NAME_SOURCE + " INTEGER DEFAULT " + SubscriptionManager.NAME_SOURCE_DEFAULT_SOURCE + ","
		+ SubscriptionManager.COLOR + " INTEGER DEFAULT " + SubscriptionManager.COLOR_DEFAULT + ","
		+ SubscriptionManager.NUMBER + " TEXT,"
		+ SubscriptionManager.DISPLAY_NUMBER_FORMAT + " INTEGER NOT NULL DEFAULT " + SubscriptionManager.DISPLAY_NUMBER_DEFAULT + ","
		+ SubscriptionManager.DATA_ROAMING + " INTEGER DEFAULT " + SubscriptionManager.DATA_ROAMING_DEFAULT + ","
		+ SubscriptionManager.MCC + " INTEGER DEFAULT 0,"
		+ SubscriptionManager.MNC + " INTEGER DEFAULT 0,"
		+ SubscriptionManager.CB_EXTREME_THREAT_ALERT + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_SEVERE_THREAT_ALERT + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_AMBER_ALERT + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_EMERGENCY_ALERT + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_ALERT_SOUND_DURATION + " INTEGER DEFAULT 4,"
		+ SubscriptionManager.CB_ALERT_REMINDER_INTERVAL + " INTEGER DEFAULT 0,"
		+ SubscriptionManager.CB_ALERT_VIBRATE + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_ALERT_SPEECH + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_ETWS_TEST_ALERT + " INTEGER DEFAULT 0,"
		+ SubscriptionManager.CB_CHANNEL_50_ALERT + " INTEGER DEFAULT 1,"
		+ SubscriptionManager.CB_CMAS_TEST_ALERT + " INTEGER DEFAULT 0,"
		+ SubscriptionManager.CB_OPT_OUT_DIALOG + " INTEGER DEFAULT 1"
		+ ");");
	if (DBG) log("dbh.createSimInfoTable:-");
}
```

这些字段都表示了什么意思，其中最重要的是_id和sim_id：

- _id：从数据库的角度来说，做过sqlite开发的都知道，他是个从1开始自增的主键。但是他在这里还代表了程序中另一个东西subId也就是subscription id
- icc_id：不解释，上面说过了
- sim_id：这个字段有两层含义，在大于-1，的情况下他表示的是卡槽序号，比如sim_id为0表示卡1，取值为1的时候表示的是卡2，以此类推，但是一般手机不会超过两个卡槽吧？！如果取值为-1，表示这张SIM卡曾经被插入过，但是现在被移除了。
- display_name：顾名思义，显示名。这个一般可以改，但是默认的是读取的运营商的名字，比如：中国移动，中国联通，中国电信
- carrier_name ：恩，运营商名字
- number：SIM卡对应的手机号，这个不一定能取到
- mcc：Mobile Country Code，移动国家码
- mnc:Mobile Network Code，移动网络码

## 获取subscription id

```java
public static void getSimInfo() {

    Uri uri = Uri.parse("content://telephony/siminfo");
    @SuppressLint("Recycle")
    Cursor cursor = Utils.getApp().getContentResolver().query(uri,
            new String[]{"_id", "sim_id", "icc_id", "display_name"}, "0=0",
            new String[]{}, null);
    try {
        if (null != cursor && cursor.moveToFirst()) {
            while (!cursor.isAfterLast()) {
                String icc_id = cursor.getString(cursor.getColumnIndex("icc_id"));
                String display_name = cursor.getString(cursor.getColumnIndex("display_name"));
                int sim_id = cursor.getInt(cursor.getColumnIndex("sim_id"));
                int _id = cursor.getInt(cursor.getColumnIndex("_id"));

                Log.d("getSimInfo", "icc_id-->" + icc_id);
                Log.d("getSimInfo", "sim_id-->" + sim_id);
                Log.d("getSimInfo", "display_name-->" + display_name);
                Log.d("getSimInfo", "subId或者说是_id->" + _id);
                Log.d("getSimInfo", "---------------------------------");
                cursor.moveToNext();
            }
        }
    } catch (Exception e) {
        ExceptionHandle.handleException(e, null);
    } finally {
        if (cursor != null && !cursor.isClosed()) {
            cursor.close();
        }
    }
}
```

如上代码，我在红米6的测试机上进行测试，插入过3张卡，两张移动，一张联通的，运行结果如下（因为是真是的SIM卡，隐藏了icc_id）：

```java
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: icc_id-->898600*************7
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: sim_id-->0
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: display_name-->中国移动
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: subId或者说是_id->1
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: ---------------------------------
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: icc_id-->898601*************6
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: sim_id-->-1
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: display_name-->CARD 2
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: subId或者说是_id->2
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: ---------------------------------
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: icc_id-->898602*************9
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: sim_id-->1
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: display_name-->中国移动
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: subId或者说是_id->3
08-14 16:46:11.208 11583-11583/me.febsky.rootcheck D/Q_M: ---------------------------------
```

[其他的获取subscription id](https://www.jianshu.com/p/1269e4ba99c9)

[获取安卓源码](http://androidxref.com/)

