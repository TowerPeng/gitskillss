在项目中,往往要手动设置一个Jetty服务器进行各种参数处理,比如之前在游戏公司,用的就是游戏服内部搭建Jetty服务器,然后方便外部访问.

主要用到这几块.
本身就是Web应用了,还用Jetty干嘛,当然,我这只是做个示例,以后做app或者平台级应用都可以用Jetty搭建外部访问Servlet.
首先,我们设置WebServer,并且设置在监听器里,使得WEB服务器启动的时候可以加载Jetty服务器,

这里是WebServer代码:
package com.dc.web;

/**
 * Date: 2014-2-17
 *
 * Copyright (C) 2013-2015 7Road. All rights reserved.
 */


import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.servlet.Servlet;

import org.dom4j.DocumentException;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.handler.HandlerList;
import org.eclipse.jetty.server.handler.ResourceHandler;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.road.pitaya.web.WebHandleAnnotation;
import com.road.util.ClassUtil;

/**
 * 
 * @author Tower
 */
public class WebServer
{
    private static final Logger LOGGER = LoggerFactory
            .getLogger(WebServer.class);

    /**
     * jetty自带的server
     */
    private Server server;

    /**
     * 触发的处理器
     */
    private ServletContextHandler context;

    /**
     * 处理器列表
     */
    private HandlerList handlerList = new HandlerList();

    /**
     * 触发的资源处理器
     */
    private ResourceHandler resourceHandler;
    
    /**
     * 单例加载器
     */
    private static class LazyHolder
    {
        private static final WebServer INSTANCE = new WebServer();
    }

    /**
     * 获取实例
     * 
     * @return
     */
    public static WebServer getInstance()
    {
        return LazyHolder.INSTANCE;
    }
    
    public boolean start()
    {
        LOGGER.info("WebServer is starting...");
        server = new Server(7039);

        try
        {
            context = new ServletContextHandler(ServletContextHandler.SESSIONS);
            context.setContextPath("/");
            context.setResourceBase("");

            resourceHandler = new ResourceHandler();
            resourceHandler.setResourceBase("/webResource/");

            handlerList.addHandler(context);
            handlerList.addHandler(resourceHandler);

            server.setHandler(handlerList);

            loadServletByWebServerConfig();

            server.start();
        }
        catch (DocumentException e)
        {
            LOGGER.error("load Xml Failed");
            e.printStackTrace();
            return false;
        }
        catch (Exception e)
        {
            e.printStackTrace();
            return false;
        }
        LOGGER.info("WebServer has started successfully");
        return true;
    }
    
    /**
     * 加载Servlet的不同接口
     * 
     * @return
     */
    public Map<String, Class<?>> resetMapHandle()
    {
        Map<String, Class<?>> HandleMap = new HashMap<String, Class<?>>();

        // 从相应的包加载Servlet的不同接口
        List<Class<?>> activityClass = ClassUtil
                .getClasses("com.dc.servlet");

        for (Class<?> class1 : activityClass)
        {
            WebHandleAnnotation annotation = class1
                    .getAnnotation(WebHandleAnnotation.class);
            if (annotation != null)
            {
                HandleMap.put(annotation.cmdName(), class1);
            }
        }

        return HandleMap;
    }

    /**
     * 使用WebServerConfig加载Servlet类
     * 
     * @return
     * @throws DocumentException
     */
    private boolean loadServletByWebServerConfig()
    {

        Map<String, Class<?>> HandleMap = resetMapHandle();
        for (Map.Entry<String, Class<?>> one : HandleMap.entrySet())
        {
            try
            {
                String path = one.getKey();
                Servlet servlet = (Servlet) one.getValue().newInstance();
                context.addServlet(new ServletHolder(servlet), path);
            }
            catch (InstantiationException e)
            {
                e.printStackTrace();
                continue;
            }
            catch (IllegalAccessException e)
            {
                e.printStackTrace();
                continue;
            }
        }
        return true;

    }
    
    public boolean close()
    {
        LOGGER.info("WebServer is closing...");
        try
        {
            server.stop();
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
        LOGGER.info("WebServer has closed successfully");
        return true;
    }

}
设置参数,启动,初始化,这些内容.
package com.dc.servlet;

import java.io.IOException;
import java.io.PrintWriter;
import java.util.Date;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.gson.Gson;
import com.road.entity.info.UserInfo;
import com.road.pitaya.component.LanguageComponent;
import com.road.pitaya.database.DataOption;

/**
 * 中控接口继承用
 * 
 * @author Tower
 */
public abstract class AbstractServlet extends HttpServlet
{
	private static final long serialVersionUID = 2421477169746085074L;
	@SuppressWarnings("unused")
	private Logger LOGGER = LoggerFactory.getLogger(AbstractServlet.class);
    /**
     * 用于处理实体类的Gson实例
     */
    protected final Gson gson = new Gson();
    /** 请求客户端的IP */
    protected String requestIP = null;
    @Override
    public void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException
    {
        doPost(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request,
            HttpServletResponse response) throws ServletException, IOException
    {
    	String result = null;
    	try
        {
    		requestIP = com.dc.util.ServletUtil.getRequestIP(request);
    		result = execute(request, response);
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    	response.setCharacterEncoding("UTF-8");

        response.setContentType("text/html");
        response.setStatus(HttpServletResponse.SC_OK);

        PrintWriter out = response.getWriter();
        out.print(result);
        out.flush();
        out.close();
    }

    public abstract String execute(HttpServletRequest request,
            HttpServletResponse response) throws ServletException, IOException;
    
    /**
	 * 创建一个新的账号
	 * @return
	 */
	protected UserInfo newUserInfo(String site, String iuid, String pssd, Date createTime)
	{
		UserInfo userInfo = new UserInfo();
		userInfo.setOp(DataOption.INSERT);
		userInfo.setSite(site);//所属平台
		userInfo.setIuid(iuid);//账号
		userInfo.setPssd(pssd);//密码
		userInfo.setCreateTime(createTime);//创建时间
		userInfo.setServerId(0);// 账号所在服编号（这里暂不分服）
		return userInfo;
	}
	
	protected String formatTime(long millis)
	{
		int m = (int)(millis/1000);
		int ss = m%60;
		int mm = m/60%60;
		int hh = m/3600%24;
		int dd = m/86400;
		String time = "";
		if(dd > 0)
			time = LanguageComponent.getResource("Login.ServerStatusdd", dd, hh, mm, ss);
		else if(hh > 0)
			time = LanguageComponent.getResource("Login.ServerStatushh", hh, mm, ss);
		else if(mm > 0)
			time = LanguageComponent.getResource("Login.ServerStatusmm", mm, ss);
		else if(ss > 0)
			time = LanguageComponent.getResource("Login.ServerStatusss", ss);
		return time;
	}

    
}


中控接口继承用


然后就是我们测试的Servlet

package com.dc.servlet;

import java.io.IOException;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import com.road.pitaya.web.WebHandleAnnotation;

/**
 * 测试接口
 * @author Tower
 *
 */
@WebHandleAnnotation(cmdName = "/doSomething", description = "测试接口.")
public class DoSomethingServlet extends AbstractServlet{

	/**
	 * 
	 */
	private static final long serialVersionUID = -7574673387670968292L;

	@Override
	public String execute(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
		System.out.println(111111111);
		return null;
	}

}

对于webResources可以看到有个配置文件:
名字叫:crossdomain.xml
从名字可以看出来,这个是跨域策略,方便跨域访问.
<?xml version="1.0"?>
<cross-domain-policy>
  <allow-access-from domain="*" />
</cross-domain-policy>

内容就这些,只是作为实现.

准备就做完了,我们启动tomcat,

Jetty正常启动.
我们之前设置的端口是7039
我们在doSomething设置一个断点,然后我们访问localhost:7039/doSomething 可以看到,断点进去,
控制台输出,于是,我们一个简单的Jetty服务器就搭建完成了,以后可以根据这个Jetty服务器进行其他各项操作.
