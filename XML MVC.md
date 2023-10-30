## .properties -> Memory

properties 파일이 미리 Memory 에 올라 있어야 성능이 더 좋기 때문에 미리 올려두는 작업을 `web.xml 의 init( ) 메서드`에서 진행한다

# web.xml
```
// main 서블릿
<servlet>
	<servlet-name>main</servlet-name>
	<servlet-class>kr.co.seoulit.common.servlet.DispatcherServlet</servlet-class>
	<init-param>
		<param-name>configFile</param-name>
		<param-value>/WEB-INF/config/main.properties</param-value>
	</init-param>
	<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>main</servlet-name>
	<url-pattern>*.do</url-pattern>
</servlet-mapping>
<servlet-mapping>
	<servlet-name>main</servlet-name>
	<url-pattern>*.html</url-pattern>
</servlet-mapping>

// member 서블릿
<servlet>
	<servlet-name>member</servlet-name>
	<servlet-class>kr.co.seoulit.common.servlet.DispatcherServlet</servlet-class>
	<init-param>
		<param-name>configFile</param-name>
		<param-value>/WEB-INF/config/member.properties</param-value>
	</init-param>
	<load-on-startup>2</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>member</servlet-name>
	<url-pattern>/member/*</url-pattern>
</servlet-mapping>

//
<context-param>
	<param-name>urlmappingFile</param-name>
	<param-value>/WEB-INF/config/urlmapping.properties</param-value>
</context-param>
<context-param>
	<param-name>pathFile</param-name>
	<param-value>/WEB-INF/config/path.properties</param-value>
</context-param>
```

- `<servlet-class>kr.co.seoulit.common.servlet.DispatcherServlet</servlet-class>`
	: 두 서블릿의 클래스는 [DispatcherServlet](#DispatcherServlet)

# DispatcherServlet

```java
public class DispatcherServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;
	private ApplicationContext applicationContext;
	private SimpleUrlHandlerMapping simpleUrlHandlerMapping;
	private InternalResourceViewResolver internalResourceViewResolver;
    public DispatcherServlet() {
        // TODO Auto-generated constructor stub
    }
	@Override
	public void init(ServletConfig config) throws ServletException {
		// TODO Auto-generated method stub
		super.init(config);
		ServletContext application=this.getServletContext();
		applicationContext=new ApplicationContext(config,application);
		simpleUrlHandlerMapping=SimpleUrlHandlerMapping.getInstance(application);
		internalResourceViewResolver = InternalResourceViewResolver.getInstance(application);
	}
	
	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		// TODO Auto-generated method stub
		doPost(request,response);
	}

	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		// TODO Auto-generated method stub
		response.setContentType("text/html; charset=UTF-8");
		request.setCharacterEncoding("utf-8");
		
		Controller controller=simpleUrlHandlerMapping.getController(applicationContext,request);
		ModelAndView modelAndView=controller.handleRequest(request,response);
		if(modelAndView!=null)
			internalResourceViewResolver.resolveView(modelAndView,request,response);	
	}
}
```

1. `private static final long serialVersionUID = 1L;`
	: static 선언 먼저 메모리에 올린다
2. `extends HttpServlet`
	: 서블릿을 상속 받아서 `config, application` 과 같은 내장 객체를 사용할 수 있다.
	-> 다른 클래스에 다른 변수에 담아서 이를 매개변수로 전달한다
3. `init(ServletConfig config) throws ServletException`
	- `super.init(config)`
		: `Servlet 클래스`의 init 메서드를 호출하는 것, 여기서 `web.xml <init-param>` 을 가져올 수 있게 된다
	- `ServletContext application=this.getServletContext();`
		: `application` 이란 변수에 `ServletContext` 객체를 담는 것이다
	- `applicationContext=new ApplicationContext(config,application);` [ApplicationContext](#ApplicationContext)
		: `ApplicationContext` 객체에 매개변수로 `config 와 application` 을 던져준다
		`config` = `<init-param>` [init-param](#web.xml)
		`application` = `ServletContext` 객체가 담겨져 있다 
	- `simpleUrlHandlerMapping=SimpleUrlHandlerMapping.getInstance(application);`
		`internalResourceViewResolver = InternalResourceViewResolver.getInstance(application);`
		: 이 둘은 싱글톤 패턴으로 객체를 생성할 필요 없다, -> getInstance() 메서드를 호출하여 `application` 을 던져준다
		공용으로 사용하는 것이기 때문에 config 는 필요 없다
		[SimpleUrlHandlerMapping](#SimpleUrlHandlerMapping)
4. ` doGet(HttpServletRequest request, HttpServletResponse response){ doPost(request,response); }`
	: request 와 response 를 doPost 메서드로 전달한다

5. `Controller controller=simpleUrlHandlerMapping.getController(applicationContext,request);`
	[SimpleUrlHandlerMapping](#SimpleUrlHandlerMapping)
	applicationContext = new ApplicationContext(config,application)
	[getController](#getController) <- 여기 설명되어 있다
	**리턴되는 것** : 클래스 (kr.co.xxx.xxx.mvc.xxx)
6. `ModelAndView modelAndView=controller.handleRequest(request,response);`
	: [ModelAndView](#ModelAndView) 를 타입으로 가진다
	: 가장 많이 나오는 것 같은 [UrlFilenameViewController](#UrlFilenameViewController) 을 대표로 따라가보자
7. `internalResourceViewResolver.resolveView(modelAndView,request,response);`
	: ModelAndView 가 null 이 아니면 위를 실행하게 되는데 `modelAndView(viewname, null)` 로 null 이 아님
	[InternalResourceViewResolver](#InternalResourceViewResolver)
	
# ApplicationContext

- `config 와 application` 는 DispatcherServlet 클래스에서 매개변수로 전달받았다
	config = `<init-param>` application = `ServletContext 객체` [[(77) XMLMVC 보충 개념]]

```java
public class ApplicationContext {
	private HashMap<String,Object> map;
	public ApplicationContext(ServletConfig config, ServletContext application){
		map=new HashMap<String,Object>();
		String path=config.getInitParameter("configFile");
		System.out.println("configFile:"+path);
		String rPath=application.getRealPath(path);
		Properties properties=new Properties();
		try {
			properties.load(new FileReader(rPath));
		} catch (FileNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		Set<String> set=properties.stringPropertyNames();
		for(String key: set){
			String value=properties.getProperty(key);
			Object obj = null;
			try {
				obj = Class.forName(value).newInstance();
			} catch (InstantiationException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} catch (IllegalAccessException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} catch (ClassNotFoundException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			map.put(key,obj);
		}
	}

	public Object getBean(String beanName) {
		// TODO Auto-generated method stub
		return map.get(beanName);
	}
}
```

1. `String path=config.getInitParameter("configFile");`
	: config 를 통해 `<init-param> 의 파라미터 명` 을 통해 값을 path 에 담는다
	= `<param-value>/WEB-INF/config/main.properties</param-value>`
2. `String rPath=application.getRealPath(path);`
	: `getRealPath` 를 통해 path 인 '/WEB-INF/config/main.properties' 절대 경로를 구할 수 있다
	`rPath` 에는 path 의 절대 경로가 담기게 된다
3. `Properties properties=new Properties();`
	: path 의 main.properties 에 접근하기 위해 properties 객체를 생성한다
	- `try { properties.load( new FileReader (rPath) ); }`
		: 'new FileReader(rPath)' 주어진 경로인 rPath 의 파일을 읽기 위해 FileReader 의 객체를 생성한다
		`FileReader` 는 텍스트 파일을 읽기 위한 문자를 기반한 스트림인데, 이 객체를 사용해 파일의 내용을 읽어올 수 있다
		: 'properties.load( )' 메서드를 통해 'FileReader' 객체를 전달하여 properties 객체에 파일의 내용을 로드할 수 있다
4. `Set<String> set=properties.stringPropertyNames();`
	: 변수 set 에 **`properties` 객체에 저장된 모든 속성의 key 를 담는다**
	-> 이를 통해 각 속성에 접근하거나 반복적으로 처리할 수 있다
5. `for(String key: set)`
	: properties 객체에 저장된 모든 속성의 key 를 담아 (배열) 변수 key 에 담는다
	- `String value=properties.getProperty(key);`
		: 'getProperty(key)' key 와 value 를 반환한다
	- `try { obj = Class.forName(value).newInstance(); }`
		: value 값과 이름이 동일한 클래스에 접근할 수 있는 개체를 생성하여 obj 에 담는다
		-> `newInstance()` 메서드는 클래스의 새로운 객체를 생성하고 클래스의 생성자를 호출한 후 생성된 새로운 인스턴스를 반환한다
6. `private HashMap<String,Object> map;` -> `map.put(key,obj);`
	:  map 이라는 변수에 `properties 의 속성` 을 key 값으로 넣고, `이에 상응하는 클래스의 객체` 를 value 값으로 넣는다
7. `public Object getBean(String beanName){ return map.get(beanName); }`
	: [getBean](#getController) (kr.co.xxxxx.mvc1.xxxxx)이다! 
	
	[[(77) XMLMVC 보충 개념]]


# SimpleUrlHandlerMapping

```java
public class SimpleUrlHandlerMapping {
	private HashMap<String,String> map;
	private static SimpleUrlHandlerMapping instance;
	public static SimpleUrlHandlerMapping getInstance(ServletContext application) {
		// TODO Auto-generated method stub
		if(instance==null) instance=new SimpleUrlHandlerMapping(application);
		return instance;
	}
	private SimpleUrlHandlerMapping(ServletContext application){
		map=new HashMap<String,String>();
		String path=application.getInitParameter("urlmappingFile");
		String rPath=application.getRealPath(path);
		Properties properties=new Properties();
		try {
			properties.load(new FileReader(rPath));
		} catch (FileNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		Set<String> set=properties.stringPropertyNames();
		for(String key: set){
			String value=properties.getProperty(key);
			map.put(key,value);
		}
	}
	public Controller getController(ApplicationContext applicationContext, HttpServletRequest request) {
		// TODO Auto-generated method stub
		String uri=request.getRequestURI();
		String contextPath=request.getContextPath();
		int sIndex=contextPath.length(); 
		String key=uri.substring(sIndex);
		System.out.println(key);
		String beanName=map.get(key);
		return (Controller)(applicationContext.getBean(beanName));
	}
}
```

1. `private static SimpleUrlHandlerMapping instance;`
2. `public static SimpleUrlHandlerMapping getInstance(ServletContext application)`
	- `if(instance==null) instance=new SimpleUrlHandlerMapping(application); return instance;`
		: instance 변수가 null 값이면 SimpleUrlHandlerMapping(application) 객체를 생성하고, null 이 아니면 instance 를 반환하면 된다
3. private SimpleUrlHandlerMapping( ServletContext application ){ }
	: DispatcherServlet 에서 전달한 application 을 매개변수로 전달한다
	- `String path=application.getInitParameter("urlmappingFile");`
		: 키 값이 `urlmappingFile` 의 값을 path 에 담는다
		= /WEB-INF/config/urlmapping.properties
	- `String rPath=application.getRealPath(path);`
		: path 의 절대 경로를 rPath 에 담는다
4. `Properties properties=new Properties();`
	: path 의 main.properties 에 접근하기 위해 properties 객체를 생성한다
	- `try { properties.load( new FileReader (rPath) ); }`
		: 'new FileReader(rPath)' 주어진 경로인 rPath 의 파일을 읽기 위해 FileReader 의 객체를 생성한다
		`FileReader` 는 텍스트 파일을 읽기 위한 문자를 기반한 스트림인데, 이 객체를 사용해 파일의 내용을 읽어올 수 있다
		: 'properties.load( )' 메서드를 통해 'FileReader' 객체를 전달하여 properties 객체에 파일의 내용을 로드할 수 있다
5. `Set<String> set=properties.stringPropertyNames();`
	: 변수 set 에 `properties` 객체에 저장된 모든 속성의 key 를 담는다
	-> 이를 통해 각 속성에 접근하거나 반복적으로 처리할 수 있다
6. `for(String key: set)`
	: properties 객체에 저장된 모든 속성의 key 를 담아 (배열) 변수 key 에 담는다
	- `String value=properties.getProperty(key);`
		: 'getProperty(key)' key 와 value 를 반환한다
7. `Controller getController(ApplicationContext applicationContext, HttpServletRequest request)`
	: [사용처 DispatcherServlet](#DispatcherServlet)
	[getController](#getController)


# getController

```java
DispatcherServlet

protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		// TODO Auto-generated method stub
		doPost(request,response);
	}

protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		// TODO Auto-generated method stub
		response.setContentType("text/html; charset=UTF-8");
		request.setCharacterEncoding("utf-8");
		
		Controller controller=simpleUrlHandlerMapping.getController(applicationContext,request);
		ModelAndView modelAndView=controller.handleRequest(request,response);
		if(modelAndView!=null)
			internalResourceViewResolver.resolveView(modelAndView,request,response);	
	}
}

SimpleUrlHandlerMapping

public Controller getController(ApplicationContext applicationContext, HttpServletRequest request) {
		// TODO Auto-generated method stub
		String uri=request.getRequestURI();
		String contextPath=request.getContextPath();
		int sIndex=contextPath.length(); 
		String key=uri.substring(sIndex);
		System.out.println(key);
		String beanName=map.get(key);
		return (Controller)(applicationContext.getBean(beanName));
	}
}
```

- **DispatcherServlet**
	1. applicationContext : new ApplicationContext(config,application)

- **SimpleUrlHandlerMapping**
	1. applicationContext : new ApplicationContext(config,application) (DispatcherServlet 으로부터 전달받음)
	2. key 가 유동적이기 때문에 고정으로 할당할 수 없기 때문에 request.getRequestURL( ) 메서드로 key 를 알아온다
		/menu.html=urlFilenameViewController
		/member/list.do=memberListController
		/error.html=urlFilenameViewController
	3. `String beanName=map.get(key);`
		: beanName 에는 urilFileNameViewController 등과 같은 이름이 들어가게 된다
	4. `return (Controller)(applicationContext.getBean(beanName));`
		: applicationContext 를 타고 `ApplicationContext 의 getBean` 메서드에 beanName 을 전달한다
		(Controller) 로 형변환 시키는 것은 beanName 에 들어가는 것들이 대부분 xxxController
		-> 리턴 되는 것은 각 컨트롤러 (beanName) 에 맞는 value (대충 obj) [(ApplicationContext 에서 getBean 메서드 확인)](#Applicationcontext)


# ModelAndView
 [어디서 왔니](#DispatcherServlet) doPost 메서드의 ModelAndView
```java
DispatcherServlet{
	ModelAndView modelAndView=controller.handleRequest(request,response);
}
```

```java
public class ModelAndView {
	private String viewName;
	private HashMap<String,Object> modelObject;
	public ModelAndView(String viewName,HashMap<String,Object> modelObject){
		this.viewName=viewName;
		this.modelObject=modelObject;
	}
	public String getViewName(){
		return viewName;
	}
	public HashMap<String,Object> getModelObject(){
		return modelObject;
	}
}
```
- viewName = `menu, member/list, error` 
[이동](#DispatcherServlet)
- `getViewName()` 현재 viewName 을 리턴한다 [resolveView](#InternalResourceViewResolver)
- `getModelObject()` 현재 modelObject 를 리턴한다 [if 에서 modelObject](#InternalResourceViewResolver)

 
# UrlFilenameViewController
[어디서 왔니](#DispatcherServlet) doPost 메서드의 controller.handleRequest( request,response )
```java
public class UrlFilenameViewController implements Controller{
@Override
	public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) {
		// TODO Auto-generated method stub
		String uri=request.getRequestURI();
		String contextPath=request.getContextPath();
		int sIndex=contextPath.length()+1;
		int eIndex=uri.lastIndexOf(".");
		String viewName=uri.substring(sIndex,eIndex);
		ModelAndView modelAndView=new ModelAndView(viewName,null);
		return modelAndView;
	}
}
```

- viewName 에는 `menu, member/list, error` 이런식으로 올 수 있다
- `modelAndView=new ModelAndView(viewName,null)`
	: 두번째 인수가 null 이 아니게 될 경우는 DB 를 한 번 거치고 오면 null 이 아니다
	[이동](#ModelAndView) 


# InternalResourceViewResolver
[어디서 왔니](#DispatcherServlet) internalResourceViewResolver.resolveView( modelAndView,request,response );	
```java
public class InternalResourceViewResolver {
	private String prefix,postfix;
	private static InternalResourceViewResolver instance;
	public InternalResourceViewResolver(ServletContext application) {
		// TODO Auto-generated constructor stub
		String path=application.getInitParameter("pathFile");
		String rPath=application.getRealPath(path);
		Properties properties=new Properties();
		try {
			properties.load(new FileReader(rPath));
		} catch (FileNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		prefix=properties.getProperty("prefix");
		postfix=properties.getProperty("postfix");
	}
	public static InternalResourceViewResolver getInstance(ServletContext application) {
		// TODO Auto-generated method stub
		if(instance==null) instance=new InternalResourceViewResolver(application);
		return instance;
	}
	public void resolveView(ModelAndView modelAndView,
			HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException{
		// TODO Auto-generated method stub
		String viewName=modelAndView.getViewName();
		if(viewName.indexOf("redirect:")==-1){
			HashMap<String,Object> modelObject=modelAndView.getModelObject();
			if(modelObject!=null){
				for(String key:modelObject.keySet())
						request.setAttribute(key, modelObject.get(key));
			}
			RequestDispatcher rd=request.getRequestDispatcher(prefix+viewName+postfix);
			rd.forward(request, response);
		}else{
			int index=viewName.indexOf(":");
			String path=viewName.substring(index+1);
			response.sendRedirect(path);
		}
	}
}
```

1. static 먼저
	private static InternalResourceViewResolver instance;
	- InternalResourceViewResolver getInstance(ServletContext application)
		+ `if(instance==null) instance=new InternalResourceViewResolver(application); `
			`return instance;`
			: instance 가 null 이면 지금 해당 클래스 객체 생성을 리턴한다
2. new InternalResourceViewResolver(application)
	- path, rPath 경로 구하기
	- prefix=properties.getProperty("prefix"); , postfix=properties.getProperty("postfix");
		: `ApplicationContext` 클래스의 생성자에서 for 돌아갈 때 properties 의 모든 속성을 다 집어 넣었기 때문에 `prefix 와 postfix` 를 꺼낼 수 있다
3. `resolveView(ModelAndView modelAndView, HttpServletRequest request, HttpServletResponse response)`
	- `String viewName=modelAndView.getViewName();`
		: [getViewName](#ModelAndView) 메서드를 통해 viewName (menu, member/lis, error) 를 가져와서 viewName 에 할당한다
	- `if(viewName.indexOf("redirect:")==-1)` 
		: viewName 에서 redirect: 를 찾지 못했을 경우 -1 을 반환하게 된다
		`HashMap<String,Object> modelObject=modelAndView.getModelObject();`
		: [getModelObject()](#ModelAndView) 의 modelObject 를 담는다
		그리고, if 문을 따라
	- `RequestDispatcher rd=request.getRequestDispatcher(prefix+viewName+postfix);`
		: viewName 과 접두사 접미어를 붙여 rd 에 담는다 -> .jsp 까지 붙이면 다음 화면 완성!
		? RequestDispatcher ?
	- `rd.forward(request, response);`
		: 다음 화면에 request 와 response 를 전달한다

