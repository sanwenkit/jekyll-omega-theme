---
layout: post
title: swagger+spring mvc文档自动生成
description: "记录自动生成接口文档的配置"
category: develop
tags: [develop]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---
#

# swagger+spring mvc文档自动生成

### pom文件依赖配置
在pom文件中添加如下依赖：

{% highlight xml %}

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.5.0</version>
            <exclusions>
                <exclusion>
                    <groupId>com.fasterxml.jackson.core</groupId>
                    <artifactId>jackson-annotations</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring-aop</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring-beans</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring-context</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.5.0</version>
        </dependency>
        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-annotations</artifactId>
            <version>1.5.9</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-staticdocs</artifactId>
            <version>2.5.0</version>
            <exclusions>
                <exclusion>
                    <groupId>com.fasterxml.jackson.core</groupId>
                    <artifactId>jackson-annotations</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.fasterxml.jackson.dataformat</groupId>
                    <artifactId>jackson-dataformat-yaml</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>joda-time</groupId>
                    <artifactId>joda-time</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.fasterxml.jackson.core</groupId>
                    <artifactId>jackson-databind</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.fasterxml.jackson.core</groupId>
                    <artifactId>jackson-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.fasterxml.jackson.datatype</groupId>
                    <artifactId>jackson-datatype-joda</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>19.0</version>
        </dependency>

{% endhighlight %}

### 创建ApplicationSwaggerConfig配置类

创建一个ApplicationSwaggerConfig配置类，源码如下：

{% highlight java %}

package apidoc;

import org.springframework.context.annotation.Bean;

import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 *
 * @author pan
 *
 */
@EnableSwagger2
public class ApplicationSwaggerConfig {

    @Bean
    public Docket addUserDocket() {
        Docket docket = new Docket(DocumentationType.SWAGGER_2);
        Contact contact = new Contact("陈胜楠", "", "chenshengnan@saicmotor.com");
        ApiInfo apiInfo = new ApiInfo("APP-MP", "WEB API文档", "V0.0.1", "", contact, "", "");
        docket.apiInfo(apiInfo);
        return docket;
    }
}

{% endhighlight %}


### 改写web.xml配置文件
web.xml配置文件添加如下配置：

{% highlight xml %}

  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.js</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.css</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.jpg</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.png</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.gif</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.html</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.wof</url-pattern>
  </servlet-mapping>

{% endhighlight %}

### 改写spring-mvc配置文件
spring-mvc配置文件改写为如下：

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<!--
  ~ **************************************************************************************
  ~
  ~   Project:        ZXQ
  ~
  ~   Copyright ©     2014-2017 Banma Technologies Co.,Ltd
  ~                   All rights reserved.
  ~
  ~   This software is supplied only under the terms of a license agreement,
  ~   nondisclosure agreement or other written agreement with Banma Technologies
  ~   Co.,Ltd. Use, redistribution or other disclosure of any parts of this
  ~   software is prohibited except in accordance with the terms of such written
  ~   agreement with Banma Technologies Co.,Ltd. This software is confidential
  ~   and proprietary information of Banma Technologies Co.,Ltd.
  ~
  ~ **************************************************************************************
  ~
  ~   Class Name: D:/workspace/app-mp/src/main/webapp/WEB-INF/zxq-servlet.xml
  ~
  ~   General Description:
  ~
  ~   Revision History:
  ~                            Modification
  ~    Author                Date(MM/DD/YYYY)   JiraID           Description of Changes
  ~ ***************************************************************************************
  ~    key                   2017-01-23
  ~
  ~ ***************************************************************************************
  -->

<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:util="http://www.springframework.org/schema/util"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd
        http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-3.2.xsd
        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd">
	<context:component-scan base-package="com.zxq.iov.cloud.app.mp" />

	<context:component-scan base-package="apidoc" />

<!-- 参数解析器 -->
	<mvc:annotation-driven>
		<mvc:argument-resolvers>
			<bean class="com.saicmotor.telematics.framework.core.web.annotation.EncodedHandlerMethodArgumentResolver" />
		</mvc:argument-resolvers>
	</mvc:annotation-driven>

	<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping" />

	<bean name="applicationSwaggerConfig" class="apidoc.ApplicationSwaggerConfig" />
	<mvc:default-servlet-handler/>
	
	
<!-- 	<bean id="RBACInterceptor" class="com.zxq.iov.cloud.app.mp.interceptor.RBACInterceptor"/>   -->

<!-- 	<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping">   -->
<!-- 		<property name="interceptors">   -->
<!-- 			<list>   -->
<!-- 				<ref bean="RBACInterceptor"/>   -->
<!-- 			</list>   -->
<!-- 		</property>   -->
<!-- 	</bean> -->

</beans>

{% endhighlight %}

### 查看接口文件

按照以上方法改写项目后，编译并部署在tomcat中。
访问接口描述文档的json文件地址如下：
>http://HOST:PORT/PROJECT/v2/api-docs

访问swagger页面地址如下：
>http://HOST:PORT/PROJECT/v2/swagger-ui.html
