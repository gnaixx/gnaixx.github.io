title: "Android中Permission的分类"
date: 2015-11-07 10:33:55
categories: android
tags: [android,permission]
toc: true
description: android中权限有分类成不同的等级，每个等级的权限调用方式都不一样。
---
 >android中权限有分类成不同的等级，每个等级的权限调用方式都不一样。
 
###Protection Level
android中Permission主要分为四大类**normal、dangerous、signature、signatureOrSystem**,平常开发中normal、dangerous用到的比较多。

- **normal** ：低风险权限，只要申请了就可以使用   
The default value. A lower-risk permission that gives requesting applications access to isolated application-level features, with minimal risk to other applications, the system, or the user. The system automatically grants this type of permission to a requesting application at installation, without asking for the user's explicit approval (though the user always has the option to review these permissions before installing).

- **dangerous** ：高风险权限，安装时需要用户的确认才可使用   
A higher-risk permission that would give a requesting application access to private user data or control over the device that can negatively impact the user. Because this type of permission introduces potential risk, the system may not automatically grant it to the requesting application. For example, any dangerous permissions requested by an application may be displayed to the user and require confirmation before proceeding, or some other approach may be taken to avoid the user automatically allowing the use of such facilities.

- **signature** 只有当申请权限的应用程序的数字签名与声明此权限的应用程序的数字签名相同时（如果是申请系统权限，则需要与系统签名相同），才能将权限授给它      
A permission that the system grants only if the requesting application is signed with the same certificate as the application that declared the permission. If the certificates match, the system automatically grants the permission without notifying the user or asking for the user's explicit approval.

- **signatureOrSystem** ：签名相同，或者申请权限的应用为系统应用（在system image中）<font color=red>ps:这个不是很理解</font>    
A permission that the system grants only to applications that are in the Android system image or that are signed with the same certificates as those in the system image. Please avoid using this option, as the signature protection level should be sufficient for most needs and works regardless of exactly where applications are installed. The "signatureOrSystem" permission is used for certain special situations where multiple vendors have applications built into a system image and need to share specific features explicitly because they are being built together.

###Protection normal:
**normal**权限不需要特殊的处理，只要在AndroidManifest中注册就可以使用了    

- android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
- android.permission.ACCESS_NETWORK_STATE
- android.permission.ACCESS_NOTIFICATION_POLICY
- android.permission.ACCESS_WIFI_STATE
- android.permission.ACCESS_WIMAX_STATE
- android.permission.BLUETOOTH
- android.permission.BLUETOOTH_ADMIN
- android.permission.BROADCAST_STICKY
- android.permission.CHANGE_NETWORK_STATE
- android.permission.CHANGE_WIFI_MULTICAST_STATE
- android.permission.CHANGE_WIFI_STATE
- android.permission.CHANGE_WIMAX_STATE
- android.permission.DISABLE_KEYGUARD
- android.permission.EXPAND_STATUS_BAR
- android.permission.FLASHLIGHT
- android.permission.GET_PACKAGE_SIZE
- android.permission.INTERNET
- android.permission.KILL_BACKGROUND_PROCESSES
- android.permission.MODIFY_AUDIO_SETTINGS
- android.permission.NFC
- android.permission.READ_SYNC_SETTINGS
- android.permission.READ_SYNC_STATS
- android.permission.RECEIVE_BOOT_COMPLETED
- android.permission.REORDER_TASKS
- android.permission.REQUEST_INSTALL_PACKAGES
- android.permission.SET_TIME_ZONE
- android.permission.SET_WALLPAPER
- android.permission.SET_WALLPAPER_HINTS
- android.permission.SUBSCRIBED_FEEDS_READ
- android.permission.TRANSMIT_IR
- android.permission.USE_FINGERPRINT
- android.permission.VIBRATE
- android.permission.WAKE_LOCK
- android.permission.WRITE_SYNC_SETTINGS
- com.android.alarm.permission.SET_ALARM
- com.android.launcher.permission.INSTALL_SHORTCUT
- com.android.launcher.permission.UNINSTALL_SHORTCUT

###permission groups
权限组中只要申请获取了其中某一个权限，组内其他权限也是默认可用的
<table style="color:#035707" class="table table-bordered table-striped table-condensed">
<tr style="color:white">
<td bgColor=#878787>Permission Groups</th>
<td bgColor=#878787>Permissions</th>
<tr>
<td>android.permission-group.CALENDAR
<td>
<li> android.permission.READ_CALENDAR
<li> android.permission.WRITE_CALENDAR
<tr>
<td>android.permission-group.CAMERA
<td><li>android.permission.CAMERA
<tr>
<td>android.permission-group.CONTACTS
<td>
<li> android.permission.READ_CONTACTS
<li> android.permission.WRITE_CONTACTS
<li> android.permission.GET_ACCOUNTS
<tr>
<td>android.permission-group.LOCATION
<td>
<li> android.permission.ACCESS_FINE_LOCATION
<li> android.permission.ACCESS_COARSE_LOCATION
<tr>
<td>android.permission-group.MICROPHONE
<td><li> android.permission.RECORD_AUDIO
<tr>
<td>android.permission-group.PHONE
<td>
<li> android.permission.READ_PHONE_STATE
<li> android.permission.CALL_PHONE
<li> android.permission.READ_CALL_LOG
<li> android.permission.WRITE_CALL_LOG
<li> com.android.voicemail.permission.ADD_VOICEMAIL
<li> android.permission.USE_SIP
<li> android.permission.PROCESS_OUTGOING_CALLS
<tr>
<td>android.permission-group.SENSORS
<td><li> android.permission.BODY_SENSORS
<tr>
<td>android.permission-group.SMS
<td>
<li> android.permission.SEND_SMS
<li> android.permission.RECEIVE_SMS
<li> android.permission.READ_SMS
<li> android.permission.RECEIVE_WAP_PUSH
<li> android.permission.RECEIVE_MMS
<li> android.permission.READ_CELL_BROADCASTS
<tr>
<td>android.permission-group.STORAGE
<td>
<li> android.permission.READ_EXTERNAL_STORAGE
<li> android.permission.WRITE_EXTERNAL_STORAGE
</table>

####<font color=red>PS:</font>
在我们的项目开发过程中运用到了`android.permission.WRITE_SETTINGS`,该权限是属于**signature**级别的。按照我的理解是系统级别的应用（具有相同签名）才可以使用生效的。   
但是在实际开发中发现Android6.0以前的版本都可以在应用中注册，并且调用

```java
Settings.System.putString();  
Settings.System.getString();
```
进行数据库读写。目前发现编译是设置`targetSdkVersion=23`并且运行Android6.0上无法使用该方法读写。（6.0针对该权限的使用也做了调整）
