> @ControllerAdvice注解是Spring3.2中新增的注解，学名是Controller增强器，作用是给Controller控制器添加统一的操作或处理。最常见的是结合@ExceptionHandler注解用于全局异常的处理。

ControllerAdvice拆开来就是Controller Advice，关于Advice，在Spring的AOP中，是用来封装一个切面所有属性的，包括切入点和需要织入的切面逻辑。这里ControllerAdvice也可以这么理解，用于对Controller进行切面环绕的，而具体的业务织入方式则是通过结合其他的注解来实现的。@ControllerAdvice是在类上声明的注解，其用法主要有三点：

1. 结合方法型注解`@ExceptionHandler`，用于捕获Controller中抛出的指定类型的异常，从而达到不同类型的异常区别处理的目的。

2. 结合方法型注解`@InitBinder`，用于request中自定义参数解析方式进行注册，从而达到自定义指定格式参数的目的。

3. 结合方法型注解`@ModelAttribute`，表示其注解的方法将会在目标Controller方法执行之前执行。

# 使用
```java
@ControllerAdvice
public class SpringControllerAdvice {
    /**
     * 应用到所有被@RequestMapping注解的方法，在其执行之前初始化数据绑定器
     * @param binder
     */
    @InitBinder
    public void initBinder(WebDataBinder binder) {}

    /**
     * 把值绑定到Model中，使全局@RequestMapping可以获取到该值
     * @param model
     */
    @ModelAttribute
    public void addAttributes(Model model) {
        model.addAttribute("words", "hello world");
    }

    /**
     * 全局异常捕捉处理
     * @param ex
     * @return
     */
    @ResponseBody
    @ExceptionHandler(value = Exception.class)
    public Map errorHandler(Exception ex) {
        Map map = new HashMap();
        map.put("code", 100);
        map.put("msg", ex.getMessage());
        return map;
    }

}
```

  
在启动应用之后，被`@ExceptionHandler`、`@InitBinder`和`@ModelAttribute`注解的方法都会作用在被`@RequestMappping`注解的方法上。比如上面的`@ModelAttribute`注解的方法参数model上设置的值，所有被`@RequestMapping`注解的方法中都可以通过ModelMap获取。

```java
@RequestMapping("/index")
public String index(ModelMap modelMap) {
    System.out.println(modelMap.get("words"));
}

// 也可以通过@ModelAttribute获取
@RequestMapping("/index")
public String index(@ModelAttribute("words") String words) {
    System.out.println(words);
}
```

# @ControllerAdvice三种使用方式

## @ExceptionHandler拦截异常并统一处理

@ExceptionHandler的作用主要在于声明一个或多个类型的异常，当符合条件的Controller抛出这些异常之后将会对这些异常进行捕获，然后按照其标注的方法的逻辑进行处理，从而改变返回的视图信息。
```java
// Spring原生注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {
    // 指定需要捕获的异常的Class类型
    Class<? extends Throwable>[] value() default {};
}
// 拦截
@ControllerAdvice
public class SpringControllerAdvice {
    @ExceptionHandler(RuntimeException.class)
    public ModelAndView runtimeExceptionHandler(RuntimeException e) {
        e.printStackTrace();
        return new ModelAndView("error");
    }
}

// controller，当访问/users的时候，因为在该方法中抛出了RuntimeException，被拦截器拦截

@Controller
public class UserController {
    @RequestMapping(value = "/users", method = RequestMethod.GET)
    public void users() {
        throw new RuntimeException("没有任何用户。");
    }
}
```

```text
当访问/users的时候，因为在该方法中抛出了RuntimeException，被拦截器拦截
```


## 使用@InitBinder绑定一些自定义参数

对于@InitBinder，该注解的主要作用是绑定一些自定义的参数。一般情况下我们使用的参数通过@RequestParam，@RequestBody或者@ModelAttribute等注解就可以进行绑定了，但对于一些特殊类型参数，比如Date，它们的绑定Spring是没有提供直接的支持的，我们只能为其声明一个转换器，将request中字符串类型的参数通过转换器转换为Date类型的参数，从而供给@RequestMapping标注的方法使用。
```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface InitBinder {
    // 这里value参数用于指定需要绑定的参数名称，如果不指定，则会对所有的参数进行适配，
    // 只有是其指定的类型的参数才会被转换
    String[] value() default {};
}

@ControllerAdvice
public class SpringControllerAdvice {
    @InitBinder
    public void globalInitBinder(WebDataBinder binder) {
	     // 只有是其指定的类型的参数才会被转换
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
  }
}


@Controller
public class UserController {
    @RequestMapping(value = "/users", method = RequestMethod.GET)
    public void users(Date date) {
        System.out.println(date); // Tue May 02 00:00:00 CST 2019
  }
}
```

```text
这里@InitBinder标注的方法注册的Formatter在每次request请求进行参数转换时都会调用，用于判断指定的参数是否为其可以转换的参数。可以看到当访问/users的时候，对request参数进行了转换，并且在接口方法中成功接收了该参数，并在控制台打印出日期格式的结果。
```

## 使用@ModelAttribute在方法执行前进行一些操作

关于@ModelAttribute的用法，除了用于方法参数时可以用于转换对象类型的属性之外，其还可以用来进行方法的声明。如果声明在方法上，并且结合@ControllerAdvice，该方法将会在@ControllerAdvice所指定的范围内的所有接口方法执行==之前执行==，并且@ModelAttribute标注的方法的返回值还可以供给后续会调用的接口方法使用。

```java
@Target({ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ModelAttribute {
    // 该属性与name属性的作用一致，用于指定目标参数的名称
    @AliasFor("name")
    String value() default "";

    @AliasFor("value")
    String name() default "";

    // 与name属性一起使用，如果指定了binding为false，那么name属性指定名称的属性将不会被处理
    boolean binding() default true;
}


@ControllerAdvice
public class SpringControllerAdvice {
    @ModelAttribute(value = "message")
    public String globalModelAttribute() {
        System.out.println("添加了message全局属性。");
        return "输出了message全局属性。";
    }
}

@Controller
public class UserController {
    @RequestMapping(value = "/users", method = RequestMethod.GET)
    public void users(@ModelAttribute("message") String message) {
        System.out.println(message);
  }
}
```

```txt
@ModelAttribute注解中指定了该属性的名称是message，这样在Controller层就可以通过@ModelAttribute注解接收名称为message的参数，从而获取到前面绑定的参数了。

添加了message全局属性。
输出了message全局属性。
```

