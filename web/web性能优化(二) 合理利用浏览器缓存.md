####系列文章包括：
[web性能优化(一) 使用压缩传输](https://github.com/kaelhuawei/blog/blob/master/web/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93.md)  
web性能优化(二)合理利用cache-control(本篇) 
"web性能优化(二)合理利用cache-control")  
web性能优化(三) uglify静态文件&前端工程化(待写)  
web性能优化(四) 合并、删除js和样式表&利用chrome developer tools做页面性能分析(待写)   
***  

####提纲：
* http request和response中缓存相关概念
* tomcat DefaultServlet源码解读，解析tomcat对静态资源的缓存处理策略
* CKM缓存最佳实践  

####http request和response中缓存相关概念
浏览器第一次访问一个网页，会下载页面需要的所有资源。通过网络获取内容既缓慢，成本又高。http1.1(rfc2616)定义了多种缓存方式，可能出现在请求头或者响应头的属性可能有这些：  
request中的：  
* If-Modify-Since: 请求头中带的浏览器缓存里保存的上次响应时保存的文件最后修改时间
* If-None-Match: 请求头中带的浏览器缓存里保存的上次响应时保存的文件ETag  
response中的：  
* Last-Modified: 资源的最后修改时间，注意是文件的修改时间不是创建时间
* Etag: 即Entity Tag，标识一个文件特定版本的字符串，可能是基于文件内容的哈希值或者是其它指纹码，不同服务器实现方式不同
* Expires: 文件的绝对过期时间，在过期前，再次请求同一文件不会和服务端交互，而是直接从缓存里取。Expires属性的行为受Cache-Control属性影响，当响应头里同时又Cache-Control属性，且Cache-Control属性的值有max-age时，max-age优先级大于Expires，会重写Expires的值。Expires因为使用绝对时间，所以它的缺点是需要客户端和服务端保持时间同步，它的优点是在文件过期前和服务端完全没有交互，对于追求性能极致的网站有很大的诱惑力。且Expires属性是http1.0定义的，对于不支持http1.1的浏览器来说很宝贵
* Cache-Control: 缓存控制策略，值可能有public/private max-age=xxxx/no-store/no-cache，public/private定义文件是否允许中继缓存(比如CDN)对其缓存，private仅允许浏览器缓存文件而不允许中继缓存存储文件，public都允许。max-age=xxxx/no-store/no-cache定义文件的缓存时长，max-age定义文件在指定的时间内无需去服务端检查是否有更新，单位是秒；no-store简单粗暴，禁止任何中继缓存和浏览器存储任何响应；no-cache指定浏览器每次都要去服务端检查文件是否有更新。检查更新的途径有多种，第一种是根据文件修改时间，request带If-Modify-Since即上次response中的Last-Modified，去服务端校验文件是否更新；二是根据文件的ETag，request带If-None-Match即上次response中的Etag，去服务端校验文件是否更新。我知道你一定会问request中f-Modify-Since和If-None-Match都有的话，是满足一个服务端就返回304吗？答案是否定的，需要两者都满足才会返回304，这篇文章的下半部分Tomcat DefaultServlet源码解读将从服务端源码角度解释这个问题  
关于ETag需要补充说一下，用还是不用是颇有争议的。ETag的问题一，企业级J2EE一般集群组网，大部分服务器(IIS、Apache)不同的节点对同一文件计算出来的Etag值是不一样的，如果不同次的请求分在不同的服务端，不重复下载文件这个初始的愿望就实现不了。ETag的另一个问题是Etag本身的计算就是一笔开销。对于必须通过最新修改日期之外的一些东西来验证的情况，规避这个问题可以通过简化Etag算法，减少ETag内容，如移除inode值、对于集群组网问题限制负载算法，如同一个sessionId或者source IP的请求落在相同的服务端节点上等。我的建议是根据最新修改日期能cover住的就去除ETag  

好了，主要的概念介绍完了，问题也来了。如何合理利用缓存定义最佳缓存策略来提升web性能？对于不同的站点(企业级J2EE应用、门户网站、电商)策略不应相同，这边文章只分析企业级J2EE应用。  

问题集中在这几个：
* 定义最优Cache-Control策略
* 版本升级，已缓存的文件有修改，如何废弃客户浏览器里已缓存的资源(不要试图让客户手动清浏览器缓存，客户完全可以说我不会也不愿意)
* 精确控制每个文件的缓存策略  

最优Cache-Control策略可以用以下决策树来制定：  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%BA%8C)%20%E5%90%88%E7%90%86%E5%88%A9%E7%94%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E7%BC%93%E5%AD%98/Cache-Control%20decision%20tree.jpg)  

我给CKM制定的缓存策略是：  
* 不启用ETag
* 启用Last-Modified
* html:no-cache,others:Last-Modified,Cache-Control:max-age=1892160000
* 静态资源文件带版本号  

####Tomcat DefaultServlet源码解析  
DefaultServlet是处理静态资源请求的servlet，下面以doGet为例，解析tomcat是如何利用缓存的。  
```java
protected void doGet(HttpServletRequest request, HttpServletResponse response)
    throws IOException, ServletException
{
    serveResource(request, response, true);
}
protected void serveResource(HttpServletRequest request, HttpServletResponse response, boolean content) throws IOException, ServletException
{
    boolean serveContent = content;
    String path = getRelativePath(request);
    if (this.debug > 0) {
    if (serveContent) {
        log("DefaultServlet.serveResource:  Serving resource '" + path + "' headers and data");
    }
    else {
        log("DefaultServlet.serveResource:  Serving resource '" + path + "' headers only");
        }
    }
    CacheEntry cacheEntry = this.resources.lookupCache(path);//从缓存中取出本次请求的资源
```  
缓存的静态文件在容器启动的时候加载，注意attribute属性，默认初始是有文件的lastModified和ETag属性的。  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%BA%8C)%20%E5%90%88%E7%90%86%E5%88%A9%E7%94%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E7%BC%93%E5%AD%98/cacheEntry.jpg)  
```java
if (!cacheEntry.exists)
{
    String requestUri = (String)request.getAttribute("javax.servlet.include.request_uri");
    if (requestUri == null) {
        requestUri = request.getRequestURI();
    }
    else{
        throw new FileNotFoundException(sm.getString("defaultServlet.missingResource", new Object[] { requestUri }));
    }
    response.sendError(404, requestUri);
    return;
}
if ((cacheEntry.context == null) && ((path.endsWith("/")) || (path.endsWith("\\"))))
{
    String requestUri = (String)request.getAttribute("javax.servlet.include.request_uri");
    if (requestUri == null) {
        requestUri = request.getRequestURI();
    }
    response.sendError(404, requestUri);
    return;
}
    boolean isError = response.getStatus() >= 400;
    if (cacheEntry.context == null)
    {
        boolean included = request.getAttribute("javax.servlet.include.context_path") != null;
        if ((!included) && (!isError) && (!checkIfHeaders(request, response, cacheEntry.attributes)))
    {
        return;
    }
}
```  
需要重点看checkIfHeaders(request, response, cacheEntry.attributes)方法。  
```java
protected boolean checkIfHeaders(HttpServletRequest request, HttpServletResponse response, ResourceAttributes resourceAttributes) throws IOException
{
    return (checkIfMatch(request, response, resourceAttributes)) && (checkIfModifiedSince(request, response, resourceAttributes)) && (checkIfNoneMatch(request, response, resourceAttributes)) && (checkIfUnmodifiedSince(request, response, resourceAttributes));
}
```  
这里的checkIfMatch、checkIfModifiedSince、checkIfNoneMatch、checkIfUnmodifiedSince分别是根据请求消息头里的If-Match、If-Modified-Since、If-None-Match、If-Unmodified-Since  
```java
protected boolean checkIfMatch(HttpServletRequest request, HttpServletResponse response, ResourceAttributes resourceAttributes) throws IOException
{
    String eTag = resourceAttributes.getETag();
    String headerValue = request.getHeader("If-Match");
    if ((headerValue != null) && (headerValue.indexOf('*') == -1))//请求头有If-Match，则比较Etag
    {
        StringTokenizer commaTokenizer = new StringTokenizer(headerValue, ",");
        boolean conditionSatisfied = false;
        while ((!conditionSatisfied) && (commaTokenizer.hasMoreTokens())) {
        String currentToken = commaTokenizer.nextToken();
        if (currentToken.trim().equals(eTag)) {
            conditionSatisfied = true;
        }
    }
    if (!conditionSatisfied) {
        response.sendError(412);
        return false;
    }
} 
return true;
}
```  
```java
protected boolean checkIfModifiedSince(HttpServletRequest request, HttpServletResponse response, ResourceAttributes resourceAttributes)
{
    try {
        long headerValue = request.getDateHeader("If-Modified-Since");
        long lastModified = resourceAttributes.getLastModified();
        if (headerValue != -1L)
        {
            if ((request.getHeader("If-None-Match") == null) && (lastModified < headerValue + 1000L))//请求头有If-Modified-Since，则比较浏览器缓存的文件修改时间和服务端缓存的文件修改时间
                {
                    response.setStatus(304);
                    response.setHeader("ETag", resourceAttributes.getETag());
                    return false;
                }
        }
    } catch (IllegalArgumentException illegalArgument) {
        return true;
    }
    return true;
}
```  
```java
protected boolean checkIfNoneMatch(HttpServletRequest request, HttpServletResponse response, ResourceAttributes resourceAttributes) throws IOException
{
    String eTag = resourceAttributes.getETag();
    String headerValue = request.getHeader("If-None-Match");
    if (headerValue != null)
    {
        boolean conditionSatisfied = false;
        if (!headerValue.equals("*"))
        {
            StringTokenizer commaTokenizer = new StringTokenizer(headerValue, ",");
             while ((!conditionSatisfied) && (commaTokenizer.hasMoreTokens())) {
                String currentToken = commaTokenizer.nextToken();
                if (currentToken.trim().equals(eTag)) {//请求头有If-None-Match，则比较浏览器缓存的文件修改时间和服务端缓存的文件修改时间
                    conditionSatisfied = true;
                }
            }
        } else {
            conditionSatisfied = true;
        }
        if (conditionSatisfied)
        {
            if (("GET".equals(request.getMethod())) || ("HEAD".equals(request.getMethod())))
            {
                response.setStatus(304);//如果服务端文件没有更新，返回304
                response.setHeader("ETag", eTag);
                return false;
            }
            response.sendError(412);
            return false;
        }
    }
    return true;
}
```  
```java
protected boolean checkIfUnmodifiedSince(HttpServletRequest request, HttpServletResponse response, ResourceAttributes resourceAttributes) throws IOException
{
    try
    {
        long lastModified = resourceAttributes.getLastModified();
        long headerValue = request.getDateHeader("If-Unmodified-Since");
        if ((headerValue != -1L) && (lastModified >= headerValue + 1000L))//请求头有If-Unmodified-Since，则比较浏览器缓存的文件修改时间和服务端缓存的文件修改时间
        {
            response.sendError(412);//客户端（如您的浏览器或我们的 CheckUpDown 机器人）发送的 HTTP 数据流包括一个没有满足的‘先决条件’规范
            return false;
        }
    }
    catch (IllegalArgumentException illegalArgument) {
        return true;
    }
    return true;
}
```  
继续走serveResource主分支代码：  
```java
String contentType = cacheEntry.attributes.getMimeType();
if (contentType == null) {
    contentType = getServletContext().getMimeType(cacheEntry.name);
    cacheEntry.attributes.setMimeType(contentType);
}
ArrayList<Range> ranges = null;
long contentLength = -1L;
if (cacheEntry.context != null)
{
    if (!this.listings) {
        response.sendError(404, request.getRequestURI());
        return;
    }
    contentType = "text/html;charset=UTF-8";
}
else {
    if (!isError) {
    if (this.useAcceptRanges)
    {
        response.setHeader("Accept-Ranges", "bytes");
    }
    ranges = parseRange(request, response, cacheEntry.attributes);
    response.setHeader("ETag", cacheEntry.attributes.getETag());//在response中设置ETag，这里需要研究一下怎么把tomcat默认的ETag关掉
    response.setHeader("Last-Modified", cacheEntry.attributes.getLastModifiedHttp());//在response中设置Last-modified
}
contentLength = cacheEntry.attributes.getContentLength();
if (contentLength == 0L) {
    serveContent = false;
}
}
``` 
serveResource剩余代码，向response的流中写响应。  

####CKM缓存最佳实践
* disable ETag
```xml
<filter>
		<filter-name>disableETagFilter</filter-name>
		<filter-class>com.huawei.universe.ckm.web.filter.DisableETagFilter</filter-class>
	</filter>
	<filter-mapping>
		<filter-name>disableETagFilter</filter-name>
		<servlet-name>default</servlet-name>
	</filter-mapping>
```  
```java
/*
 * 文 件 名:  DisableETagFilter.java
 * 版    权:  Huawei Technologies Co., Ltd. Copyright 1988-2015,  All rights reserved
 * 描    述:  <描述>
 * 修 改 人:  z00189322
 * 修改时间:  2015年7月10日
 * 跟踪单号:  <跟踪单号>
 * 修改单号:  <修改单号>
 * 修改内容:  <修改内容>
 */
package com.huawei.universe.ckm.web.filter;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletResponseWrapper;

/**
 * 禁用ETag过滤器
 * Cache相关的一系列Filter参见我的性能优化专题文章:http://3ms.huawei.com/hi/blog/61737_1805261.html?h=h
 * 
 * @author z00189322
 * @version [版本号, 2015年7月10日]
 * @see [相关类/方法]
 * @since Universe CKM V200R002C10
 */
public class DisableETagFilter implements Filter
{
    @Override
    public void init(FilterConfig filterConfig) throws ServletException
    {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException
    {
        chain.doFilter(request, new HttpServletResponseWrapper(
                (HttpServletResponse) response)
        {
            @Override
            public void setHeader(String name, String value)
            {
                if (!"ETag".equalsIgnoreCase(name))
                {
                    super.setHeader(name, value);
                }
            }
        });
    }

    @Override
    public void destroy()
    {
    }
}
```  
* tomcat DefaultServlet默认设置了response的Last-Modified头
* 启用Expires、灵活控制Cache-Control
```xml
<filter-name>cacheFilter</filter-name>
		<filter-class>com.huawei.universe.ckm.web.filter.CacheFilter</filter-class>
		<init-param>
			<param-name>expiration</param-name>
			<param-value>31536000</param-value>
		</init-param>
		<!--<init-param>
			<param-name>vary</param-name>
			<param-value>Accept-Encoding</param-value>
		</init-param>
		<init-param>
			<param-name>private</param-name>
			<param-value>true</param-value>
		</init-param>-->
	</filter>
	<filter-mapping>
		<filter-name>cacheFilter</filter-name>
		<url-pattern>*.jpeg</url-pattern>
	</filter-mapping>
	<filter-mapping>
		<filter-name>cacheFilter</filter-name>
		<url-pattern>*.png</url-pattern>
	</filter-mapping>
	<filter-mapping>
		<filter-name>cacheFilter</filter-name>
		<url-pattern>*.css</url-pattern>
	</filter-mapping>
	<filter-mapping>
		<filter-name>cacheFilter</filter-name>
		<url-pattern>*.js</url-pattern>
	</filter-mapping>
```  
```java
/*
 * 文 件 名:  CacheFilter.java
 * 版    权:  Huawei Technologies Co., Ltd. Copyright 1988-2015,  All rights reserved
 * 描    述:  <描述>
 * 修 改 人:  z00189322
 * 修改时间:  2015年7月10日
 * 跟踪单号:  <跟踪单号>
 * 修改单号:  <修改单号>
 * 修改内容:  <修改内容>
 */
package com.huawei.universe.ckm.web.filter;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletResponseWrapper;

/**
 * 设置response中cache相关header的过滤器
 * Cache相关的一系列Filter参见我的性能优化专题文章:http://3ms.huawei.com
 * /hi/blog/61737_1805261.html?h=h
 * 
 * @author z00189322
 * @version [版本号, 2015年7月10日]
 * @see [相关类/方法]
 * @since Universe CKM V200R002C10
 */
public class CacheFilter implements Filter
{
    private long expiration;

    private String cacheability;

    private boolean mustRevalidate;

    private String vary;

    /**
     * @param filterConfig
     * @throws ServletException
     */
    @Override
    public void init(FilterConfig filterConfig) throws ServletException
    {
        try
        {
            expiration = Long.valueOf(filterConfig.getInitParameter("expiration"));
        }
        catch (NumberFormatException e)
        {
            throw new ServletException(new StringBuilder("The initialization parameter ")
                    .append("expiration")
                    .append(" is invalid or is missing for the filter ")
                    .append(filterConfig.getFilterName()).append(".").toString());
        }
        cacheability = Boolean.valueOf(filterConfig.getInitParameter("private")) ? "private"
                : "public";
        mustRevalidate = Boolean
                .valueOf(filterConfig.getInitParameter("must-revalidate"));
        vary = filterConfig.getInitParameter("vary");
    }

    /**
     * @param request
     * @param response
     * @param chain
     * @throws IOException
     * @throws ServletException
     */
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException
    {
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        StringBuilder cacheControl = new StringBuilder(cacheability).append(", max-age=")
                .append(expiration);
        if (mustRevalidate)
        {
            cacheControl.append(", must-revalidate");
        }

        // Set cache directives
        httpServletResponse.setHeader("Cache-Control", cacheControl.toString());
        httpServletResponse.setDateHeader("Expires", System.currentTimeMillis()
                + expiration * 1000L);

        // Set Vary field
        if (vary != null && !vary.isEmpty())
        {
            httpServletResponse.setHeader("Vary", vary);
        }

        /*
         * By default, some servers (e.g. Tomcat) will set headers on any SSL content to deny caching. Omitting the
         * Pragma header takes care of user-agents implementing HTTP/1.0.
         */
        chain.doFilter(request, new HttpServletResponseWrapper(httpServletResponse)
        {
            @Override
            public void addHeader(String name, String value)
            {
                if (!"Pragma".equalsIgnoreCase(name))
                {
                    super.addHeader(name, value);
                }
            }

            @Override
            public void setHeader(String name, String value)
            {
                if (!"Pragma".equalsIgnoreCase(name))
                {
                    super.setHeader(name, value);
                }
            }
        });
    }

    /**
     * 
     */
    @Override
    public void destroy()
    {
    }
}
```  
至此，性能提升已经超过500%，but，优化未完。  
