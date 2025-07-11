
## 서블릿 예외 처리
 - 스프링이 아닌 순수 서블릿 컨테이너는 예외를 어떻게 처리하는지 알아보자.
 - 서블릿은 다음과 같이 예외 처리를 지원한다
   - Exception(예외)
        - 자바 직접 실행
          - 자바의 메인 메서드를 직접 실행하는 경우 main 이라는 이름의 쓰레드가 실행된다.
          - 실행도중 예외를 잡지 못하고 처음 실행한 main() 메서드를 넘어서 예외가 던져지면, 예외정보를 남기고 해당 쓰레드는 종료된다.
        - 웹 애플리케이션
          - 사용자 요청별로 별도의 쓰레드가 할당되고, 서블릿 컨테이너 안에서 실행된다.
          - 애플리케이션에서 예외가 발생했는데, 어디선가 try ~ catch로 예외를 처리하면 아무런 문제가 없다
          - 만약 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖까지 예외가 전달되면 어떻게 되는가
            - WAS <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
            - 결국 Tomcat과 같은 WAS까지 예외가 전달된다. WAS 까지 예외가 올라오면 어떻게 처리해야할까
                - 스프링 부트가 제공하는 기본 예외 페이지 설정을 일단 false 처리한다. 
                - server.error.whitelabel.enabled=false
            - Exception의 경우 서버 내부에서 처리할 수 없는것으로 판단하고 HTTP 상태코드 500에러를 발생시킨다.
            - 처리할 수 있는 상태가 아닌경우 404 에러를 발생한다.
   - response.sendError(HTTP 상태코드, 오류메시지)
        - 오류가 발생했을때 HttpServeltReponse 가 제공하는 sendError 라는 메서드를 사용해도된다.
        - 이것을 호출한다고 바로 예외가 발생하는 것은 아니지만, 서블릿 컨테이너에게 오류가 발생했다는 점을 전달할 수 있다.
        - WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())
          - reponse.sendError()를 호출하면 response 내부에는 오류가 발생했다는 상태를 저장해둔다.
          - 그리고 서블릿 컨테이너는 고객에게 응답전에 reponse에 sendError가 호출되었는지 확인한다.
          - 그리고 호출되었다면 설정한 오류코드에 맞추어 기본 오류 페이지를 보여준다.

## 서블릿 예외 처리 - 오류 화면 제공
 - 서블릿이 제공하는 오류 화면 기능을 사용해보자
 - 서블릿은 Exception 이 발생해서 서블릿 밖으로 전달되거나 또는 response.sendError()가 호출되었을때 각각의 상황에 맞춘 오류 처리 기능을 제공한다.
 - errorPage를 등록하면 해당 에러 발생시 등록한 경로의 URL이 다시 호출된다.


## 서블릿 예외처리 - 오류 페이지 작동 원리
 - 서블릿은 Exception, response.sendError()가 호출 되었을때 설정된 오류페이지를 찾는다.
   - WAS <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
   - WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())
 - WAS는 해당 예외를 처리하는오류 페이지 정보를 확인한다.
   - new ErrorPage(RuntimeException.class, "/error-page/500");
     - 예를 들어서 RuntimeException 예외가 WAS까지 전달되면, WAS는 오류페이지 정보를 확인한다
     - RuntimeException의 오류 페이지로 /error-page/500이 지정되어 있고, WAS는 오류페이지를 출력하기 위해 /error-page/500 을 다시 요청한다.
       - WAS /error-page/500 다시 요청 > 필터 > 서블릿 > 인터셉터 > 컨트롤러(/error-page/500)
 - 여기서, 웹브라우저는 서버 내부에서 이런일이 일어나는지 모른다. 오직 서버 내부에서 오류 페이지를 찾기 위해 추가적인 호출을 한다.
 - 정리, 
   - 예외 발생시 WAS 까지 전파
   - WAS는 오류 페이지 경로를 찾아 내부에서 오류페이지를 호출 > 이때 오류 페이지 경로로 필터, 인터셉터, 컨트롤러가 모두 재호출됨

 - 오류 정보 추가
   - WAS는 오류 페이지를 단순히 재 요청하는 것만이 아니라, 오류 정보를 request의 attribute에 추가해서 넘겨준다.


## 서블릿 예외 처리 - 필터
 - 예외처리에 따른 필터와 인터셉터 그리고 서블릿이 제공하는 DispatchType 이해하기
 - 클라이언트로 발생한 정상 요청인지, 오류페이지를 출력하기 위한 내부 요청인지 구분할 수 있어야 한다. 이를 위해 DispatcherType 이라는 추가 정보를 제공한다
 - DispatcherType
   - 오류페이지에서 log.info("dispatcherType={}", request.getDispatcherType()); 을 출력해보면 dispatcherType=ERROR 출력되는 것을 확인 할 수 있다
   - 고객이 처음 요청하면 dispatcherType=REQUEST 이다.
   - 이렇게 서블릿 스펙은 실제 고객이 요청한 것인지, 서버 내부에서 요청하는 것인지는 DispathcerType으로 구분할 수 있는 방법을 제공한다.
     - REQUEST  : 클라이언트 요청
     - ERROR    : 오류요청 
     - FORWARD  : 서블릿에서 다른 서블릿이나 JSP 호출할때 RequestDispatcher.forward(request, response);
     - INCLUDE  : 서블릿에서 다른 서블릿이나 JSP 결과를 포함할때 RequestDispatcher.include(request, response);
     - ASYNC    : 서블릿 비동기 호출
```
    Filter 등록시에 DispatcherType 에 따라 필터가 적용 되고, 말고가 가능함
    setDispatcherTypes() 의 파라미터로 적용되어야할 요청을 넣어주면 됨
    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterFilterRegistrationBean = new FilterRegistrationBean<Filter>();
        filterFilterRegistrationBean.setFilter(new LogFilter());
        filterFilterRegistrationBean.setOrder(1);
        filterFilterRegistrationBean.addUrlPatterns("/*");
        filterFilterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);  // request, error 인경우에 호출됩니다 (LogFilter)
        return filterFilterRegistrationBean;
    }
```


## 서블렛 예외처리 - 인터셉터
 - Filter처럼 DispatcherType 으로 구분하는건 아니고 excludePathPatterns() 를 통해서 컨트롤한다.
```
   @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor())
                .order(1)
                .addPathPatterns("/**")
                .excludePathPatterns("/css/**", "*.icon", "/error", "/error-page/**"); // 오류 페이지 경로를 추가시켜버림

    }
```

 - /hello 정상요청
   - WAS(/hello, dispatcherType=REQUEST) -> 필터 > 서블릿 > 인터셉터 > 컨트롤러 > view
 - /error-ex 요청
   - 필터는 DispatchType 으로 중복 호출 제거(dispatchType=REQUEST)
   - 인터셉터는 경로 정보로 중복 호출 제거(excludePathPatterns("error-page/**))
     - 1. WAS > 필터 > 서블릿 > 인터셉터 > 컨트롤러
     - 2. WAS < 필터 < 서블릿 < 인터셉터 < 컨트롤러(예외발생)
     - 3. WAS 오류페이지 확인
     - 4. WAS(/error-page/500, dispatchType=Error) > 필터(X) > 서블릿 > 인터셉터(X) > 컨트롤러 (/error-page/500) > View

## 스프링 부트 - 오류 페이지 1
 - 지금까지 예외 처리 페이지를 만들기 위해서 복잡한 과정을 거쳤다
 - WebServerCustomizer를 만들고, 예외종류에 따른 ErrorPage를 추가하고, 예외 처리용 컨트롤러 ErrorPageController를 만들었다.
 - 스프링 부트는 이런 과정을 모두 제공해 준다.
   - ErrorPage를 자동으로 등록한다. 이때 /error 라는 경로로 기본 오류 페이지를 설정한다.
   - new ErrorPage("/error") 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용된다.
   - 서블릿 밖으로 예외가 발생하거나, response.sendError() 호출되면 모든 오류는 /error를 호출하게 된다.
   - 이를 처리하기 위해서 스프링부트는 BasicErrorController라는 스프링 컨트롤러를 자동으로 등록한다
   - ErrorPage에서 등록한 /error를 매핑하여 처리하는 컨트롤러이다.
   - ** ErrorMvcAutoConfiguration이라는 클래스가 오류 페이지를 자동으로 등록하는 역할을 한다.
 - 주의사항
   - 스프링 부트가 제공하는 기본 오류 메커니즘을 사용하도록 WebServerCustomizer의 어노테이션을 주석처리한다.
   - 개발자는 오류 페이지 화면만 BasicErrorController가 제공하는 룰과 우선순위에 따라서 등록하면 된다.
   - 정적 HTML이면 정적 리소스, 뷰 템플릿을 사용해서 동적으로 오류화면을 만들고 싶으면 뷰템플릿 경로에 파일을 만들어서 넣어두면된다.

 - 뷰 선택 우선순위 (뷰 템플릿이 정적 리소스보다 우선순위가 높고, 구체적인게 덜 구체적인 것보다 우선순위가 높다, 5xx, 4xx 처리시 500대 400대 오류를 처리해 준다.)
   - 1. 뷰템플릿
    - resources/templates/error/500.html
    - resources/templates/error/5xx.html
   - 2. 정적 리소스(static, pulic)
    - resources/templates/error/400.html
    - resources/templates/error/404.html
    - resources/templates/error/4xx.html
   - 3. 적용 대상이 없을때 뷰이름
    - resources/templates/error.html

## BasicErrorController 가 제공하는 기본 정보들
 - BasicErrorController에서 오류 컨트롤러에서 오류정보를 model에 정보를 포함할지 여부를 선택할 수 있다.
```
    server.error.include-exception=true
    server.error.include-message=on_param
    server.error.include-stacktrace=on_param
    server.error.include-binding-errors=on_param
    
    스프링 부트 오류 관련 옵션
    server.error.whitelabel.enabled=false // 오류 처리 화면을 못찾을시 whitelabel 오류 페이지 적용
    server.error.path=/error // 오류페이지 경로, 스프링이 자동 등록하는 서블릿 글로벌 오류 페이지 경로와 BasicErrorController 오류 컨트롤러 경로에 함꼐 사용된다.
    
    
    
```
 - 에러 공통 처리 컨트롤러의 기능을 변경하고 싶으면 확장포인트 ErrorController 인터페이스를 상속 받아서 구현하거나
 -  BasicErrorCOntroller 상속 받아서 기능을 추가하면된다



## API 예외 처리 - 서블릿 오류 처리
 - API는 각 오류 상황에 맞는 오류 응답 스펙을 정하고 JSON으로 데이터를 내려줘야한다.
 - 클라이언트는 정상 요청이든, 오류 요청이든 JSON이 반환되기를 기대한다. 웹브라우저가 아닌 이상 HTML 받아서 할 수 있는 것은 없다.
 - 문제해결을 하려면 오류 페이지 컨트롤러도 JSON응답을 할 수 있도록 수정해야 한다.
 - @RequestMapping(value = "/error-page/500", produces = MediaType.APPLICATION_JSON_VALUE)

## API 예외 처리 - 스프링부트 기본 오류처리
 - BasicErrorController - error() 에의해서 처리된다.

## API 예외 처리 - HandlerExceptionResolver 시작
 - 500 에러 처리 말고 각각의 상태코드를 처리하고 싶다
 - 상태코드 변환
   - IllegalArgumentException 을 처리하지 못해서 컨트롤러 밖으로 넘어가는 일이 발생하면 HTTP 상태코드를 400으로 처리하고 싶다.
   - HandleExceptionResolver
     - 스프링 MVC는 컨트롤러 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는 방법을 제공한다.
     - 컨트롤러 밖으로 던져진 예외를 해결하고 동작 방식을 변경하고 싶으면 HandlerExceptionResolver를 사용하면 된다.
     - ExceptionResolver라고 한다.
     - DispatcherServlet > ExeceptionResolver > view > afterCompletion() // postHandle()은 호출이 되지 않는다.

 - 빈 ModelAndView > 뷰를 렌더링 하지 않고 정상 흐름으로 서블릿이 리턴된다
 - ModelAndView > View를 렌더링한다.
 - null > 다음 ExceptionResolver를 찾아서 실행한다, 만약 처리할 수 있는 ExceptionResolver가 없을경우 다시 tjqmfflt qkRdmfh ejswlsek.

- ExceptionResolver 활용
  - configureHandlerExceptionResovlers를 오버라이딩 해도되지만 그러면 ExceptionResolver가 삭제되므로 사용하지 말자
 - 예외 상태 코드 변환 >>  상태코드에 따른 오류를 처리하도록 위임할 수 있다  >> 이후 WAS는 서블릿 오류 페이지를 찾아서 내부 호출한다
 - 뷰 템플릿 처리 > ModelAndView에 값을 채워서 예외에 따른 새로운 오류 화면 뷰렌더링 하여 고객에게 제공
 - API 응답 처리 > response.getWriter().println("asddsf : askd"); 처럼 HTTP응답 바디에 직접 데이터를 넣어주는것도 가능하다.


## API 예외 처리 - HandlerExceptionResolver 활용
 - 예외를 마무리하기
 - 예외가 발생하면 WAS까지 예외가 던져지고, WAS에서 오류페이지 정보를 찾아서 다시 /error를 호출하는 과정은 너무 복잡하다
 - ExceptionResolver를 활용하면 예외가 발생했을때 이런 복잡한 과정 없이 문제를 해결할 수 있다.
