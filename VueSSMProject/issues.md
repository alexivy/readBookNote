前后分离部署的项目中遇到的问题及解决方法。

- [Frontend--Vue](#frontend--vue)
  - [Vue脚手架快速搭建项目](#vue脚手架快速搭建项目)
  - [项目运行中出现的问题](#项目运行中出现的问题)
    - [局域网中其他设备无法访问](#局域网中其他设备无法访问)
    - [部分页面工作不正常](#部分页面工作不正常)
  - [项目的设置](#项目的设置)
  - [Vue](#vue)
    - [Vue父子组件](#vue父子组件)
    - [Vue组件高度设置](#vue组件高度设置)
    - [boolean值的显示处理](#boolean值的显示处理)
  - [Vuex](#vuex)
  - [Vue实现token登录](#vue实现token登录)
  - [axios发送表单数据](#axios发送表单数据)
  - [axios.interceptors](#axiosinterceptors)
    - [全局接受response后显示相应提示](#全局接受response后显示相应提示)
- [Backend--SSM](#backend--ssm)
  - [跨域问题](#跨域问题)
  - [ControllerAdvice](#controlleradvice)
    - [ExceptionHandler处理异常](#exceptionhandler处理异常)
  - [shiro](#shiro)
    - [配置shiro后访问Controller出现404](#配置shiro后访问controller出现404)
    - [shiro重复登陆404](#shiro重复登陆404)
    - [shiro的role和permission](#shiro的role和permission)
    - [shiro处理后的返回信息](#shiro处理后的返回信息)
    - [shiro session管理](#shiro-session管理)
  - [MyBatis](#mybatis)
    - [Mapper配置问题](#mapper配置问题)
    - [多字段更新](#多字段更新)
  - [spring容器启动时的一个报错](#spring容器启动时的一个报错)
  - [hibernate validator做后端校验](#hibernate-validator做后端校验)
  - [activiti](#activiti)
    - [启动错误](#启动错误)

# Frontend--Vue  

## Vue脚手架快速搭建项目  

    npm install -g cnpm --registry=https://registry.npm.taobao.org      //安装淘宝镜像 执行后可用cnpm代替npm
    cnpm install vue     //安装Vue
    cnpm install --global vue-cli       //安装vue脚手架，安装后可用下一条命令快速搭建项目
    vue init webpack projectname        //搭建vue demo项目，名字无大写字母
    安装后进入项目目录
    cnpm install        //安装依赖
    npm run dev     //运行项目
    //此命令实际上运行了package.json文件中的scripts.dev中的命令

## 项目运行中出现的问题  

### 局域网中其他设备无法访问  
在package.json文件中的scripts.dev的末尾加上"--host 192.168.3.28",然后便可在其他设备上通过内网ip访问，此时本机上也要通过ip进行访问（localhost失效）。  

### 部分页面工作不正常  
应该是windows防火墙拦截了axios请求。所有axios请求不能工作。  

## 项目的设置  
config/index.js文件中可配置运行的端口  

## Vue  

### Vue父子组件  

子组件的script中  

    export default {
        name: "MenuFrame",
    }

父组件的script中加入  

    import MenuFrame from './frame'  //import子组件

    export default {
        components:{
            MenuFrame,
        }
    }

父组件的templacte中在需要引入子组件的位置加入  

    <menu-frame></menu-frame>

### Vue组件高度设置  

template中配置:style属性。  

    <el-container :style="{height:elHeight}">

script脚本中，配置在data中修改，或者在生命周期中修改。  

    export default {
        name: "MenuFrame",
        data() {
            return {
            elHeight: document.body.clientHeight + "px",
            };
        }
    }

document.body.clientHeight可能无法获得正确值。此时在html的\<head\>标签中如下配置：  

    <!-- 为能够通过document.body.clientHeight正确获得高度 -->
    <style type="text/css">
        html,body{
            height: 100%;
        }
    </style>

### boolean值的显示处理  

在element-ui的表格中显示  
```
<el-table-column prop="sex" label="性别" width="180" :formatter="formatBoolean"/>
```

```
script中methods中定义函数
formatBoolean: function (row, column, cellValue) {
        var ret = ''  //你想在页面展示的值
        if (cellValue) {
            ret = "男"  //根据自己的需求设定
        } else {
            ret = "女"
        }
        return ret;
      },
```


## Vuex  

在template中可通过下面的方式访问store中的属性  

    {{$store.state.object}}

## Vue实现token登录  

整体思路：  
1. 首次登录时，后端服务器判断用户账号密码正确之后，根据用户id、用户名、定义好的秘钥、过期时间生成 token ，返回给前端；  
2. 前端拿到后端返回的 token ,存储在 localStroage 和 Vuex 里；  
3. 前端每次路由跳转，判断 localStroage 有无 token ，没有则跳转到登录页，有则请求获取用户信息，改变登录状态；  
4. 每次请求接口，在 Axios 请求头里携带 token;  
5. 后端接口判断请求头有无 token，没有或者 token 过期，返回401；  
6. 前端得到 401 状态码，重定向到登录页面。  

action中存储token，设置请求携带token信息：  
    
    .then(res => {
        if (res.status === 200) {
            commit(types.LOGIN, res.data.token)

            axios.defaults.headers.common['Authorization'] = `Bearer ` + res.data.token
            window.localStorage.setItem('token', res.data.token)
            resolve(res)
        }
    })

router中配置requiresAuth的路径需要有token，没有就跳转登录页

    router.beforeEach((to, from, next) => {
    let token = window.localStorage.getItem('token')
    if (to.meta.requiresAuth) {
        if (token) {
        next()
        } else {
        next({
            path: '/login',
        })
        }
    } else {
        next()
    }
    })

设置axios的拦截器，respone拦截器,无token或者token无效时服务器返回status为401，此时将页面重定向至login页面：  

    axios.interceptors.response.use(
    response => {
        return response
    },
    error => {
        if (error.response) {
            switch (error.response.status) {
                case 401:
                    router.replace({
                        path: 'login',
                    })
            }
        }
        return Promise.reject(error.response)
    }
    )

## axios发送表单数据  

```
axios({
    method: 'post',
    url: '/login',
    data: data,
    transformRequest: [function (data) {
      // Do whatever you want to transform the data
      let ret = ''
      for (let it in data) {
        ret += encodeURIComponent(it) + '=' + encodeURIComponent(data[it]) + '&'
      }
      return ret
    }],
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    }
  })
```

## axios.interceptors  

### 全局接受response后显示相应提示  

用到了elementUI中的notification。。[参照此处](https://element.eleme.cn/#/zh-CN/component/notification#fang-fa)  

```
import { Notification } from 'element-ui';

axios.interceptors.response.use(
  response => {
    return response
  },
  error => {
    if (error.response) {
      switch (error.response.status) {
        case 401:
          //原来是replace 改为push，replace时会出现/home/userlist==》/home/userlist/login
          router.push({
            path: '/login',
            //query: { redirect: router.currentRoute.fullPath }  将跳转的路由path作为参数，登录成功后跳转到该路由
          })
          break;
        case 400:
          // console.log("400"+JSON.stringify(error))
          Notification({
            title: "错误",  
            message: error.response.data.message,  //现实的信息
            type: "error",  //类型：success、info、error、warning四种
            duration: 3000  //持续时间，单位毫秒。0为不自动关闭
          });
      }
    }
    return Promise.reject(error.response)
  }
)
```

# Backend--SSM  

## 跨域问题  
三种思路。  
1. 使用@CrossOrigin注解在相应controller上。Interceptor拦截后仍有跨域的问题。  
2. 使用spring mvc全局配置：  

```
<mvc:cors>
    <mvc:mapping path="/**" allowed-origins="*"
        allowed-methods="POST, GET, OPTIONS, DELETE, PUT"
        allowed-headers="Content-Type, Access-Control-Allow-Headers, Authorization, X-Requested-With"
        allow-credentials="true" />
</mvc:cors>
```

3. 使用cors-filter解决，  

```
pom中
<dependency>
    <groupId>com.thetransactioncompany</groupId>
    <artifactId>cors-filter</artifactId>
    <version>2.5</version>
</dependency>
```
```
web.xml中
<!-- 配置跨域过滤器 -->
  <filter>
    <filter-name>CORS</filter-name>
    <filter-class>com.thetransactioncompany.cors.CORSFilter</filter-class>
    <init-param>
      <param-name>cors.allowOrigin</param-name>
      <param-value>*</param-value>
    </init-param>
    <init-param>
      <param-name>cors.supportedMethods</param-name>
      <!-- <param-value>*</param-value> --> <!-- 表示所有请求都有效 -->
      <param-value>GET, POST, HEAD, PUT, DELETE</param-value>
    </init-param>
    <init-param>
      <param-name>cors.supportedHeaders</param-name>
      <param-value>Accept, Origin, X-Requested-With, Content-Type, Last-Modified, Access-Control-Allow-Headers, Authorization</param-value>
    </init-param>
    <init-param>
      <param-name>cors.exposedHeaders</param-name>
      <param-value>Set-Cookie</param-value>
    </init-param>
    <init-param>
      <param-name>cors.supportsCredentials</param-name>
      <param-value>true</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>CORS</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```

## ControllerAdvice  

### ExceptionHandler处理异常  

为避免sql异常信息直接返回到前端，对异常进行处理。  

```
首先是class  
@ControllerAdvice
public class ExceptionController {

    private Logger logger = Logger.getLogger ( ApiController.class );

    /**
     * 处理可能遇到的sql异常，避免其向前端返回错误信息，统一返回应答status 400 ，Bad  Request。
     */
    @ExceptionHandler(SQLSyntaxErrorException.class)
    @ResponseBody
    public Result sqlExceptionController(Exception exception, HttpServletRequest request, HttpServletResponse response) throws Exception{
        //记录异常
        logger.error(exception.getMessage(), exception);
        response.setStatus(ResultStatus.BADREQUEST.value());
        Result<Object> res = new Result<>();
        res.setCode(ResultStatus.BADREQUEST.value());
        res.setMessage("parameter wrong");
        return res;
    }
}
```

spring mvc中配置扫描component，ControllerAdvice继承了Component。

```
<context:component-scan base-package="cn.alexivy.sim.controller" use-default-filters="false">
        <context:include-filter type="annotation" expression="org.springframework.stereotype.Component"/>
</context:component-scan>
```

## shiro  

### 配置shiro后访问Controller出现404  

原本是参照[此链接](https://www.iteye.com/blog/jinnianshilongnian-2029717)进行配置的。  
后来发现把之前配置的拦截器注释掉就可以正常访问了。  

### shiro重复登陆404  

shiro在filter中会先判定isAccessAllowed，正常登录时此方法会返回false，但重复登陆时会返回true，此时会向spring mvc的dispatcherservlet请求原登录的uri，找不到然后返回404。

### shiro的role和permission  

controller注解中配置的所有role和permission都必须满足，请求的subject才能访问。同时shiro permission支持通配符。
如下：  
```
某个subject有"stu:*"的permission，那么他就可以访问下面注解所标注的方法。  
@RequiresPermissions(value = "stu:update")所注解的方法  
```

### shiro处理后的返回信息  

在xml文件中的配置能将其返回到某些jsp页面中，但可以通过如下的方式实现返回json。  
```
此处以登录成功返会json数据为例。  

1、定位shiro重定向至相应页面的filter的相应方法。此处定位到了FormAuthenticationFilter的下面这个方法：  
protected boolean onLoginSuccess(AuthenticationToken token, Subject subject, ServletRequest request, ServletResponse response) throws Exception {
        this.issueSuccessRedirect(request, response);
        return false;
    }

2、实现自己的filter继承之前的filter并重写相应方法，方法里直接添加返回信息即可。  
public class MyFormAuthenticationFilter extends FormAuthenticationFilter {
    //...
    @Override
    protected boolean onLoginSuccess (AuthenticationToken token, Subject subject, ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        httpServletResponse.setContentType("text/json;charset=UTF-8");//设置响应头
        Result<User> res = new Result<>();
        res.setMessage("登陆成功！");
        res.setData(userService.getUserInfoByUsername((String) token.getPrincipal()));
        httpServletResponse.getWriter().write(res.toJson());
        return false;
    }
}

3、将自己实现的filter配置到shiroFilter中。如下：
    <!-- 基于Form表单的filter  -->
    <bean id="formAuthenticationFilter" class="cn.alexivy.sim.shiro.filter.MyFormAuthenticationFilter">
        <!--   略    -->
    </bean>
    <bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
        <property name="filters">
            <util:map>
                <!-- 配置为自己的filter，实现特殊的返回结果  -->
                <entry key="authc" value-ref="formAuthenticationFilter"/>
            </util:map>
        </property>
        <!--   略    -->
    </bean>
```
ps：shiro中重定向常用下面这个方法，给他打个断点然后看调用链就可快速找出在哪个方法判断进行重定向/操作response。
```WebUtils.redirectToSavedRequest(request, response, this.getSuccessUrl());```


### shiro session管理  
碰到了java.lang.InstantiationError: org.quartz.SimpleTrigger异常。  
解决方法参考[shiro quartz 版本不兼容](https://blog.csdn.net/m0_37499059/article/details/81532515?utm_source=blogxgwz8)。  
由于没有其他地方用到quartz，应该是用了新版本的shiro，spring配置中却参考了老版本的导致尝试实例化接口出现的问题。  


## MyBatis  

### Mapper配置问题

报错Invalid bound statement，检查了一下几项：  

1. mybatis的mapper配置，一般在spring配置文件中class="org.mybatis.spring.SqlSessionFactoryBean"的bean的mapperLocations属性，或是在mybatis的配置文件中的mappers标签中。  

2. 检查编译好的项目中是否有对应mapper.xml文件。IDEA有时编译时不会将source文件夹下的其他文件编译或者说打包。将mapper文件放入resources文件夹下，或者将项目的pom.xml文件中的build节点下加入如下代码：  

```
<resources>
    <resource>
        <directory>src/main/java</directory>
        <includes>
            <include>**/*.xml</include>
        </includes>
    </resource>
</resources>
```

3. mapper文件有没有错误，包括namespace配置是否正确（对应其类文件）、是否有interface中的方法对应的条目、返回类型resultMap是否正确。  

### 多字段更新  
原本是用的动态表名，现在改成如下的if判断。
使用#{}可以避免sql注入  
${}的话是sql拼接，会有sql注入风险，statementType只能是STATEMENT，且需要自己判断是否加引号。  
```
方法：
public boolean changeStuInfo(Student student);
对应xml部分：注意trim段的suffixOverrides配置可以去掉多余的逗号。（多的逗号会引起sql错误）
<update id="changeStuInfo" parameterType="cn.alexivy.sim.bean.Student">
      UPDATE stuinfo
      <trim prefix="set" suffixOverrides=",">
          <if test="name!=null">realname=#{name},</if>
          <if test="sex!=null">sex=#{sex},</if>
          <if test="homeAddress!=null">homeAddress=#{homeAddress},</if>
          <if test="localAddress!=null">localAddress=#{localAddress},</if>
          <if test="homePhone!=null">homePhone=#{homePhone},</if>
          <if test="selfPhone!=null">selfPhone=#{selfPhone},</if>
          <if test="depart!=null">depart=#{depart},</if>
      </trim>
      WHERE uid = #{id};
</update>
```

## spring容器启动时的一个报错  
报错```class path resource [spring/] cannot be resolved to URL because it does not exist```   
解决办法1：  

```
pom.xml文件的build节点下加入
<resources>
<resource>
  <directory>src/main/resources</directory>
  <includes>
    <include>**/*.properties</include>
    <include>**/*.xml</include>
  </includes>
  <filtering>false</filtering>
</resource>
</resources>
```

办法2：  
```
改写成 classpath*:（后面略）  
classpath 和 classpath* 区别：  
classpath：只会到你的class路径（IDEA中编译后在 target/classes 文件夹中）中查找文件;  
classpath*：不仅包含class路径，还包括jar文件中(class路径)进行查找。当项目中有多个classpath路径，并同时加载多个classpath路径下（此种情况多数不会遇到）的文件，* 就发挥了作用，如果不加 * ，则表示仅仅加载第一个classpath路径。  
```

## hibernate validator做后端校验  
引入依赖  
```
 <dependency>
      <groupId>org.hibernate.validator</groupId>
      <artifactId>hibernate-validator</artifactId>
      <version>6.1.5.Final</version>
    </dependency>
```
注解扫描  
```
<mvc:annotation-driven />
```
需要校验的实体
```
public class Student {
    @NotNull(message = "id can't be null")
    @Min(value=-1)
    private int id;
    ...
}
```
对应的方法，注意方法的参数err用于获取校验的信息，没有这个参数的话会404错误。如果有校验出错的信息的话会抛异常。  
```
@RequiresPermissions(value = "stu:update")
@RequestMapping(value="/changeWithStu",method = RequestMethod.POST)
public Result<Object> changeStuInfo(@Valid @RequestBody Student stu, BindingResult err) throws ApiException {
    Result<Object> res = new Result<>();
    List<ObjectError> allErrors = err.getAllErrors();
    if(allErrors!=null && allErrors.size()!=0){
        StringBuilder mes=new StringBuilder();
        for(ObjectError e:allErrors) {
            mes.append(e.getDefaultMessage() + "\n");
        }
        ApiException throwables = new ApiException(mes.toString());
        throw throwables;
    }
    res.setData(studentService.changeStuInfoByStu(stu));
    return res;
}
```
对应异常类  
```
public class ApiException extends Exception{
    public ApiException(String mes) {
        super(mes);
    }
}
```
捕获异常并向前端返回校验信息
```
@ControllerAdvice
public class ExceptionController {
    private Logger logger = Logger.getLogger ( ApiController.class );
    /**
     * 处理可能遇到的api异常，设置向前端返回错误信息，统一返回应答status 400 ，Bad  Request。
     * @param exception
     * @return
     */
    @ExceptionHandler({ApiException.class})
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ResponseBody
    public Result<Object> apiExceptionController(Exception exception) {
        //记录异常
        logger.info(exception.getMessage());
        Result<Object> res = new Result<>();
        res.setMessage(exception.getMessage());
        return res;
    }
```

## activiti  

### 启动错误  
java.sql.SQLSyntaxErrorException: Table 'sim.act_ge_property' doesn't exist  
参考[此处解决](https://blog.csdn.net/weixin_43770545/article/details/101197861)  
dataSOurce的url中添加配置nullCatalogMeansCurrent=true  
