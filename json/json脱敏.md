## fastjson和jackSon的区别

fastjson是一个阿里开源的一个json处理的框架，主要作用是在对json的处理中实现较快速度的处理，jackSon是国外的一个json处理框架，其相对fastjson可能在执行速度上比不上，但是在兼容性和规范性上更强。

fastjson缺点：虽然速度快，但是不是很符合json的标准，在程序稳定性上有比较大的问题，可能会出现一些莫名其妙的bug

jackSon的缺点：速度不如fastjson，但是没有其他明显的缺点，使用起来可能没有fastjson用起来方便，但是在我们遇到一些奇怪的需求的时候，jackson依旧有提供灵活的api来实现我们的功能

## fastJson实现消息脱敏

解释：自定义一个注解，指定序列化时对应的字段进行的特殊处理，需要自定义一个加密注解、自定义一个脱敏枚举、自定义一个加密处理工具类、自定义一个序列化处理类，使用@Configuration注解指定我们自定义的序列化类

1、创建一个自定义注解

```java
package com.xhqb.oidcadmin.common.core.annotation;

/**
 * @Describe 脱敏注解
 * @Classname SensitivityEncrypt
 * @Date 2023/6/15 11:16
 * @Created by wanghaifeng
 */


import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;

import java.lang.annotation.*;

/**
 * 自定义数据脱敏注解
 */

@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface SensitivityEncrypt {

    /**
     * 脱敏数据类型（必须指定类型）
     */
    SensitivityTypeEnum type();

    /**
     * 前面有多少不需要脱敏的长度
     */
    int prefixNoMaskLen() default 1;

    /**
     * 后面有多少不需要脱敏的长度
     */
    int suffixNoMaskLen() default 1;

    /**
     * 用什么符号进行打码
     */
    String symbol() default "*";
}


```

2、自定义一个枚举类型

```java
package com.xhqb.oidcadmin.common.core.enums;

import lombok.Getter;

/**
 * @Describe 脱敏类型枚举
 * @Classname SensitivityTypeEnum
 * @Date 2023/6/15 11:15
 * @Created by wanghaifeng
 */

@Getter
public enum SensitivityTypeEnum {

    /**
     * 姓名
     */
    NAME,

    /**
     * 身份证号
     */
    ID_CARD,

    /**
     * 邮箱
     */
    EMAIL,

    /**
     * 手机号
     */
    PHONE,

    /**
     *  自定义（此项需设置脱敏的前置后置长度）
     */
    CUSTOMER,
}
```

3、定义一个加密工具类，实现具体的加密逻辑

```
package com.xhqb.oidcadmin.common.core.utils;

/**
 * @Describe
 * @Classname SensitivityUtil
 * @Date 2023/6/15 11:14
 * @Created by wanghaifeng
 */

public class SensitivityUtil {
    /**
     * 【中文姓名】只显示第一个汉字，其他隐藏为星号，比如：才**
     */
    public static String hideChineseName(String chineseName) {
        if (chineseName == null) {
            return null;
        }
        return desValue(chineseName, 1, 0, "*");
    }

    /**
     * 隐藏邮箱
     */
    public static String hideEmail(String email) {
        return email.replaceAll("(\\w?)(\\w+)(\\w)(@\\w+\\.[a-z]+(\\.[a-z]+)?)", "$1****$3$4");
    }

    /**
     * 隐藏手机号中间四位
     */
    public static String hidePhone(String phone) {
        return phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
    }

    /**
     * 隐藏身份证
     */
    public static String hideIDCard(String idCard) {
        return idCard.replaceAll("(\\d{4})\\d{10}(\\w{4})", "$1*****$2");
    }

    /**
     * 对字符串进行脱敏操作
     *
     * @param origin          原始字符串
     * @param prefixNoMaskLen 左侧需要保留几位明文字段
     * @param suffixNoMaskLen 右侧需要保留几位明文字段
     * @param maskStr         用于遮罩的字符串, 如'*'
     * @return 脱敏后结果
     */
    public static String desValue(String origin, int prefixNoMaskLen, int suffixNoMaskLen, String maskStr) {
        if (origin == null) {
            return null;
        }
        StringBuilder sb = new StringBuilder();
        for (int i = 0, n = origin.length(); i < n; i++) {
            if (i < prefixNoMaskLen) {
                sb.append(origin.charAt(i));
                continue;
            }
            if (i > (n - suffixNoMaskLen - 1)) {
                sb.append(origin.charAt(i));
                continue;
            }
            sb.append(maskStr);
        }
        return sb.toString();
    }

}

```

4、序列化处理类，实现fastJson的接口，实现自定义序列化

```java
package com.xhqb.oidcadmin.common.core.serializer;

import com.alibaba.fastjson.serializer.ValueFilter;
import com.xhqb.oidcadmin.common.core.annotation.SensitivityEncrypt;
import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;
import com.xhqb.oidcadmin.common.core.utils.SensitivityUtil;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.lang.reflect.Field;


/**
 * @Describe
 * @Classname SensitivitySerializer
 * @Date 2023/6/15 11:18
 * @Created by wanghaifeng
 */


@NoArgsConstructor
@Slf4j
public class SensitivitySerializer implements ValueFilter {


    @Override
    public Object process(Object object, String name, Object value) {
        if (null == value || !(value instanceof String) || ((String) value).length() == 0) {
            return value;
        }
        try {
            Field field = object.getClass().getDeclaredField(name);
            SensitivityEncrypt sensitivityEncrypt;
            if (String.class != field.getType() || (sensitivityEncrypt = field.getAnnotation(SensitivityEncrypt.class)) == null) {
                return value;
            }
            String valueStr = (String) value;
            SensitivityTypeEnum type = sensitivityEncrypt.type();
            switch (type) {
                case CUSTOMER:
                    return SensitivityUtil.desValue(valueStr, sensitivityEncrypt.prefixNoMaskLen(),
                            sensitivityEncrypt.suffixNoMaskLen(), sensitivityEncrypt.symbol());
                case NAME:
                    return SensitivityUtil.hideChineseName(valueStr);
                case ID_CARD:
                    return SensitivityUtil.hideIDCard(valueStr);
                case PHONE:
                    return SensitivityUtil.hidePhone(valueStr);
                case EMAIL:
                    return SensitivityUtil.hideEmail(valueStr);
                default:
                    throw new IllegalArgumentException("unknown privacy type enum " + sensitivityEncrypt);
            }
        } catch (NoSuchFieldException e) {
            log.error("当前数据类型为{},值为{}", object.getClass(), value);
            return value;
        }
    }
}
```

5、使用上述的序列化处理类会有一个问题：使用该方式，如果对象的属性名称不符合命名规范（驼峰式命名），那么将会导致报错，找不到对应的属性，因此将这一部分代码去除

```java
package com.xhqb.oidcadmin.common.core.serializer;

import com.alibaba.fastjson.serializer.BeanContext;
import com.alibaba.fastjson.serializer.ContextValueFilter;
import com.xhqb.oidcadmin.common.core.annotation.SensitivityEncrypt;
import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;
import com.xhqb.oidcadmin.common.core.utils.SensitivityUtil;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;


/**
 * @Describe
 * @Classname SensitivitySerializer
 * @Date 2023/6/15 11:18
 * @Created by wanghaifeng
 * 差异在于使用的ContextValueFilter，process中使用了BeanContext，避免了在对象中获取对应的属性
 */


@NoArgsConstructor
@Slf4j
public class SensitivitySerializer implements ContextValueFilter {


    @Override
    public Object process(BeanContext context, Object object, String name, Object value) {
        if (null == value || !(value instanceof String) || ((String) value).length() == 0) {
            return value;
        }
        try {
            SensitivityEncrypt sensitivityEncrypt  = context.getAnnation(SensitivityEncrypt.class);
            if ( sensitivityEncrypt == null) {
                return value;
            }
            String valueStr = (String) value;
            SensitivityTypeEnum type = sensitivityEncrypt.type();
            switch (type) {
                case CUSTOMER:
                    return SensitivityUtil.desValue(valueStr, sensitivityEncrypt.prefixNoMaskLen(),
                            sensitivityEncrypt.suffixNoMaskLen(), sensitivityEncrypt.symbol());
                case NAME:
                    return SensitivityUtil.hideChineseName(valueStr);
                case ID_CARD:
                    return SensitivityUtil.hideIDCard(valueStr);
                case PHONE:
                    return SensitivityUtil.hidePhone(valueStr);
                case EMAIL:
                    return SensitivityUtil.hideEmail(valueStr);
                default:
                    throw new IllegalArgumentException("unknown privacy type enum " + sensitivityEncrypt);
            }
        } catch (Exception e) {
            log.error("当前数据类型为{},值为{}", object.getClass(), value);
            return value;
        }
    }
}


```

6、指定fastjson配置，加上我们自定义的序列化类

```java
package com.xhqb.oidcadmin.configuration;

import com.alibaba.fastjson.serializer.SerializerFeature;
import com.alibaba.fastjson.support.config.FastJsonConfig;
import com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter;
import com.xhqb.oidcadmin.common.core.serializer.SensitivitySerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;

import java.util.Arrays;

@Configuration
public class MessageConvertConfiguration {
    @Bean
    public HttpMessageConverter fastJsonHttpMessageConverters() {
        FastJsonHttpMessageConverter fastConverter = new FastJsonHttpMessageConverter();
        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        fastJsonConfig.setDateFormat("yyyy-MM-dd HH:mm:ss");
        fastJsonConfig.setSerializeFilters(new SensitivitySerializer());
        fastJsonConfig.setSerializerFeatures(SerializerFeature.WriteMapNullValue, SerializerFeature.DisableCircularReferenceDetect);
        fastConverter.setFastJsonConfig(fastJsonConfig);
        fastConverter.setSupportedMediaTypes(Arrays.asList(MediaType.APPLICATION_JSON_UTF8));
        return fastConverter;
    }
}

```

7、在实体类上添加对应的注解即可

```java


import com.fasterxml.jackson.annotation.JsonFormat;
import com.xhqb.oidcadmin.common.core.annotation.SensitivityEncrypt;
import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;
import io.swagger.annotations.ApiModelProperty;
import lombok.*;

import java.time.LocalDateTime;

/**
 * @Describe
 * @Classname GetUserLoginLogDetailDto
 * @Date 2023/5/12 17:14
 * @Created by wanghaifeng
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Builder
public class GetUserLoginLogDetailVo {


    @ApiModelProperty("用户手机号码")
    @SensitivityEncrypt(type = SensitivityTypeEnum.PHONE)
    private String phoneNum;
}

```



## jackSon实现消息脱敏

1、方法1：最简单的方式：直接使用@JsonSerialize注解，然后自定义系列化器，优点是简单，问题是一个序列化规则我们就要定义一个对应的序列化器
```Java
// 直接定义一个序列起，@JsonSerialize(using = PhoneJsonSerializer.class)
public class PhoneJsonSerializer extends JsonSerializer {

    @Override
    public void serialize(Object value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        if (value != null && value instanceof String){
            if (!Objects.equals(value,"") && value.toString().length() > 5){
                String phone = value.toString();
                phone = phone.substring(0, 3) + "******" + phone.substring(phone.length() - 2);
                gen.writeString(phone);
            } else {
                gen.writeObject(value);
            }
        } else {
            gen.writeObject(value);
        }
    }
}
```
2、方法2：自定义一个注解，使用这个注解实现脱敏功能，特定类型使用特定脱敏规则，还有普遍脱敏规则
1、创建一个自定义注解

```java
package com.xhqb.oidcadmin.common.core.annotation;

/**
 * @Describe 脱敏注解
 * @Classname SensitivityEncrypt
 * @Date 2023/6/15 11:16
 * @Created by wanghaifeng
 */


import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;

import java.lang.annotation.*;

/**
 * 自定义数据脱敏注解
 */

@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface SensitivityEncrypt {

    /**
     * 脱敏数据类型（必须指定类型）
     */
    SensitivityTypeEnum type();

    /**
     * 前面有多少不需要脱敏的长度
     */
    int prefixNoMaskLen() default 1;

    /**
     * 后面有多少不需要脱敏的长度
     */
    int suffixNoMaskLen() default 1;

    /**
     * 用什么符号进行打码
     */
    String symbol() default "*";
}



```

2、自定义一个枚举类型

```java
package com.xhqb.oidcadmin.common.core.enums;

import lombok.Getter;

/**
 * @Describe 脱敏类型枚举
 * @Classname SensitivityTypeEnum
 * @Date 2023/6/15 11:15
 * @Created by wanghaifeng
 */

@Getter
public enum SensitivityTypeEnum {

    /**
     * 姓名
     */
    NAME,

    /**
     * 身份证号
     */
    ID_CARD,

    /**
     * 邮箱
     */
    EMAIL,

    /**
     * 手机号
     */
    PHONE,

    /**
     *  自定义（此项需设置脱敏的前置后置长度）
     */
    CUSTOMER,
}


```

3、定义一个加密工具类，实现具体的加密逻辑

```java
package com.xhqb.oidcadmin.common.core.utils;

/**
 * @Describe
 * @Classname SensitivityUtil
 * @Date 2023/6/15 11:14
 * @Created by wanghaifeng
 */

public class SensitivityUtil {
    /**
     * 【中文姓名】只显示第一个汉字，其他隐藏为星号，比如：才**
     */
    public static String hideChineseName(String chineseName) {
        if (chineseName == null) {
            return null;
        }
        return desValue(chineseName, 1, 0, "*");
    }

    /**
     * 隐藏邮箱
     */
    public static String hideEmail(String email) {
        return email.replaceAll("(\\w?)(\\w+)(\\w)(@\\w+\\.[a-z]+(\\.[a-z]+)?)", "$1****$3$4");
    }

    /**
     * 隐藏手机号中间四位
     */
    public static String hidePhone(String phone) {
        return phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
    }

    /**
     * 隐藏身份证
     */
    public static String hideIDCard(String idCard) {
        return idCard.replaceAll("(\\d{4})\\d{10}(\\w{4})", "$1*****$2");
    }

    /**
     * 对字符串进行脱敏操作
     *
     * @param origin          原始字符串
     * @param prefixNoMaskLen 左侧需要保留几位明文字段
     * @param suffixNoMaskLen 右侧需要保留几位明文字段
     * @param maskStr         用于遮罩的字符串, 如'*'
     * @return 脱敏后结果
     */
    public static String desValue(String origin, int prefixNoMaskLen, int suffixNoMaskLen, String maskStr) {
        if (origin == null) {
            return null;
        }
        StringBuilder sb = new StringBuilder();
        for (int i = 0, n = origin.length(); i < n; i++) {
            if (i < prefixNoMaskLen) {
                sb.append(origin.charAt(i));
                continue;
            }
            if (i > (n - suffixNoMaskLen - 1)) {
                sb.append(origin.charAt(i));
                continue;
            }
            sb.append(maskStr);
        }
        return sb.toString();
    }

}

```

4、序列化处理类，实现fastJson的接口，实现自定义序列化

```java
package com.xhqb.oidcadmin.common.core.serializer;

import com.alibaba.fastjson.serializer.ValueFilter;
import com.xhqb.oidcadmin.common.core.annotation.SensitivityEncrypt;
import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;
import com.xhqb.oidcadmin.common.core.utils.SensitivityUtil;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.lang.reflect.Field;


/**
 * @Describe
 * @Classname SensitivitySerializer
 * @Date 2023/6/15 11:18
 * @Created by wanghaifeng
 */


@NoArgsConstructor
@AllArgsConstructor
@Slf4j
public class SensitivitySerializer  extends JsonSerializer<String> implements ContextualSerializer  {

    // 脱敏类型
    private SensitivityTypeEnum sensitivityTypeEnum;
    // 前几位不脱敏
    private Integer prefixNoMaskLen;
    // 最后几位不脱敏
    private Integer suffixNoMaskLen;
    // 用什么打码
    private String symbol;

    @Override
    public void serialize(final String origin, final JsonGenerator jsonGenerator,
                          final SerializerProvider serializerProvider) throws IOException {
        switch (SensitivityTypeEnum) {
            case CUSTOMER:
                jsonGenerator.writeString(SensitivityUtil.desValue(origin, prefixNoMaskLen, suffixNoMaskLen, symbol));
                break;
            case NAME:
                jsonGenerator.writeString(SensitivityUtil.hideChineseName(origin));
                break;
            case ID_CARD:
                jsonGenerator.writeString(SensitivityUtil.hideIDCard(origin));
                break;
            case PHONE:
                jsonGenerator.writeString(SensitivityUtil.hidePhone(origin));
                break;
            case EMAIL:
                jsonGenerator.writeString(SensitivityUtil.hideEmail(origin));
                break;
            default:
                throw new IllegalArgumentException("unknown sensitive type enum " + sensitiveTypeEnum);
        }
    }

    @Override
    public JsonSerializer<?> createContextual(final SerializerProvider serializerProvider,
                                              final BeanProperty beanProperty) throws JsonMappingException {
        if (beanProperty != null) {
            if (Objects.equals(beanProperty.getType().getRawClass(), String.class)) {
                SensitivityEncrypt sensitive = beanProperty.getAnnotation(SensitivityEncrypt.class);
                if (sensitive == null) {
                    sensitive = beanProperty.getContextAnnotation(Sensitive.class);
                }
                if (sensitive != null) {
                    return new SensitiveSerialize(sensitive.type(), sensitive.prefixNoMaskLen(),
                            sensitive.suffixNoMaskLen(), sensitive.symbol());
                }
            }
            return serializerProvider.findValueSerializer(beanProperty.getType(), beanProperty);
        }
        return serializerProvider.findNullValueSerializer(null);
    }
}


```


5、在实体类上添加对应的注解即可

```java

package com.xhqb.oidcadmin.common.vo;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.xhqb.oidcadmin.common.core.annotation.SensitivityEncrypt;
import com.xhqb.oidcadmin.common.core.enums.SensitivityTypeEnum;
import io.swagger.annotations.ApiModelProperty;
import lombok.*;

import java.time.LocalDateTime;

/**
 * @Describe
 * @Classname GetUserLoginLogDetailDto
 * @Date 2023/5/12 17:14
 * @Created by wanghaifeng
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Builder
public class GetUserLoginLogDetailVo {

    @ApiModelProperty("用户手机号码")
    @SensitivityEncrypt(type = SensitivityTypeEnum.PHONE)
    private String phoneNum;
}


```