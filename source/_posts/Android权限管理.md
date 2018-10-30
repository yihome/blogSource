---
title: Android权限管理
date: 2018-07-11 14:26:22
tags: [Android,permission]
categories: Android
---



## Android动态权限

Android6.0（API23）开始，系统权限出现了很大的变化，此前在权限的检查和获取只发生在app安装时，同时获取后可以一直享有权限。在6.0以后，一些敏感的权限需要动态的获取，同时每次用户可以随时关闭权限，因此需要在每次使用前进行权限检查和获取。

## 权限等级

6.0以后也不是所有权限的获取都需要动态的申请，权限被分成了几个等级，权限的等级主要有四种，分别是Normal、Signature、Dangerous和SignatureOrSystem。

* Normal等级：任何应用都可以申请，直接在manifest声明，安装时提示用户，之后会自动获取此类权限。
* Dangerous等级：任何应用都可以申请，需要在manifest中声明，同时在使用时需要动态的获取权限。
* Signature等级：只有与声明权限的apk使用相同的签名私钥才能申请权限。
* SignatureOrSystem:只有与声明权限的apk使用相同的签名私钥或者在/system/app目录下的应用才能申请权限。

这四种权限都是在manifest中声明权限时指定的，在普通的应用开发中我们一般需要关注的只有Normal和Dangerous两种，其中需要动态申请权限的只有Dangerous权限。

## Dangerous权限

Android下所有的Dangerous权限都被分成权限组，在动态申请权限时，申请的时提示给用户的信息都以组为单位的，比如如果app申请`READ_CONTACTS`权限,那么系统仅仅提示用户申请联系人权限，如果用户允许，在8.0以下的版本中同组的`WRITE_CONTACTS`也会一并被获取到，而在8.0及以上`WRITE_CONTACTS`权限还需要我们去动态的申请，但是系统会立即授予该权限而不会提示用户。

Dangerous权限分组

| 权限组        | 权限                                       |
| ---------- | ---------------------------------------- |
| CALENDAR   | READ_CALENDAR<br/>WRITE_CALENDAR         |
| CAMERA     | CAMERA                                   |
| CONTACTS   | READ_CONTACTS<br/>WRITE_CONTACTS<br/>GET_ACCOUNTS |
| LOCATION   | ACCESS_FINE_LOCATION<br/>ACCESS_COARSE_LOCATION |
| MICROPHONE | RECORD_AUDIO                             |
| PHONE      | READ_PHONE_STATE<br/>READ_PHONE_NUMBERS<br/>CALL_PHONE<br/>ANSWER_PHONE_CALLS<br/>READ_CALL_LOG<br/>WRITE_CALL_LOG<br/>ADD_VOICEMAIL<br/>USE_SIP<br/>PROCESS_OUTGOING_CALLS |
| SENSORS    | BODY_SENSORS                             |
| SMS        | SEND_SMS<br/>RECEIVE_SMS<br/>READ_SMS<br/>RECEIVE_WAP_PUSH<br/>RECEIVE_MMS |
| STORAGE    | READ_EXTERNAL_STORAGE<br/>WRITE_EXTERNAL_STORAGE |



## 动态权限申请

### 申明权限

对于Dangerous的权限，我们首先也需要在manifest中申请，这里以联系人权限为例


    <uses-permission android:name="android.permission.READ_CONTACTS"/>
    <uses-permission android:name="android.permission.WRITE_CONTACTS"/>

### 权限检查

在使用试图操作联系人前需要首先确认有没有这个权限

	if(ContextCompat.checkSelfPermission(this, Manifest.permission.READ_CONTACTS)
	            != PackageManager.PERMISSION_GRANTED){
	    //申请权限    
	}else{
	  	//读取联系人
	}
`ContextCompat.checkSelfPermission(Context context,  String permission)`方法用于检查App是否拥有参数指定的权限，方法的返回值为int，可能的取值如下

* `PERMISSION_GRANTED` 已经获取权限

* `PERMISSION_DENIED` 还没有获取权限


### 权限申请

如果尚未获取权限则在使用之前需要先申请权限

	ActivityCompat.requestPermissions(this,
	                new String[]{Manifest.permission.READ_CONTACTS},CODE_REQUEST_CONTACTS);
一般情况下，`ActivityCompat.requestPermissions`方法会使用一个标准的对话框提示用户，但是有些机型则会跳转到一个新的页面做这件事（会导致Activity声明周期变化，甚至重启）。

申请之后则需要等待用户选择，如果如果你的`Activity`不是继承自`AppCompatActivity`，则需要实现`OnRequestPermissionsResultCallback`用于接收权限请求的结果回掉。

	public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
	    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
	    if(requestCode==CODE_REQUEST_CONTACTS){
	        if(grantResults[0]==PackageManager.PERMISSION_GRANTED){
				//读取联系人
	        }else{
				//权限获取失败
	        }
	    }
	}

### 合理的权限申请

用户首次拒绝授予权限之后，当第二次进行权限请求时，会出现“不再提示”的选项，如果用户勾选，则后续对此权限的申请将不再提示用户，直接失败。因此我们需要在合适的时机解释获取权限的原因，我们可以使用`ActivityCompat.shouldShowRequestPermissionRationale`方法来确定是否需要告知用户，这个方法默认情况下返回false，当用户拒绝过授予此权限后，这个返回值为true（用户勾选不再提示后返回值为false）。

	if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_CONTACTS)
	            != PackageManager.PERMISSION_GRANTED) {
	    if (ActivityCompat.shouldShowRequestPermissionRationale(this,
	                Manifest.permission.READ_CONTACTS)) {
	        //提示用户后申请权限
	    } else {
	        //申请权限
	        ActivityCompat.requestPermissions(this,
	                new String[]{Manifest.permission.READ_CONTACTS}, CODE_REQUEST_CONTACTS);
	    }
	} else {
	    //读取联系人
	}

### 权限被拒绝

权限被拒绝后，再次申请会继续弹出，但是当用户勾选“不再提示”后，下次权限申请系统将不再提示用户，而是直接拒绝，这种情况下，我们只能引导用户到设置里自己打开权限，我们可以在`onRequestPermissionsResult`中通过`ActivityCompat.shouldShowRequestPermissionRationale`来判断用户是否选择了不再提示。

	public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
	    if (requestCode == CODE_REQUEST_CONTACTS) {
	        if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
				//读取联系人
	        } else {
	            if(!ActivityCompat.shouldShowRequestPermissionRationale(this,
	                    Manifest.permission.READ_CONTACTS)){
	               //引导用户到设置页面开启权限
	            }
	        }
	    }
	}
	
	//跳转到设置页面相关代码
	Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
	Uri uri = Uri.fromParts("package",getApplicationContext().getPackageName(), null);
	intent.setData(uri);
	startActivity(intent);


## 参考文章
* [Android 权限的一些细节](https://blog.csdn.net/u013553529/article/details/53167072) 
* [Android中的权限管理](https://blog.csdn.net/jltxgcy/article/details/48288467)
* [Android 6.0 运行时权限处理完全解析](https://blog.csdn.net/lmj623565791/article/details/50709663)
* [Android6.0动态权限申请步骤以及需要注意的一些坑](https://www.jianshu.com/p/a51593817825)

