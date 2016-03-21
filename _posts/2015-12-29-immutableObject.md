---
layout: post
title:  "ImmutableObject（不可变对象模型）"
date:   2015-12-29 16:12
categories: java
permalink: /blogs/immutableObject
---
不可变对象模型：
      即：该对象实例创建后它的属性就不能被修改了，除非使用新的对象实例；在更新该不可变对象时使用volatile 关键字，保证多线程下的可见  
               性

/**
 * 彩信中心信息
 *
 * 模式角色：ImmutableObject.ImmutableObject
 */
public final class MMSCInfo {
           private final String deviceID;
           private final String url;
           private final int maxAttachmentSizeInBytes;

           public MMSCInfo(String deviceID, String url, int maxAttachmentSizeInBytes) {
                    this. deviceID = deviceID;
                    this. url = url;
                    this. maxAttachmentSizeInBytes = maxAttachmentSizeInBytes;
          }
          
           public MMSCInfo(MMSCInfo prototype) {
                    this. deviceID = prototype. deviceID;
                    this. url = prototype. url;
                    this. maxAttachmentSizeInBytes = prototype.maxAttachmentSizeInBytes;
          }
          
           public String getDeviceID() {
                    return deviceID;
          }

           public String getUrl() {
                    return url;
          }

           public int getMaxAttachmentSizeInBytes() {
                    return maxAttachmentSizeInBytes;
          }
          
}



/**
*彩信中心路由规则管理器
* 模式角色：ImmutableObject.ImmutableObject
*/
public class MMSCRouter {
    
     private static volatile MMSCRouter instance = new MMSCRouter();
    
     private final Map<String, MMSCInfo> routerMap;
    
     public MMSCRouter(){
          this.routerMap = MMSCRouter.retrieveRouteMapFromDB();
     }

     public static MMSCRouter getInstance(){
          return instance;
     }
    
     private static Map<String, MMSCInfo> retrieveRouteMapFromDB() {
          Map<String, MMSCInfo> map = new HashMap<String, MMSCInfo>();
          // 省略其它代码
          return map;
     }
    
     /**
     * 根据手机号码前缀获取对应的彩信中心信息
     *
     * @param msisdnPrefix
     *          手机号码前缀
     * @return 彩信中心信息
     */
     public MMSCInfo getMMSC(String msisdnPrefix) {
          return routerMap.get(msisdnPrefix);
     }

     /**
     * 将当前MMSCRouter的实例更新为指定的新实例
     *
     * @param newInstance
     *          新的MMSCRouter实例
     */
     public static void setInstance(MMSCRouter newInstance) {
          instance = newInstance;
     }

     private static Map<String, MMSCInfo> deepCopy(Map<String, MMSCInfo> m) {
          Map<String, MMSCInfo> result = new HashMap<String, MMSCInfo>();
          for (String key : m.keySet()) {
               result.put(key, new MMSCInfo(m.get(key)));
          }
          return result;
     }

     public Map<String, MMSCInfo> getRouteMap() {
          //做防御性拷贝
          return Collections.unmodifiableMap(deepCopy(routerMap));
     }
}



public class OMCAgent extends Thread{
          
           @Override
           public void run() {
                    boolean isTableModificationMsg= false;
                   String updatedTableName= null;
            while( true){
                     /**省略其它代码
                     * 从与OMC连接的Socket中读取消息并进行解析,
                     * 解析到数据表更新消息后,重置MMSCRouter实例。
                     */
                     if(isTableModificationMsg){
                              if( "MMSCInfo".equals(updatedTableName)){
                                       MMSCRouter. setInstance(new MMSCRouter());
                             }
                    }
                    
                    MMSCRouter. getInstance().getMMSC("");
                     //省略其它代码
            }
          }

}
