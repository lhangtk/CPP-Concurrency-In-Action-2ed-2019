# 简单TCPServer

```java
package com.vflynote.recorder;


import android.util.Log;

import java.io.IOException;
import java.io.InputStream;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * @author hangli2
 * @date 2019/3/7
 * tcp 服务
 */
public class TCPServer {
    private String TAG = this.getClass().getName();
    protected int port;
    private ServerSocket serverSocket;
    private Listener listener;
    private boolean running = false;

    private static final int ERR_CODE_DEFAULT = -1;


    public TCPServer(int port) {
        this.port = port;
        running = false;
    }


    public boolean start(){
        Log.e(TAG, "start: ");
        running = true;
        new TCPReceivedThread().start();
        return true;
    }

    public void stop(){
        Log.e(TAG, "stop: ");
        running = false;
        try{

            if (serverSocket!=null){
                serverSocket.close();
                serverSocket = null;
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }

    public void setListener(Listener listener){
        this.listener = listener;
    }


    public interface Listener {

        void onError(int error, String msg);

        void onMessage(byte[] data,int offset,int len);

        void onFrameSend(long time, int size);
    }


    private class TCPReceivedThread extends Thread {
        @Override
        public void run() {
            initSocket();
            Log.e(TAG, "run: stop" );
        }
    }

    private int count = 0;

    private void initSocket() {
        try {
            if (serverSocket == null){
                serverSocket = new ServerSocket(port);
                //循环监听等待客户端的连接
                Log.e(TAG,"S: Receiving...");
                while(running){
                    //调用accept()方法开始监听，等待客户端的连接
                    Socket socket=serverSocket.accept();
                    //创建一个新的线程
                    ServerThread serverThread=new ServerThread(socket);
                    //启动线程
                    serverThread.start();

                    count++;//统计客户端的数量
                    System.out.println("客户端的数量："+count);
                    InetAddress address=socket.getInetAddress();
                    System.out.println("当前客户端的IP："+address.getHostAddress());
                }
                Log.e(TAG, "run: stop" );
            }

        } catch (Exception e) {
            e.printStackTrace();
            try {
                if (serverSocket != null)
                    serverSocket.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
            serverSocket = null;
        }
    }
    public class ServerThread extends Thread {
        // 和本线程相关的Socket
        Socket socket = null;

        public ServerThread(Socket socket) {
            this.socket = socket;
        }

        //线程执行的操作，响应客户端的请求
        public void run(){
            InputStream is=null;
            try {
                //获取输入流，并读取客户端信息
                is = socket.getInputStream();
//                isr = new InputStreamReader(is);
//                br = new BufferedReader(isr);
//                String info=null;
                while (socket!=null&&socket.isConnected()&&running){
                    byte[] data = new byte[4096];
                    int len = is.read(data);
                    if (len > 0) {
                        if (listener != null && running) {
                            listener.onMessage(data, 0, len);
                            System.out.println(len);
                        }
                    }
                    Thread.sleep(40);
                }
                Log.e(TAG, "ServerThread: stop");

//                while((info=br.readLine())!=null){//循环读取客户端的信息
//                    System.out.println("我是服务器，客户端说："+info);
//                }
                socket.shutdownInput();//关闭输入流
                //获取输出流，响应客户端的请求
//                os = socket.getOutputStream();
//                pw = new PrintWriter(os);
//                pw.write("欢迎您！");
//                pw.flush();//调用flush()方法将缓冲输出
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally{
                //关闭资源
                try {
                    if(is!=null)
                        is.close();
                    if(socket!=null)
                        socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public boolean isRunning() {
        return running;
    }
}


```

