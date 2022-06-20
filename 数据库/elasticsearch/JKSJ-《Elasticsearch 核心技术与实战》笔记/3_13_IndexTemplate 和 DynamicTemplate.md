# Index Template

## 什么是Index Template

- Index Templates , 顾名思义 , 索引模板 , 就是将创建index时配置的一些参数, 抽象成一个模板 . 方便后面创建模板时复用相同的规则.  

   

- 注意 : 
   - 模版仅在一个索引被新创建时，才会产生作用。修改模版**不会影响已创建的索引**
   - 可以设定多个索引模版，这些模板所指定的设置.  会在创建index时合并到一起.
   - 可以指定 `order` 的数值.  控制使用多个模板时,  哪些模板的相同的规则优先级更高. 



## Index Template 的工作方式

当一个索引被创建时 

- 默认使用 elasticsearch 默认的 settings 和 mappings
- 应用 `order` 数值低的 index template 中的设定
- 应用 `order` 数值高的 index template 中的设定 , 相同的设定 , 之前的order更低的会被order更高的覆盖.
- 应用创建索引时 , 用户显式指定的 `settings` 和 `mappings` , 并覆盖之前模板中的设定. 





- 示例 : 

  ![image-20220620000116065](http://assets.studymachine.cn/img/202206200025784.png)

  - 上述示例为创建两个不同的 index template


  - 其中左边语句 `"index_patterns" : ["*"]`  表示该模板 适用于所有新建的index.  `"index_patterns" : ["test*"]`表示该模板适用于新建的test开头的index.


  -  关于`order`字段, 左边 = 0 ,  右边 = 1.  所以 右边的 `settings` 会覆盖左边的template 中的 设定.

​    




# Dynamic Template

## 什么是 Dynamic Template

- 可以理解为 dynamic mapping 的自定义配置模板.   根据 elasticsearch 识别的数据类型.   结合字段的名称  , 来动态设置字段的类型. 
- 例如 : 
  - 所有字符串的类型都是设置为keyword , 或者关闭keyword 字段
  - `is` 开头的字段都设置为boolean 类型
  - `long_`开头的都设置为long类型.

- 示例 : 

  ![image-20220620004411281](http://assets.studymachine.cn/img/202206200046052.png)



## 与 index template 的区别

-  index template 是 针对 index级别的设定模板.  dynamic template 是针对 index 内部 字段 field 的设定模板. 

- dynamic template 是定义在某个索引的mapping 中的 . 

- 匹配规则是一个数组类型.  

- 为匹配到的字段设置mapping

- 例如 : 

  ![image-20220620003745982](http://assets.studymachine.cn/img/202206200046508.png)

    - `"path_match":"name.*`    :  匹配字段名为 name 的所有子级字段
  
    - `"path_unmatch":"*.middle`  :  不匹配任意一级的子级字段名为 middle 的字段
  
    - mapping :  指定匹配的字段mapping 为 text 类型 , 复制并生成新的索引字段 `full_name`
  
    - 验证 : 
  
      ![image-20220620005130059](http://assets.studymachine.cn/img/202206200051164.png)

