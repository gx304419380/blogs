0x0 背景
==
>项目中使用LocalDateTime系列作为dto中时间的类型，但是spring收到参数后总报错，为了全局配置时间类型转换，尝试了如下3中方法。
**注：本文基于Springboot2.0测试，如果无法生效可能是spring版本较低导致的。PS：如果你的Controller中的LocalDate类型的参数啥注解（RequestParam、PathVariable等）都没加，也是会出错的，因为默认情况下，解析这种参数使用```ModelAttributeMethodProcessor```进行处理，而这个处理器要通过反射实例化一个对象出来，然后再对对象中的各个参数进行convert，但是LocalDate类没有构造函数，无法反射实例化因此会报错！！！**

#0x1 当LocalDateTime作为RequestParam或者PathVariable时

>这种情况要和时间作为Json字符串时区别对待，因为前端json转后端pojo底层使用的是Json序列化Jackson工具（HttpMessgeConverter）；而时间字符串作为普通请求参数传入时，转换用的是Converter，两者有区别哦。
在这种情况下，有如下几种方案：
###1. 使用Converter

```
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.http.converter.HttpMessageConverter;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Configuration
public class DateConfig {

    @Bean
    public Converter<String, LocalDate> localDateConverter() {
        return new Converter<>() {
            @Override
            public LocalDate convert(String source) {
                return LocalDate.parse(source, DateTimeFormatter.ofPattern("yyyy-MM-dd"));
            }
        };
    }

    @Bean
    public Converter<String, LocalDateTime> localDateTimeConverter() {
        return new Converter<>() {
            @Override
            public LocalDateTime convert(String source) {
                return LocalDateTime.parse(source, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            }
        };
    }

}
```
>以上两个bean会注入到spring mvc的参数解析器（好像叫做ParameterConversionService），当传入的字符串要转为LocalDateTime类时，spring会调用该Converter对这个入参进行转换。
###2. 使用ControllerAdvice配合initBinder
```
@ControllerAdvice
public class GlobalExceptionHandler {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(LocalDate.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                setValue(LocalDate.parse(text, DateTimeFormatter.ofPattern("yyyy-MM-dd")));
            }
        });
        binder.registerCustomEditor(LocalDateTime.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                setValue(LocalDateTime.parse(text, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
            }
        });
        binder.registerCustomEditor(LocalTime.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                setValue(LocalTime.parse(text, DateTimeFormatter.ofPattern("HH:mm:ss")));
            }
        });
    }
}
```
>从名字就可以看出来，这是在controller做环切（这里面还可以全局异常捕获），在参数进入handler之前进行转换；转换为我们相应的对象。

#0x2 当LocalDateTime作为Json形式传入

>这种情况下，如同上文描述，要利用Jackson的json序列化和反序列化来做：
```
@Configuration
public class JacksonConfig {

    /** 默认日期时间格式 */
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    /** 默认日期格式 */
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    /** 默认时间格式 */
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";


    @Bean
    public ObjectMapper objectMapper(){
        ObjectMapper objectMapper = new ObjectMapper();
//            objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
//            objectMapper.disable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE);
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class,new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        javaTimeModule.addSerializer(LocalDate.class,new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));
        javaTimeModule.addSerializer(LocalTime.class,new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));
        javaTimeModule.addDeserializer(LocalDateTime.class,new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        javaTimeModule.addDeserializer(LocalDate.class,new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));
        javaTimeModule.addDeserializer(LocalTime.class,new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));
        objectMapper.registerModule(javaTimeModule).registerModule(new ParameterNamesModule());
        return objectMapper;
    }

}
```


#0x3 来个完整的配置吧

```
package com.fly.hi.common.config;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalTimeSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;

import java.io.IOException;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.Date;

@Configuration
public class DateConfig {

    /** 默认日期时间格式 */
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    /** 默认日期格式 */
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    /** 默认时间格式 */
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    /**
     * LocalDate转换器，用于转换RequestParam和PathVariable参数
     */
    @Bean
    public Converter<String, LocalDate> localDateConverter() {
        return new Converter<>() {
            @Override
            public LocalDate convert(String source) {
                return LocalDate.parse(source, DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT));
            }
        };
    }

    /**
     * LocalDateTime转换器，用于转换RequestParam和PathVariable参数
     */
    @Bean
    public Converter<String, LocalDateTime> localDateTimeConverter() {
        return new Converter<>() {
            @Override
            public LocalDateTime convert(String source) {
                return LocalDateTime.parse(source, DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT));
            }
        };
    }

    /**
     * LocalTime转换器，用于转换RequestParam和PathVariable参数
     */
    @Bean
    public Converter<String, LocalTime> localTimeConverter() {
        return new Converter<>() {
            @Override
            public LocalTime convert(String source) {
                return LocalTime.parse(source, DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT));
            }
        };
    }

    /**
     * Date转换器，用于转换RequestParam和PathVariable参数
     */
    @Bean
    public Converter<String, Date> dateConverter() {
        return new Converter<>() {
            @Override
            public Date convert(String source) {
                SimpleDateFormat format = new SimpleDateFormat(DEFAULT_DATE_TIME_FORMAT);
                try {
                    return format.parse(source);
                } catch (ParseException e) {
                    throw new RuntimeException(e);
                }
            }
        };
    }


    /**
     * Json序列化和反序列化转换器，用于转换Post请求体中的json以及将我们的对象序列化为返回响应的json
     */
    @Bean
    public ObjectMapper objectMapper(){
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        objectMapper.disable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE);

        //LocalDateTime系列序列化和反序列化模块，继承自jsr310，我们在这里修改了日期格式
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class,new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        javaTimeModule.addSerializer(LocalDate.class,new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));
        javaTimeModule.addSerializer(LocalTime.class,new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));
        javaTimeModule.addDeserializer(LocalDateTime.class,new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        javaTimeModule.addDeserializer(LocalDate.class,new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));
        javaTimeModule.addDeserializer(LocalTime.class,new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));


        //Date序列化和反序列化
        javaTimeModule.addSerializer(Date.class, new JsonSerializer<>() {
            @Override
            public void serialize(Date date, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
                SimpleDateFormat formatter = new SimpleDateFormat(DEFAULT_DATE_TIME_FORMAT);
                String formattedDate = formatter.format(date);
                jsonGenerator.writeString(formattedDate);
            }
        });
        javaTimeModule.addDeserializer(Date.class, new JsonDeserializer<>() {
            @Override
            public Date deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
                SimpleDateFormat format = new SimpleDateFormat(DEFAULT_DATE_TIME_FORMAT);
                String date = jsonParser.getText();
                try {
                    return format.parse(date);
                } catch (ParseException e) {
                    throw new RuntimeException(e);
                }
            }
        });

        objectMapper.registerModule(javaTimeModule);
        return objectMapper;
    }


}

```
#0x4 深入研究SpringMVC数据绑定过程
>接下来进入debug模式，看看mvc是如何将我们request中的参数绑定到我们controller层方法入参的：

写一个简单controller，下个断点看看方法调用栈：
```
    @GetMapping("/getDate")
    public LocalDateTime getDate(@RequestParam LocalDate date,
                                 @RequestParam LocalDateTime dateTime,
                                 @RequestParam Date originalDate) {
        System.out.println(date);
        System.out.println(dateTime);
        System.out.println(originalDate);
        return LocalDateTime.now();
    }
```
断住以后，我们看下方法调用栈中一些关键方法：
```
//进入DispatcherServlet
doService:942, DispatcherServlet
//处理请求
doDispatch:1038, DispatcherServlet
//生成调用链（前处理、实际调用方法、后处理）
handle:87, AbstractHandlerMethodAdapter
//反射获取到实际调用方法，准备开始调用
invokeHandlerMethod:895, RequestMappingHandlerAdapter
invokeAndHandle:102, ServletInvocableHandlerMethod
//这里是关键，参数从这里开始获取到
invokeForRequest:142, InvocableHandlerMethod
doInvoke:215, InvocableHandlerMethod
//这个是Java reflect调用，因此一定是在这之前获取到的参数
invoke:566, Method
```
根据上述分析，发现invokeForRequest:142, InvocableHandlerMethod这里的代码是用来拿到实际参数的：
```
    @Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
        //这个方法是获取参数的，在这里下个断
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
        //这里开始调用方法
		return doInvoke(args);
	}
```
进入这个方法看看是什么操作：
```
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
        //获取方法参数数组，包含了入参信息，比如类型、泛型等等
		MethodParameter[] parameters = getMethodParameters();
        //这个用来存放一会从request parameter转换的参数
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            //这里看起来没啥卵用（providedArgs为空）
			args[i] = resolveProvidedArgument(parameter, providedArgs);
            //这里开始获取到方法实际调用的参数，步进
            if (this.argumentResolvers.supportsParameter(parameter)) {
                //从名字就看出来：参数解析器解析参数
				args[i] = this.argumentResolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
				continue;
			}
		}
		return args;
	}
```
进入resolveArgument看看：
```
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        //根据方法入参，获取对应的解析器
		HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
         //开始解析参数（把请求中的parameter转为方法的入参）
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	}
```
这里根据参数获取相应的参数解析器，看看内部如何获取的：
```
//遍历，调用supportParameter方法，跟进看看
            for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
				if (methodArgumentResolver.supportsParameter(parameter)) {
					result = methodArgumentResolver;
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
```
**这里，遍历参数解析器，查找有没有适合的解析器！那么，有哪些参数解析器呢(我测试的时候有26个)？？？我列出几个重要的看看，是不是很眼熟！！！**
```
{RequestParamMethodArgumentResolver@7686} 
{PathVariableMethodArgumentResolver@8359} 
{RequestResponseBodyMethodProcessor@8366} 
{RequestPartMethodArgumentResolver@8367} 
```
我们进入最常用的一个解析器看看他的supportsParameter方法，发现就是通过参数注解来获取相应的解析器的。
```
	public boolean supportsParameter(MethodParameter parameter) {
        //如果参数拥有注解@RequestParam，则走这个分支（知道为什么上文要对RequestParam和Json两种数据区别对待了把）
		if (parameter.hasParameterAnnotation(RequestParam.class)) {
            //这个似乎是对Optional类型的参数进行处理的
			if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
				RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
				return (requestParam != null && StringUtils.hasText(requestParam.name()));
			}
			else {
				return true;
			}
		}
        //......
	}
```
**也就是说，对于```@RequestParam```和```@RequestBody```以及```@PathVariable```注解的参数，SpringMVC会使用不通的参数解析器进行数据绑定！**
**那么，这三种解析器分别使用什么Converter解析参数呢？我们分别进入三种解析器看一看：**
**首先看下```RequestParamMethodArgumentResolver```发现内部使用WebDataBinder进行数据绑定，底层使用的是ConversionService （也就是我们的Converter注入的地方）**
```

WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
//通过DataBinder进行数据绑定的
arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);

```
```
    //跟进convertIfNecessary()
	public <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType,
			@Nullable MethodParameter methodParam) throws TypeMismatchException {

		return getTypeConverter().convertIfNecessary(value, requiredType, methodParam);
	}

```
```
        //继续跟进，看到了把
		ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
		if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
			TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
			if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
				try {
					return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
				}
				catch (ConversionFailedException ex) {
					// fallback to default conversion logic below
					conversionAttemptEx = ex;
				}
			}
		}

```

**然后看下```RequestResponseBodyMethodProcessor```发现使用的转换器是HttpMessageConverter类型的：**
```
//resolveArgument方法内部调用下面进行参数解析
Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());

//step into readWithMessageConverters()，我们看到这里的Converter是HttpMessageConverter
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
					if (message.hasBody()) {
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
					}
					else {
						body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
					}
					break;
				}
			}
```

**最后看下```PathVariableMethodArgumentResolver```发现 和RequestParam走的执行路径一致（二者都是继承自AbstractNamedValueMethodArgumentResolver解析器），因此代码就不贴了。**


0xFF总结
==
如果要转换request传来的参数到我们指定的类型，根据入参注解要进行区分：
>1. 如果是RequestBody，那么通过配置ObjectMapper（这个玩意儿会注入到Jackson的HttpMessagConverter里面，即```MappingJackson2HttpMessageConverter```中）来实现Json格式数据的序列化和反序列化；
>2. 如果是RequestParam或者PathVariable类型的参数，通过配置Converter实现参数转换（这些Converter会注入到ConversionService中）。




