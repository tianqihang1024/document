```java
package com.lingyun.shop.manager.util;

import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.NameValuePair;
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.message.BasicNameValuePair;
import org.apache.http.util.EntityUtils;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

public class HttpClient {

   public String doPost(String url, Map<String,String> map, String charset){
       CloseableHttpClient httpClient = null;
       HttpPost httpPost = null;
       String result = null;
       try{
           httpClient = HttpClients.createDefault();
           httpPost = new HttpPost(url);
           //设置参数
           List<NameValuePair> list = new ArrayList<NameValuePair>();
           Iterator iterator = map.entrySet().iterator();
           while(iterator.hasNext()){
               Map.Entry<String,String> elem = (Map.Entry<String, String>) iterator.next();
               list.add(new BasicNameValuePair(elem.getKey(),elem.getValue()));
           }
           if(list.size() > 0){
               UrlEncodedFormEntity entity = new UrlEncodedFormEntity(list,charset);
               httpPost.setEntity(entity);
           }
           HttpResponse response = httpClient.execute(httpPost);
           if(response != null){
               HttpEntity resEntity = response.getEntity();
               if(resEntity != null){
                   result = EntityUtils.toString(resEntity,charset);
               }
           }
       }catch(Exception ex){
           ex.printStackTrace();
       }
       return result;
   }
}
```