## 异步Servlet
&emsp; 定义、映射、注册、服务

* 定义  

	//asyncSupported=true： 异步支持
	@WebServlet(urlPatterns="/servlet/hello", asyncSupported=true)
	public class HelloServlet extends HttpServlet
	
* 映射  
&emsp;Mapping主要有：RequestMapping、GetMapping、PostMapping、PutMapping、DeleteMapping、PatchMapping、
	
	urlPatterns="/servlet/hello"

* 注册  
	
	//Servlet自动装配
	@ServletComponentScan(basePackages="com.mirror.game.servlet")
	
* 服务  
	
	@Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		//开启异步Servlet
        AsyncContext asyncContext = req.startAsync();
        asyncContext.start(() -> {
            try {
                resp.getOutputStream().print("hello,world");
				//异步事件结束
                asyncContext.complete();
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }