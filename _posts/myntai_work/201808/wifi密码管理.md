# wifi密码管理

## 背景
 ### 为啥要做密码管理
     机器人项目中，由于多个板子需要通过wifi进行联网。此时只有一个UI界面，为保证其他板子能正常的联网，就不得不，对wifi的SSID和PASSWORD做单独的管理，可是，我们的Android 系统中是不支持用户查询wifi密码的。
 ### 方案
     方案一：UI所在app自行管理
           app中将用户输入的密码，保存在自己的文件中，需要查询密码时，就解析之前保存好的密码。
           弊端：如果用户使用Android系统设置界面的wifi管理界面去连接网络时，就会造成和我们的app中所记录的密码，不一致。这样在我们的实际项目中，就存在bug.
     
     方案二：就用系统自己的密码文件来管理
            Android系统中保存wifi的数据文件是/data/misc/wifi/wpa_supplicant.conf
## 实践
      既然方案一存在，不同步的问题。我么就按照方案二进行管理。
1. 读并且解析wpa_supplicant.conf文件
2. 系统和app进行信息传递

#### wpa_supplicant.conf文件的读和解析
路径：FireNow-Nougat/packages/apps/Settings/src/com/android/settings/wifi
```java
package com.android.settings.wifi;                                                                                                                                                             

import android.util.Log;
import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.InputStreamReader;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Created by colbycao on 18-8-28.
 */

public class SlightechWifiInfo {
    private static final String TAG = "SlightechWifiInfo";

    private HashMap read() throws Exception {
        HashMap wifiInfoMap = new HashMap();
        Process process = null;
        DataOutputStream dataOutputStream = null;
        DataInputStream dataInputStream = null;
        String ssidString = "";
        String password = "";
        StringBuffer wifiConf = new StringBuffer();
        try {
            process = Runtime.getRuntime().exec("su");
            dataOutputStream = new DataOutputStream(process.getOutputStream());
            dataInputStream = new DataInputStream(process.getInputStream());
            dataOutputStream.writeBytes("cat /data/misc/wifi/*.conf\n");
            dataOutputStream.writeBytes("exit\n");
            dataOutputStream.flush();
            InputStreamReader inputStreamReader = new InputStreamReader(
                    dataInputStream, "UTF-8");
            BufferedReader bufferedReader = new BufferedReader(
                    inputStreamReader);
            String line = null;
            while ((line = bufferedReader.readLine()) != null) {
                wifiConf.append(line);
            }
            bufferedReader.close();
            inputStreamReader.close();
            process.waitFor();
        } catch (Exception e) {
            throw e;
        } finally {
            try {
                if (dataOutputStream != null) {
                    dataOutputStream.close();
                }
                if (dataInputStream != null) {
                    dataInputStream.close();
                }
                process.destroy();
            } catch (Exception e) {
                throw e;
            }
        }
        Pattern network = Pattern.compile("network=\\{([^\\}]+)\\}",
                Pattern.DOTALL);
        Matcher networkMatcher = network.matcher(wifiConf.toString());
        while (networkMatcher.find()) {
            String networkBlock = networkMatcher.group();
            Pattern ssid = Pattern.compile("ssid=\"([^\"]+)\"");
            Matcher ssidMatcher = ssid.matcher(networkBlock);
            if (ssidMatcher.find()) {
                ssidString = "" + ssidMatcher.group(1).toString();
                Log.i(TAG, "read: ssidString = " + ssidString);
                Pattern psk = Pattern.compile("psk=\"([^\"]+)\"");
                Matcher pskMatcher = psk.matcher(networkBlock);
                if (pskMatcher.find()) {
                    password = pskMatcher.group(1).toString();
                    Log.i(TAG, "read: password = " + password);
                } else {
                    password = "无密码";
                }
                wifiInfoMap.put(ssidString, password);
            }
        }
        return wifiInfoMap;
    }

    String ssidInquiryPassword(String ssid) {
        String password = "";
        try {
            HashMap wifiInfoHashMap = read();
            password = (String) wifiInfoHashMap.get(ssid);
            if (null == password) {                                                                                                                                                            
                Log.i(TAG, "ssidInquiryPassword: 配置文件中没有记录该SSID信息");
                password = "配置文件中没有记录该SSID信息";
            }
        } catch (Exception e) {
            Log.i(TAG, "ssidInquiryPassword e=" + e);
        }
        return password;
    }


    String savedSsidList() {
        String ssidList = "";
        try {
            HashMap wifiInfoHashMap = read();
            Set<Map.Entry<String, String>> set = wifiInfoHashMap.entrySet();
            for (Map.Entry<String, String> me : set) {
                String key = me.getKey();
                ssidList += key + " ";
            }
        } catch (Exception e) {
            Log.i(TAG, "savedSsidList: e = " + e);
        }
        return ssidList;
    }
}
```

#### 系统和app进行信息传递
路径：FireNow-Nougat/packages/apps/Settings/src/com/android/settings/wifi
```java
package com.android.settings.wifi;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

/**
 * Created by colbycao on 18-8-28.
 */

public class WifiSsidPasswordReceiver extends BroadcastReceiver {

    private static final String TAG = "WifiSsidPasswordReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        if ("com.android.settings.wifi.request.pass".equals(intent.getAction())) {
            Log.i(TAG, "onReceive action==" + intent.getAction());
            String ssid = intent.getStringExtra("SSID");
            String ssidList = intent.getStringExtra("SSIDLIST");
            String password = "" ;

            if (ssidList.toString().equals("SSIDLIST")) {
                ssidList = new SlightechWifiInfo().savedSsidList();
            }else{
                ssidList = "请输入合法关键子查询:[SSIDLIST]";
            }   

            Intent intentReturn = new Intent("com.android.settings.wifi.offer.pass");
            intentReturn.putExtra("SSIDLIST", ssidList);
            intentReturn.putExtra("SSID", ssid);

            password = new SlightechWifiInfo().ssidInquiryPassword(ssid);
            intentReturn.putExtra("PASSWORD", password);
            context.sendBroadcast(intentReturn);
                                                                                                                                                                                               
            Log.i(TAG, "查询 " + ssid + " 密码: " + password);
        }   
    }   
}

```

#### AndroidManifest.xml
路径：FireNow-Nougat/packages/apps/Settings
```xml
       <!-- colby add by wifi ssid password-->
        <receiver
            android:name=".wifi.WifiSsidPasswordReceiver"
            android:exported="true"
            android:enabled="true">
            <intent-filter>
                <action android:name="com.android.settings.wifi.request.pass"/>                                                                                                                
            </intent-filter>
        </receiver>
```

#### APP测试
文件：MainActivity.java
```java
package com.example.colbycao.testsendbroadcast;

import android.content.Intent;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent return_data_intent=new Intent("com.android.settings.wifi.request.pass");
        return_data_intent.putExtra("SSID", "yixia");
        return_data_intent.putExtra("SSIDLIST", "SSIDLIST");
        sendBroadcast(return_data_intent);
    }
}

```

文件：WifiSsidPasswordReturnReceiver.java
```java
package com.example.colbycao.testsendbroadcast;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

/**
 * Created by colbycao on 18-8-28.
 */

public class WifiSsidPasswordReturnReceiver extends BroadcastReceiver {
    private static final String TAG ="WifiSsidPasswordReturnReceiver" ;

    @Override
    public void onReceive(Context context, Intent intent) {
        if("com.android.settings.wifi.offer.pass".equals(intent.getAction())){
            Log.i(TAG, "onReceive action=="+intent.getAction());
            String ssid=intent.getStringExtra("SSID");
            String password = intent.getStringExtra("PASSWORD");
            String ssidList = intent.getStringExtra("SSIDLIST");
            Log.i(TAG, "onReceive: ssid = " + ssid + "  password = " + password + "  ssidList = " + ssidList);

        }
    }
}
```

文件 AndroidManifest.xml
```xml
        <receiver                                                                                                                                                                              
            android:name=".WifiSsidPasswordReturnReceiver"
            android:exported="true"
            android:enabled="true">
            <intent-filter>
                <action android:name="com.android.settings.wifi.offer.pass"/>
            </intent-filter>

        </receiver>>

```

## 总结

1. 有时候我们在读取wifi数据文件的时候，会报空指针的错误，是因为我们没有权限，此时我们应该在安全策略文件中，赋予缺失的权限
2. 读文件类似于命令行的操作。该部分函数调用非常具有参考价值。在以后的工程问题中，我们可以灵活的使用类似的这种方案去做。
3. 我们不应该在信息传递部分，使用广播的方式去处理。理想的是启动一个系统服务去查询。**后期可以改进**