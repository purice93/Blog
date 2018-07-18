---
title: 微信公众号平台搭建连接javaweb
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
* 先决条件：
	* 一个本地可运行的javaweb：我的是一个网上git的springboot项目	
	* 注册一个微信公众号[申请链接][1]

* 配置前的准备-内网穿透
>由于项目搭建在本地电脑上，外网无法访问，所以需要使用工具将本地地址映射到公网。免费工具使用：natapp
[natapp下载网址][2]
[natapp配置教程][3]
注意：这里只能使用80端口，因为微信公众号只开放80端口使用

**免费隧道配置**
先注册，注册成功后登录。
![enter description here][4] 
![enter description here][5]
隧道购买成功后，在我的隧道中就可以看到已拥有的隧道： 
![enter description here][6]
**客户端下载**
我们访问到natapp的客户端下载，下载natapp客户端： 
![enter description here][7]
下载后，解压，会有一个natapp.exe的文件。

**运行natapp**
在运行natapp之前需要先配置，详细教程参考：[使用本地配置文件config.ini][8]。
config.ini内容： 
![enter description here][9]
注意：config.ini配置文件需要与natapp.exe在同一个目录下。 
![enter description here][10]

**连接**
本地启动javaweb项目后：访问`localhost:80`，
![enter description here][11]
直接双击natapp.exe运行，访问自己的连接`http://wge5gq.natappfree.cc/：80`
![enter description here][12]

注意：由于我这里使用的是springboot，默认端口为8080，所以需要修改配置文件application中的端口配置

* 关闭iis服务器:Internet Information Services
由于win10系统的iis占用80端口，所以导致配置失败，需要关闭
步骤：控制面板 -> 程序 -> 启动或关闭 Windows 功能 -> 取消 'Internet Information Services' 选项 -> 确定 

* 请求验证
>微信公众平台需要发送给服务器一个请求来确认服务器连接
这个需要在项目中添加单独的请求连接
1. 几个参数
![enter description here][13]

2. 请求校验代码编写
	* controller类：HelloWorldController.class
	* 检查接口一致性工具类：CheckoutUtil.class
``` stylus
package cn.ictgu.tools;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/**
 * @author: ZouTai
 * @date: 2018/5/3
 * @description:
 */
public class CheckoutUtil {
    // 与接口配置信息中的Token要一致
    private static String token = "TOKEN";

    /**
     * 验证签名
     *
     * @param signature
     * @param timestamp
     * @param nonce
     * @return
     */
    public static boolean checkSignature(String signature, String timestamp, String nonce) {
        String[] arr = new String[] { token, timestamp, nonce };
        // 将token、timestamp、nonce三个参数进行字典序排序
        // Arrays.sort(arr);
        sort(arr);
        StringBuilder content = new StringBuilder();
        for (int i = 0; i < arr.length; i++) {
            content.append(arr[i]);
        }
        MessageDigest md = null;
        String tmpStr = null;

        try {
            md = MessageDigest.getInstance("SHA-1");
            // 将三个参数字符串拼接成一个字符串进行sha1加密
            byte[] digest = md.digest(content.toString().getBytes());
            tmpStr = byteToStr(digest);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        content = null;
        // 将sha1加密后的字符串可与signature对比，标识该请求来源于微信
        return tmpStr != null ? tmpStr.equals(signature.toUpperCase()) : false;
    }

    /**
     * 将字节数组转换为十六进制字符串
     *
     * @param byteArray
     * @return
     */
    private static String byteToStr(byte[] byteArray) {
        String strDigest = "";
        for (int i = 0; i < byteArray.length; i++) {
            strDigest += byteToHexStr(byteArray[i]);
        }
        return strDigest;
    }

    /**
     * 将字节转换为十六进制字符串
     *
     * @param mByte
     * @return
     */
    private static String byteToHexStr(byte mByte) {
        char[] Digit = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F' };
        char[] tempArr = new char[2];
        tempArr[0] = Digit[(mByte >>> 4) & 0X0F];
        tempArr[1] = Digit[mByte & 0X0F];
        String s = new String(tempArr);
        return s;
    }
    public static void sort(String a[]) {
        for (int i = 0; i < a.length - 1; i++) {
            for (int j = i + 1; j < a.length; j++) {
                if (a[j].compareTo(a[i]) < 0) {
                    String temp = a[i];
                    a[i] = a[j];
                    a[j] = temp;
                }
            }
        }
    }
}



package cn.ictgu.tools;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/**
 * @author: ZouTai
 * @date: 2018/5/3
 * @description:
 */
public class CheckoutUtil {
    // 与接口配置信息中的Token要一致
    private static String token = "TOKEN";

    /**
     * 验证签名
     *
     * @param signature
     * @param timestamp
     * @param nonce
     * @return
     */
    public static boolean checkSignature(String signature, String timestamp, String nonce) {
        String[] arr = new String[] { token, timestamp, nonce };
        // 将token、timestamp、nonce三个参数进行字典序排序
        // Arrays.sort(arr);
        sort(arr);
        StringBuilder content = new StringBuilder();
        for (int i = 0; i < arr.length; i++) {
            content.append(arr[i]);
        }
        MessageDigest md = null;
        String tmpStr = null;

        try {
            md = MessageDigest.getInstance("SHA-1");
            // 将三个参数字符串拼接成一个字符串进行sha1加密
            byte[] digest = md.digest(content.toString().getBytes());
            tmpStr = byteToStr(digest);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        content = null;
        // 将sha1加密后的字符串可与signature对比，标识该请求来源于微信
        return tmpStr != null ? tmpStr.equals(signature.toUpperCase()) : false;
    }

    /**
     * 将字节数组转换为十六进制字符串
     *
     * @param byteArray
     * @return
     */
    private static String byteToStr(byte[] byteArray) {
        String strDigest = "";
        for (int i = 0; i < byteArray.length; i++) {
            strDigest += byteToHexStr(byteArray[i]);
        }
        return strDigest;
    }

    /**
     * 将字节转换为十六进制字符串
     *
     * @param mByte
     * @return
     */
    private static String byteToHexStr(byte mByte) {
        char[] Digit = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F' };
        char[] tempArr = new char[2];
        tempArr[0] = Digit[(mByte >>> 4) & 0X0F];
        tempArr[1] = Digit[mByte & 0X0F];
        String s = new String(tempArr);
        return s;
    }
    public static void sort(String a[]) {
        for (int i = 0; i < a.length - 1; i++) {
            for (int j = i + 1; j < a.length; j++) {
                if (a[j].compareTo(a[i]) < 0) {
                    String temp = a[i];
                    a[i] = a[j];
                    a[j] = temp;
                }
            }
        }
    }
}

```
* 公众号服务器配置
填入URL和代码证对应的token，点击启用
![enter description here][14]
可能的错误：访问超时，token验证错误
验证错误可能是token不一致或者不能访问，需要查看内网穿透是否正确
访问超时，可能是natapp免费使用在某些时候无效，所以可以删除，重新申请免费的，重试 

参考链接：
[内网穿透][15]
[请求验证][16][本文代码来自这里][17]
[关闭iis服务器][18]


  [1]: https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index
  [2]: https://natapp.cn/
  [3]: https://natapp.cn/article/natapp_newbie
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319197567.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319205658.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319242786.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319265044.jpg
  [8]: https://natapp.cn/article/config_ini
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319345823.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319358299.jpg
  [11]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319442871.jpg
  [12]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525319552928.jpg
  [13]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525320128880.jpg
  [14]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525320474857.jpg
  [15]: https://blog.csdn.net/rongxiang111/article/details/78765514
  [16]: https://blog.csdn.net/rongxiang111/article/details/78767733
  [17]: https://blog.csdn.net/u010448530/article/details/72885642
  [18]: https://zhidao.baidu.com/question/219160401.html
