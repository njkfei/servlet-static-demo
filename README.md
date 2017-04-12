# servlet-static-demo
三步打造servlet 静态页方案

# 三步打造servlet 静态页方案

## 1 问题提出
* 搞静态化，nginx是最方便的，但nginx不宜过多介入业务
* 这时候，可以考虑基于servlet打造静态页方案

## 2 原理
* 基于拦截器
* 发现了匹配的utl,先看静态页是否存在且未过期
* 如果存在且未过期，直接使用静态页文件
* 否则，请求后台服务，然后将后台返回的数据，写入静态页文件

## 3 实施步骤
### 3.1 实现静态页快照功能 FileCaptureFilter

```
import com.gemantic.wealth.utils.FileCaptureResponseWrapper;
import org.apache.commons.io.FileUtils;
import java.io.File;
import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class FileCaptureFilter implements Filter {
    private FilterConfig filterConfig;
    private String protDirPath;// 项目绝对路径
    private ServletContext context;
    private String contextPath;
    public void init(FilterConfig filterConfig) throws ServletException {
        this.filterConfig = filterConfig;
        this.context = filterConfig.getServletContext();
        this.protDirPath = filterConfig.getServletContext().getRealPath("/");
        this.contextPath = context.getContextPath();
    }

    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        String path = req.getRequestURI();
        System.out.println(String.format("path : %s",path ));
        if (!path.contains("/hello")) {
            chain.doFilter(request, response);
            return;
        }
        String fileName = protDirPath + path + ".myhtml";
        System.out.println(String.format("file name : %s",fileName));
        File file = new File(fileName);

        // 如果文件存在则直接使用缓存的静态文件，否则保存静态文件再转发,100秒更新
        if (file.exists() && ( System.currentTimeMillis() - file.lastModified() < 100000l) ) {
            System.out.println(String.format("file exist %s",fileName));

            // 设置响应内容类型
            response.setContentType("text/html");
            response.setCharacterEncoding("utf-8");
            String fileContent = FileUtils.readFileToString(file,"UTF-8");

            // 实际的逻辑是在这里
            PrintWriter out = response.getWriter();
            out.print(fileContent.toCharArray());
            out.close();

        } else {
            String filePath = fileName;
            System.out.println(String.format("wirte file : %s",filePath));
            FileCaptureResponseWrapper responseWrapper = new
                    FileCaptureResponseWrapper(
                    (HttpServletResponse) response);
            chain.doFilter(request, responseWrapper);
            // 写成html 文件
            responseWrapper.writeFile(filePath);
        }
    }
    public void destroy() {

    }
}
```

### 3.2 扩展HttpServletResponseWrapper
```
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletResponseWrapper;
import java.io.*;

public class FileCaptureResponseWrapper extends HttpServletResponseWrapper {
    private CharArrayWriter output;
    private HttpServletResponse response;
    private PrintWriter out;

    @Override
    public String toString() {
        return output.toString();
    }

    public FileCaptureResponseWrapper(HttpServletResponse response) {
        super(response);
        this.response = response;
        output = new CharArrayWriter();
    }

    // 覆写getWriter()
    @Override
    public PrintWriter getWriter() {
        return new PrintWriter(output);
    }


    public void writeFile(String fileName) throws IOException {
        FileOutputStream fos = new FileOutputStream(fileName);
        OutputStreamWriter osw = new OutputStreamWriter(fos, "UTF-8");
        osw.write(output.toCharArray());
        osw.close();
    }


    public void writeResponse() throws IOException {
        out = response.getWriter();
// 重置响应输出的内容长度
        response.setContentLength(-1);
        out.print(output.toCharArray());
        out.flush();
        out.close();
    }
    public void close(){
        out.close();
    }
}
```

### 3.3 配置web.xml
```
	<!-- 静态页面拦截器 -->
	<filter>
		<filter-name>FileCaptureFilter</filter-name>
		<filter-class>com.njp.demo.filter.FileCaptureFilter</filter-class>
	</filter>
	<filter-mapping>
		<filter-name>FileCaptureFilter</filter-name>
		<url-pattern>/hellodemo</url-pattern>
	</filter-mapping>
```

### 4 总结
> 来个性能测试吧
