# 简单TCPClient



```java
package com.iflytek.einkcast.socket;


import android.util.Log;

import org.json.JSONException;
import org.json.JSONObject;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.net.SocketException;
import java.net.SocketTimeoutException;
import java.util.Arrays;
import java.util.Date;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * @author hangli2
 * @date 2019/3/7
 * tcp传输客户端 使用阻塞队列处理数据，保证数据发送的时序
 */
public class TCPChannel {
    private String TAG = this.getClass().getName();
    protected String ip;
    protected int port;
    private int timeout = 6000;
    private Socket tcpSocket;
    private OutputStream out;
    private Listener listener;
    private boolean running = false;
    private LinkedBlockingQueue<byte[]> queue;

    private int id = 600000000;
    private boolean useKeepalive = false;

    private static final int ERR_CODE_DEFAULT = -1;


    public TCPChannel(String ip, int port) {
        this.ip = ip;
        this.port = port;
        queue = new LinkedBlockingQueue<>(10);
        running = true;
    }

    public void send(byte[] data, int len) {
        send(data,0,len);
    }

    public void send(byte[] data, int offset, int len) {
        try {
            byte[] bytes = Arrays.copyOfRange(data,offset,offset+len);
            queue.add(bytes);
            //queue.notifyAll();
        }catch (Exception e){
            e.printStackTrace();
            if (listener!=null){
                listener.onError(ERR_CODE_DEFAULT,"add bytes err,queue if full");
            }
        }
    }

    public boolean start(){
        running = true;
        new TCPSendThread().start();
        return true;
    }

    public void stop(){
        running = false;
        try{
            id = 600000000;
            queue.clear();

            if (tcpSocket!=null){
                tcpSocket.close();
                tcpSocket = null;
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }

    public void dispose(){
        stop();
    }

    public String getIp() {
        return ip;
    }

    public int getPort() {
        return port;
    }

    public int getTimeout() {
        return timeout;
    }

    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }

    public void setListener(Listener listener){
        this.listener = listener;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public boolean isUseKeepalive() {
        return useKeepalive;
    }

    public void setUseKeepalive(boolean useKeepalive) {
        this.useKeepalive = useKeepalive;
    }

    public interface Listener {

        void onError(int error,String msg);

        void onMessage(String data);

        void onFrameSend(long time,int size);
    }


    private class TCPSendThread extends Thread{
//        int maxRetryTimes = 5;
        @Override
        public void run() {
//            try {
//                if (ip!=null && port>0 ){
//                    if (tcpSocket==null || tcpSocket.isConnected()){
//                        initSocket();
//                        tcpSocket = new Socket();
//                        tcpSocket.connect(new InetSocketAddress(ip,port),timeout);
//                        out = tcpSocket.getOutputStream();
//                    }
//                }
//            } catch (Exception e) {
//                e.printStackTrace();
//                try {
//                    if (tcpSocket!=null)
//                        tcpSocket.close();
//                } catch (IOException e1) {
//                    e1.printStackTrace();
//                }
//                tcpSocket = null;
//            }
            initSocket();

            while (running){
                try {
                    byte[] bytes = queue.take();
                    boolean result = sendBytes(bytes,0,bytes.length);
                    if (!result && listener!=null){
                        listener.onError(ERR_CODE_DEFAULT,"send bytes error");
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            try {
                if (tcpSocket!=null)
                    tcpSocket.close();
                tcpSocket = null;
            } catch (IOException e) {
                e.printStackTrace();
            }
//            super.run();
        }

        private boolean sendBytes(byte[] data, int offset, int len){
//            int retry = 0;
            //while (++retry<=maxRetryTimes){//重试5次
                try {
                    long startTime = new Date().getTime();
                    if (tcpSocket!=null && tcpSocket.isConnected() && out!=null){
                        if (isUseKeepalive()){
                            JSONObject obj = new JSONObject();
                            try {
                                obj.put("dataLength", len);

                            } catch (JSONException ex) {
                                throw new RuntimeException(ex);
                            }
                            Request req = new Request();
                            req.methodValue = CmdMessage.REQ_DATALENGTH;
                            req.id = id++;
                            req.data = obj;
                            byte[] idFlag = ("==="+req.toJSON()+"===").getBytes();
                            out.write(idFlag);
                            out.flush();
                        }

                        out.write(data,offset,len);
                    }else {
                        initSocket();
//                        if (isUseKeepalive()){
//                            byte[] idFlag = ("==="+(id++)+"===").getBytes();
//                            out.write(idFlag);
//                            out.flush();
//                        }
                        out.write(data,offset,len);
                    }
                    if (listener!=null){
                        long endTime = new Date().getTime();
                        listener.onFrameSend(endTime-startTime,len);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                    if (e.getMessage().equals("Broken pipe")){
                        try {
                            tcpSocket.close();
                        } catch (IOException e1) {
                            e1.printStackTrace();
                        }
                        tcpSocket = null;
                    }
                    return false;
                } catch (Exception e){
                    return false;
                }
            //}
            return true;
        }
    }

    private class TCPReceivedThread extends Thread{
        @Override
        public void run() {
            while (tcpSocket!=null&& tcpSocket.isConnected() && !tcpSocket.isClosed()) {
                //接收socket数据
                try {
                    InputStream is = tcpSocket.getInputStream();
                    byte[] data = new byte[1024];
                    int len = is.read(data);
//                    Log.i(TAG,new String(data,0,len));
                    if (len>0){
                        if (listener!=null){
                            listener.onMessage(new String(data,0,len));
                        }
                    }else {
                        Thread.sleep(1000);
                    }

                } catch (SocketTimeoutException e) {
//                    e.printStackTrace();
                } catch (SocketException e) {
//                    e.printStackTrace();
                    if (tcpSocket == null ||tcpSocket.isClosed()) {
                        break;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private void initSocket() {
        try {
            if (ip != null && port > 0) {
                if (tcpSocket == null || tcpSocket.isConnected()) {
                    tcpSocket = new Socket();
                    tcpSocket.connect(new InetSocketAddress(ip, port), timeout);
                    out = tcpSocket.getOutputStream();

                    new TCPReceivedThread().start();
                    if (isUseKeepalive()){
                        sendBreadth();
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
            try {
                if (tcpSocket != null)
                    tcpSocket.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
            tcpSocket = null;
        }
    }

    /**
     * 内部处理：发送心跳
     */
    private void sendBreadth() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (tcpSocket != null && tcpSocket.isConnected() &&  !tcpSocket.isClosed()) {
                    try {
                        Request req = new Request();
                        req.methodValue = CmdMessage.REQ_KEEPALIVE;
                        req.id = id++;

                        OutputStream os = tcpSocket.getOutputStream();
                        byte[] idFlag = ("===" + req.toJSON() + "===").getBytes();
                        os.write(idFlag);//发送心跳
                        os.flush();
                        Thread.sleep(5000);
                    } catch (IOException | InterruptedException e) {
                        e.printStackTrace();
                        if (e.getMessage().equals("Broken pipe")){
                            break;
                        }
                        if (tcpSocket== null ||tcpSocket.isClosed()) {
                            break;
                        }
                    }
                }
            }
        }).start();
    }

}

```

