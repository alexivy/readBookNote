前后分离部署的项目中遇到的问题及解决方法。

- [Frontend--Vue](#frontend--vue)
  - [Vue脚手架快速搭建项目](#vue脚手架快速搭建项目)
  - [项目运行中出现的问题](#项目运行中出现的问题)
    - [局域网中其他设备无法访问](#局域网中其他设备无法访问)
  - [项目的设置](#项目的设置)
  - [Vue](#vue)
    - [Vue父子组件](#vue父子组件)
    - [Vue组件高度设置](#vue组件高度设置)
  - [Vuex](#vuex)
  - [Vue实现token登录](#vue实现token登录)
- [Backend--SSM](#backend--ssm)
  - [跨域问题](#跨域问题)

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
