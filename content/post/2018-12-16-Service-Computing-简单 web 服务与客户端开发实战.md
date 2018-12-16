---
title: "Service Computing 简单 web 服务与客户端开发实战"
date: 2018-12-16T11:38:52+08:00
lastmod: 2018-12-16T11:41:52+08:00
menu: "main"
weight: 50
author: "hansenbeast"
tags: [
    "Service Computing"
]
categories: [
    "Tech Blogs"
]
# you can close something for this content if you open it in config.toml.
comment: false
mathjax: false
---



# 项目总结

16340232 王泽浩

- [本人承担的任务](## 本人承担的任务)

- [具体流程](## 具体流程)

- [心得体会](## 心得体会)


## 本人承担的任务

在后端根据建立完成的boltdb数据库，添加路由处理不同请求的url。



## 具体流程

1. 首先是API的设计和路由的添加，由于复制的SWAPI的RESTful风格，根据官网的文档可以将url分为一下几种：

- root 

  `hostname/api`类似于https://swapi.co/api/所有API的根路径需要以这种前缀开始

- schema

  `hostname/api/<resource>/schema`不同类型的资源的所有详细字段内容需要以JSON格式返回

- search

  `hostname/api/<resource>/?search=r2`利用query查询所有满足条件的资源

- id

  `hostname/api/<resource>/1/`根据id查询

- page

  `hostname/api/<resource>/?page=9`利用query返回满足页码中的资源，每页最多输出10个结果



2. 根据利用golang中的mux库注册多个相应的Route并实现其handler

   注意其中id可以通过添加子路由

   ```go
   mx.HandleFunc("/api/people/{id:[0-9]+}/",peopleIdHandler(formatter,db)).Methods("GET")
   ```

   并在handler中得到相应的参数

   ```go
   vars := mux.Vars(req)
   // 获取id
   id := vars["id"]
   ```



​	而search和page等请求可以通过利用url query进行处理

```go
func peopleHandler(formatter *render.Render,db *bolt.DB) http.HandlerFunc{
	vars := req.URL.Query();
	search, search_ok:= vars["search"]
	page_param , page_ok := vars["page"]
    ...
}
```



3.  boltdb的读写与JSON格式的返回

   由于组员已经实现好了接口：

   ```go
   func GetElementById(db *bolt.DB, blockName string, id string) ([]byte, error)
   func GetElementsBySearchField(db *bolt.DB, blockName string, value string) ([][]byte, error) 
   func GetAllResources(db *bolt.DB, blockName string) 
   func GetSchemaByBucket(db *bolt.DB, blockName string)
   ```

   根据返回的一维和二维byte，可以利用json库中的Unmarshal，解析为resource的data interface，即不同资源的结构体，并根据结构体转换为JSON格式输出，相当于将数据库存储的格式进行类型转换。

   其中render库中的

   ```go
   formatter.JSON(w,http.StatusOK,interface{})
   ```

   可以直接将interface输出为JSON格式，在根据响应状态添加头部信息

   ```json
   HTTP/1.0 200 OK
   Content-Type: application/json
   {
     "name": "Luke Skywalker",
     "height": "172",
     "mass": "77",
     "hair_color": "blond",
     "skin_color": "fair",
     "eye_color": "blue",
     "birth_year": "19BBY",
     "gender": "male",
     "homeworld": "https://swapi.co/api/planets/1/",
     	...
   }
   ```



4. 处理分页输出

   当得到请求所有resource的时候，可以首先从数据库中获得所有资源，然后判断page是否符合逻辑，如果缺省page参数，则默认为1，然后根据当前页添加对应下标的资源。

   ```go
   type Peoples struct {
   	Count int `json:"count"`
   	Next string `json:"next"`
   	Previous string `json:"previous"`
   	Results []interface{} `json:"results"`
   }
   ```

   Count记录当前页显示的数量，Next为下一页的url，如果为最后一页，则默认为空，Previous同理，Results保存当前页要显示的资源。





## 心得体会

本人理解的RESTful架构：

- 每一个URI代表一种资源，不应该包括动词信息
- 客户端和服务器之间，传递这种资源的某种表现层，如此次API中的"资源"JSON实体。
- 客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。GET用来获取资源，POST用来新建资源（也可以用于更新资源），PUT用来更新资源，DELETE用来删除资源。



熟悉了使用Github组织进行前后端分离的开发过程和协作方式，如建立不同的分支更好地处理冲突，最后再利用pull request进行版本迭代。

