---
title: Android蓝牙通信笔记
layout: post
guid: urn:uuid:0d93c768-1796-44e7-b944-62e19722280b
tags:
  - android
  - bluetooth
---

最近在研究安卓设备直接的蓝牙通信，遇到了很多困难，但最后通过查资料和自己摸索解决了大部分，在此分享一下。  
##基础概念
安卓之间的蓝牙通信主要是基于安卓API里的[Bluetoothadapter](http://developer.android.com/reference/android/bluetooth/BluetoothAdapter.html)并基于[RFCOMM协议](https://developer.bluetooth.org/TechnologyOverview/Pages/RFCOMM.aspx)，scan的部分我就略过不谈了，主要讨论连接的部分。  
##如何建立连接
>>出于避免用户操作的初衷，所有连接都采用了insecure的形式,从而避免用户输入pin码配对  

连接的概念有点类似TCP连接，需要一个Server和一个Client，Server创建serversocket并进行accept()操作，Client创建socket并调用connect()。connect成功则accept()返回一个可以做通信的socket,如果失败Client端会抛出IOException。  
与TCP通信不同的是，蓝牙通信没有port number的概念，取而代之的是UUID。  
Bluetoothserversocket的获取方式是在用**BluetoothAdapter.getDefaultAdapter()**得到BluetoothAdapter之后调用
{% highlight JAVA %}
public BluetoothServerSocket listenUsingInsecureRfcommWithServiceRecord(String name, UUID uuid)
{% endhighlight %}
而Client需要得到Server的BluetoothDevice对象之后调用
{% highlight JAVA %}
public BluetoothSocket createInsecureRfcommSocketToServiceRecord(UUID uuid) 
{% endhighlight %}
BluetoothDevice可以通过BluetoothAdapter的**getRemoteDevice(String address)**方法得到。  
##UUID的问题
运用上面的方法，如果我们只有两台设备，一台做Server一台做Client，只需要他们俩用一样的UUID既可以实现连接。  
**问题来了**，如果我们有多台设备，每台设备需要**同时**做Server和Client应该如何完成呢？  
很多网上的资料都表示，如果A设备需要同时和B、C连接，他需要使用两个不同的UUID，否则将无法成功。然而通过实验，我发现这种说法是**错误**的。  
整个App可以全局只定义一个UUID，在创建serversocket和socket的时候都调用同一个UUID，既可以实现同时多个连接。  
>>**Attention**: 当连接个数过多时，会导致无法预期的结果，包括：新的连接失败、新的连接成功但是与其他设备的连接断开、甚至蓝牙崩溃。  
而『过多』的定义对于不同的设备是不同的，从我测试过的设备来看Nexus 7在超过两个连接后就很容易出现问题，而S4等可以同时有四个以上的连接。  

##简单范例
{% highlight JAVA %}
class ConnectTask implements Runnable {
        BluetoothDevice device;
        BluetoothSocket socket;
        ConnectTask(final BluetoothDevice device) {
            try {
                this.socket = device.createInsecureRfcommSocketToServiceRecord(DEFAULT_SDP_RECORD_UUID1);
                this.device = device;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void run() {
            if (socket == null) {
                return;
            }
            try {
                socket.connect();
                // do your work, e.g. data transmission
            } catch (final IOException e) {
                e.printStackTrace();
                try {
                    socket.close();
                } catch (IOException e2) {
                    e2.printStackTrace();
                }
            }
        }
    }
{% endhighlight %}
{% highlight JAVA %}
class AcceptTask extends Thread {
        bluetoothServerSocket bluetoothServerSocket;
        AcceptTask() {
                try {
                    bluetoothServerSocket = BluetoothAdapter.getDefaultAdapter().listenUsingInsecureRfcommWithServiceRecord(serviceType, DEFAULT_SDP_RECORD_UUID1);
                } catch (IOException e) {
                    e.printStackTrace();
                }
        }

        @Override
        public void run() {
            while (true) {
                BluetoothSocket socket = null;
                try {
                    socket = bluetoothServerSocket.accept(10 * 1000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                if (socket != null && this.isInterrupted()) {
                    try {
                        socket.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    return;
                }

                if (socket != null) {
                    // do your work in another thread e.g. data transmission
                }
            }
        }
    }
{% endhighlight %}
##读写操作
连接成功后的读写操作和TCP基本相同，通过**getInputStream()**和**getOutputStream()**得到InputStream和Outputstream后就可以进行read()和write()。  
连接可以认为是可靠的无需进行error detection，且为full duplex。  
传输速度也是因设备而异，从30kB/s到300kB/s不等，与wifi-direct相比非常慢。