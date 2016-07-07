# 服务接口v2（Service Interface Version 2)  

---  
  
## 1. 说明  
  
### 1.1. 接口协议  
本文档中的接口，指的是客户端程序可以通过网络发起调用的单元。接口调用的请求和结果数据通过**HTTP**协议传输，不同的HTTP请求方法对应不同的操作。每一个接口对应于一个相对于的**URI**，以及一系列可以针对这个URI进行的操作。  

除非有特殊说明，通常情况下，HTTP请求方法对应的操作含义如下：  
  
* **POST** 创建对象  
* **PUT** 更新对象  
* **GET** 查询对象  
* **DELETE** 删除对象  
  
请求参数和返回参数格式：  
  
* 对于GET请求，使用URL中的查询字符串来传递参数；  
* 对于其他请求，使用请求的消息体中的json字符串来传递参数；  
* 对于所有请求的返回结果，使用返回的消息体中的json字符串来传递参数。  

#### 注：
* 对于删除操作，由于保存数据的需要，实际并**不会删除数据**，而是加上**删除标记**；客户端仍然能请求到标记为已删除的数据，需要根据标记自行处理，但是一般不能对标记为已删除的数据进行修改。  
* 对于更新操作，若无特殊说明，客户端仅需要将有更新的字段传递给服务端，而不必传递整个对象。

  
### 1.2. 用户身份验证  
本文档中除了登陆以外的所有接口，在调用时都需要携带用户身份标识。目前阶段，用户身份标识由**JWT**(Json Web Token)来实现。客户端首先进行登陆（参考[“2.6. 登陆”](#login)），登陆成功时，服务端签署JWT并将其返回给客户端。在后续的每一个请求中，客户端都必须携带此JWT，以便服务端对用户身份进行识别。客户端携带JWT的方法是在请求的HTTP header中包含如下的头部来实现的：  

    Authorization: Bearer <token>

**如果条件允许，应该将传输协议升级为HTTPS，以防止token泄露以及代理人攻击。**  

### 1.3. 数据类型  
通过本接口协议传输的所有数据，其类型主要参考javascript数据类型。对于一些较为特殊的数据类型，说明如下：  

* date 数据类型表示日期时间，用从UTC1970年1月1日午夜（零时）开始经过的毫秒数来表示  
* ARRAY 为数组类型  
* bool 数据用0、1代替
* TREE 为树数据结构，在JSON中通过（嵌套的）对象来表示  
  
<span id="reactive"></span>  
### 1.4. 响应式API  
所谓响应式API，是指一套API能够满足不同的应用场景,根据用户请求所带的参数，返回不同的内容。  
实现响应式API的目的是为了在一个请求中返回给用户需要的有用数据。  
考虑调用API接口查询一个集合users，并在客户端界面上显示一个包含用户名和id的列表时的情况。通常情况下客户端通过GET方法请求集合users，API返回所有user的id列表，随后客户端对每一个user通过GET方法请求user的详细信息，并获取其中的用户名。这是一般的调用流程，但是在这种流程下客户端需要多次对API的调用才能得到需要的所有信息，而且对每一个用户的GET请求返回的信息中，其实只有用户名是需要的，其他的字段浪费了网络流量。因此采用响应式API的改进接口调用流程是这样的：客户端通过GET方法请求集合users，并携带“fields=username”参数指示API返回的每条记录中需要的字段信息，API直接返回包含每个user的id和username的列表。通过响应式API，客户端只需一次调用就能得到所有数据。目前响应式API仅针对查询（GET）请求。  
本协议中使用的与响应式API相关的查询参数包含：  
* Paging  
    由offset和limit查询参数来实现分页控制，offset指定开始位置，默认值为0，limit指定一次返回的记录数量，默认值为0表示返回从offset开始的所有记录。当使用Paging时，返回消息的HTTP头部包含X-Total-Count头，指出总记录条数。  
    `GET /users?offset=100&limit=100`  
* Fields  
    仅返回特定字段的数据集，对于集合的查询默认情况下仅返回id列表，对于具体记录的查询默认返回所有可用字段。可用的字段名称通常为记录的字段名称的子集，具体参见具体接口中的描述。  
    `GET /users?fields=username,year,class`  
* Filter  
    仅返回满足指定条件的数据集。条件由对某个字段的值的判断组成。可选字段通常为记录的字段的子集。  
    表示范围时使用数学中的区间符号来表示；对字符串字段的匹配可以使用通配符（后面可以考虑加入正则表达式支持）。  
    可以用逗号分隔的列表来表示一个集合，比如id=1,3,5,16,20，表示匹配这个集合中的任意一个。  
    例如：  
    `GET /user?year=[16,18)&class=5&id=36,45,87,167&name=student*3`  
  
<span id='longpolling'></span>
### 1.5. 长轮询  
由于用户的通知消息现在是通过客户端主动查询来实现的，这种机制也被称为轮询。轮询机制通常是低效的，因为在没有通知消息的时候也必须不停的发出请求，采用轮询机制的原因是其实现的简单性。  
为了降低服务器负担，轮询的查询时间间隔不宜太小；但是为了有更快的响应速度，又要求查询时间足够小。为了解决这个矛盾，对于用户通知消息的获取引入长轮询机制。这个机制保留了轮询机制的简单性，同时又有很好的实时性，并且几乎不增加服务器负担。  
长轮询的一般模式是：  
* 在请求一个可能不是立即可用的资源时，客户端在请求中携带一个超时值参数，这里假设叫做timeout  
* 服务器检查请求的资源，如果资源可用，则立即返回该资源，处理结束  
* 如果资源不可用，则挂起客户请求（不立即回应）直到资源变得可用或者超时值到期  
* 如果资源变得可用，则返回该资源，处理结束  
* 如果直到超时到期时资源仍不可用，则返回一个超时失败的错误给客户端  
* 客户端接收到超时错误的返回后，立即再发起一个这样的请求，整个过程一直循环，直到服务器返回该资源  
这种机制避免了频繁的向服务器请求资源，同时在资源变得可用时又能立即获取到资源。  
具体来说，对于通知消息的获取，客户端请求的是通知消息的**变化**，因此客户端在请求中除了携带超时参数外，还应该携带客户端当前通知消息的状态。服务器接收到客户端的请求后，将客户端通知消息状态与服务器端用户的通知消息状态进行对比，如果不同则返回服务器端的状态给客户端，如果相同则执行上面的长轮询模式。具体的接口描述见后面接口部分。
  
## 2. 用户管理  
  
### 2.1. 班级  
  
#### 2.1.1. 创建班级  
* URI：/banjis  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                      |  
    |:------ |:-------|:----:|:------------------------- |  
    | name   | string | 是   | 班级名称，比如“二（2）班” |  
    | ruxue  | date   | 是   | 班级入学时间              |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 返回成功创建班级的ID   |  
  
* 说明  
    * 返回的ID用来在对已经创建的班级进行其他操作时唯一的标识一个班级  
  
#### 2.1.2. 查询班级列表  
* URI：/banjis  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、ruxue、renshu、shanchu、xiugaitime，默认只包括id  
    * Filter，支持的字段同上
  
* 返回参数  
  
    | 名称          | 类型   | 说明                      |  
    |:------------- |:-------|:------------------------- |  
    | banjis[count] | ARRAY  | count个班级信息组成的数组 |  
    
    其中“banjis[count]”数组的每个元素的结构为(Fields包含该字段时)：  
  
    | 名称      | 类型   | 说明                   |  
    |:--------- |:-------|:---------------------- |  
    | id        | int    | 班级的ID               |  
    | name      | string | 班级名称               |  
    | ruxue     | date   | 班级入学时间           |  
    | renshu    | int    | 班级人数               |  
    | shanchu   | int    | 是否已删除             |  
    | xiugaitime| date   | 最后修改时间           |  
  
* 说明  
    关于Paging、Fields、Filter参考[“1.4 响应式API”](#reactive)  
  
#### 2.1.3. 查询班级  
* URI：/banjis/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、ruxue、renshu、shanchu、xiugaitime，默认包括所有字段  
  
* 返回参数  
  
    | 名称      | 类型   | 说明                   |  
    |:--------- |:-------|:---------------------- |  
    | id        | int    | 班级的ID               |  
    | name      | string | 班级名称               |  
    | ruxue     | date   | 班级入学时间           |  
    | renshu    | int    | 班级人数               |  
    | shanchu   | int    | 是否已删除             |  
    | xiugaitime| date   | 最后修改时间           |  
  
* 说明  
    URI中的“id”为创建班级时返回的所创建班级的ID  
  
#### 2.1.4. 修改班级  
* URI：/banjis/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选  | 说明                     |  
    |:------ |:-------|:-----:|:------------------------ |  
    | name   | string | 否    | 班级名称，比如“二（2）班”|  
    | ruxue  | date   | 否    | 班级入学时间             |  
  
* 返回参数  
    无  
* 说明  
    无  
  
#### 2.1.5. 删除班级  
* URI：/banjis/:id  
* 请求方法：DELETE  
* 请求参数  
    无  
* 返回参数  
    无  
* 说明  
    无  
  
  
### 2.2. 班级课程  
  
#### 2.2.1. 创建班级课程  
* URI：/banjis/:bjid/kechengs  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                         |  
    |:------ |:-------|:----:|:---------------------------- |  
    | name   | string | 是   | 课程名称，比如“数学七年级下” |  
    | laoshi | int    | 是   | 任课老师的ID                 |  
    | jihua  | int    | 是   | 课程所属教学计划ID           |  
    | jindu  | int    | 否   | 当前教学进度ID               |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                         |  
    |:------ |:-------|:---------------------------- |  
    | id     | int    | 返回成功创建的班级课程的ID |  
  
* 说明  
    * 返回的ID用来在对已经创建的班级课程进行其他操作时唯一的标识一个班级课程。  
    * 如果没有指定教学进度，则自动设置为当前教学计划的第一个教学任务。  
    * 创建课程时将自动创建一个默认[分组方案](#fenzufangan)，该方案只有一个分组，包含班级所有学生
  
#### 2.2.2. 查询班级课程列表  
* URI：/banjis/:bjid/kechengs  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、laoshi、jihua、jindu、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段同上  
  
* 返回参数  
  
    | 名称            | 类型   | 说明                          |  
    |:--------------- |:-------|:----------------------------- |  
    | kechengs[count] | ARRAY  | count个班级课程信息组成的数组 |  
    
    其中“kechengs[count]”数组的每个元素的结构为（Fields包含该字段时）：  
  
    | 名称       | 类型   | 说明                   |  
    |:---------- |:-------|:---------------------- |  
    | id         | int    | 班级课程的ID           |  
    | name       | string | 班级课程名称           |  
    | laoshi     | int    | 任课老师的ID           |  
    | jihua      | int    | 课程所属教学计划ID     |  
    | jindu      | int    | 当前教学进度ID         |  
    | shanchu    | int    | 是否已删除             |  
    | xiugaitime | date   | 最后修改时间           |     
  
* 说明  
    无  
  
#### 2.2.3. 查询班级课程  
* URI：/banjis/:bjid/kechengs/:kcid  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、laoshi、jihua、jindu、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称       | 类型   | 说明                   |  
    |:---------- |:-------|:---------------------- |  
    | id         | int    | 班级课程的ID           |  
    | name       | string | 班级课程名称           |  
    | laoshi     | int    | 任课老师的ID           |  
    | jihua      | int    | 课程所属教学计划ID     |  
    | jindu      | int    | 当前教学进度ID         |  
    | shanchu    | int    | 是否已删除             |  
    | xiugaitime | date   | 最后修改时间           | 
    
* 说明  
    无  
  
#### 2.2.4. 修改班级课程  
* URI：/banjis/:bjid/kechengs/:kcid  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                         |  
    |:------ |:-------|:----:|:---------------------------- |  
    | name   | string | 否   | 课程名称，比如“数学七年级下” |  
    | laoshi | int    | 否   | 任课老师的ID                 |  
    | jihua  | int    | 否   | 课程所属教学计划ID           |  
    | jindu  | int    | 否   | 当前教学进度ID               |  
  
* 返回参数  
    无  
* 说明  
    * 如果修改了教学计划ID，当时没有对教学进度ID进行同步修改，则当前教学进度ID会自动设置为指定教学计划的第一个教学任务。  
  
#### 2.2.5. 删除班级课程  
* URI：/banjis/:bjid/kechengs/:kcid  
* 请求方法：DELETE  
* 请求参数  
    无  
* 返回参数  
    无  
* 说明  
    无  
  
  
<span id="fenzufangan"></span>
### 2.3. 分组方案  
  
#### 2.3.1. 创建分组方案  
* URI：/banjis/:bjid/kechengs/:kcid/fenzufangans  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                         |  
    |:------ |:-------|:----:|:---------------------------- |  
    | name   | string | 是   | 分组方案名称                 |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                         |  
    |:------ |:-------|:---------------------------- |  
    | id     | int    | 返回成功创建的分组方案的ID   |  
  
* 说明  
    * 一个分组方案属于一个班级的一门课程
    * 一门课程可能有多个分组方案
    * 创建课程时将自动创建一个默认分组方案，该方案只有一个分组，包含班级所有学生
  
#### 2.3.2. 查询分组方案列表  
* URI：/banjis/:bjid/kechengs/:kcid/fenzufangans  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、fenzus、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段包括：id、name、shanchu、xiugaitime  
  
* 返回参数  
  
    | 名称            | 类型   | 说明                          |  
    |:--------------- |:-------|:----------------------------- |  
    | fangans[count]  | ARRAY  | count个分组方案信息组成的数组 |  
    
    其中“fangans[count]”数组的每个元素的结构为（Fields包含该字段时）：  
  
    | 名称          | 类型   | 说明                   |  
    |:------------- |:-------|:---------------------- |  
    | id            | int    | 分组方案的ID           |  
    | name          | string | 分组方案名称           |  
    | fenzus[count] | ARRAY  | count个分组构成的数组  |  
    | shanchu       | int    | 是否已删除             |  
    | xiugaitime    | date   | 最后修改时间           |  
    
    其中“fenzus[count]”数组的每个元素的结构为：
    
    | 名称             | 类型   | 说明                     |  
    |:---------------- |:-------|:------------------------ |  
    | id               | int    | 分组的ID                 |  
    | xueshengs[count] | int    | count个学生id组成的数组  |  
    | name             | string | 分组名称                 |  
  
* 说明  
    * 一个分组方案有多个分组，当Fields包含fenzus字段时，将返回分组方案的所有分组信息
    * 关于分组的结构和操作详见["2.4.分组"](#fenzu)
  
<span id="queryfangan"></span>  
#### 2.3.3. 查询分组方案  
* URI：/banjis/:bjid/kechengs/:kcid/fengzufangans/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、fenzus、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称          | 类型   | 说明                   |  
    |:------------- |:-------|:---------------------- |  
    | id            | int    | 分组方案的ID           |  
    | name          | string | 分组方案名称           |  
    | fenzus[count] | ARRAY  | count个分组构成的数组  |  
    | shanchu       | int    | 是否已删除             |  
    | xiugaitime    | date   | 最后修改时间           |  
    
    其中“fenzus[count]”数组的每个元素的结构为：
    
    | 名称             | 类型   | 说明                     |  
    |:---------------- |:-------|:------------------------ |  
    | id               | int    | 分组的ID                 |  
    | xueshengs[count] | int    | count个学生id组成的数组  |  
    | name             | string | 分组名称                 |  
    
* 说明  
    无  
  
#### 2.3.4. 修改分组方案  
* URI：/banjis/:bjid/kechengs/:kcid  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                         |  
    |:------ |:-------|:----:|:---------------------------- |  
    | name   | string | 否   | 分组方案名称                 |  
  
* 返回参数  
    无  
* 说明  
    * 本接口只支持修改分组方案的名称，对分组情况的修改应当使用["2.4.分组"](#fenzu)的增、改、删操作来完成 
  
#### 2.3.5. 删除分组方案  
* URI：/banjis/:bjid/kechengs/:kcid/fengzufangans/:id  
* 请求方法：DELETE  
* 请求参数  
    无  
* 返回参数  
    无  
* 说明  
    当删除一个分组方案时，首先将其shanchu字段置为1，并将该方案下所有分组shanchu字段置为1
  
  
<span id="fenzu"></span>  
### 2.4. 分组  
  
#### 2.4.1. 创建分组  
* URI：/banjis/:bjid/kechengs/:kcid/fenzufangans/:faid/fenzus  
* 请求方法：POST  
* 请求参数  
  
    | 名称             | 类型   | 必选 | 说明                     |  
    |:---------------- |:-------|:----:|:------------------------ |  
    | xueshengs[count] | int    | 是   | count个学生id组成的数组  |  
    | name             | string | 是   | 分组名称                 |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                         |  
    |:------ |:-------|:---------------------------- |  
    | id     | int    | 返回成功创建的分组的ID       |  
  
* 说明  
    * 创建分组时需要指定组内的学生id，且这些id不能出现在同一分组方案下的已有分组中
    * 一个分组方案下的所有分组构成对应班级下所有学生的一个[划分](http://baike.baidu.com/view/2797429.htm)
  
#### 2.4.2. 查询分组列表  
* URI：/banjis/:bjid/kechengs/:kcid/fenzufangans/:faid/fenzus  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、xueshengs、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段包括：id、name、shanchu、xiugaitime  
  
* 返回参数  
  
    | 名称           | 类型   | 说明                      |  
    |:-------------- |:-------|:------------------------- |  
    | fenzus[count]  | ARRAY  | count个分组信息组成的数组 |  
    
    其中“fenzus[count]”数组的每个元素的结构为（Fields包含该字段时）：  
  
    | 名称             | 类型   | 说明                    |  
    |:---------------- |:-------|:----------------------- |  
    | id               | int    | 分组的ID                |  
    | name             | string | 分组名称                |  
    | xueshengs[count] | ARRAY  | count个学生id组成的数组 |  
    | shanchu          | int    | 是否已删除              |  
    | xiugaitime       | date   | 最后修改时间            |  
  
* 说明  
    * 本接口与[2.3.3.查询分组方案](#queryfangan)均可以获取分组方案的所有分组，本接口侧重于对分组的条件查询，而后者侧重于对完整分组方案的查询。
  
#### 2.4.3. 查询分组  
* URI：/banjis/:bjid/kechengs/:kcid/fengzufangans/:faid/fenzus/:fzid  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、xueshengs、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称             | 类型   | 说明                    |  
    |:---------------- |:-------|:----------------------- |  
    | id               | int    | 分组的ID                |  
    | name             | string | 分组名称                |  
    | xueshengs[count] | ARRAY  | count个学生id组成的数组 |  
    | shanchu          | int    | 是否已删除              |  
    | xiugaitime       | date   | 最后修改时间            |  
    
* 说明  
    无  
  
#### 2.4.4. 修改分组  
* URI：/banjis/:bjid/kechengs/:kcid/fengzufangans/:faid/fenzus/:fzid  
* 请求方法：PUT  
* 请求参数  
  
    | 名称             | 类型   | 必选 | 说明                     |  
    |:---------------- |:-------|:----:|:------------------------ |  
    | xueshengs[count] | int    | 否   | count个学生id组成的数组  |  
    | name             | string | 否   | 分组名称                 |  
  
* 返回参数  
    无  
* 说明  
    * 若指定了xueshengs字段，将重新建立该分组的学生id列表，这些id不能出现在同一分组方案下的其他已有分组中 
  
#### 2.3.5. 删除分组  
* URI：/banjis/:bjid/kechengs/:kcid/fengzufangans/:faid/fenzus/:fzid  
* 请求方法：DELETE  
* 请求参数  
    无  
* 返回参数  
    无  
* 说明  
    无  
  
### 2.5. 用户  
  
#### 2.5.1. 添加用户  
* URI：/users  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                           |  
    |:------ |:-------|:----:|:------------------------------ |  
    | user   | string | 是   | 登录用户名                     |  
    | passwd | string | 是   | 登录密码的MD5字符串            |  
    | type   | int    | 是   | 用户类型，2：老师；3：学生     |  
    | banji  | int    | 否   | 所属班级ID（仅学生）           |  
    | name   | string | 是   | 姓名或者昵称，用来显示在界面上 |  
    | avatar | string | 否   | 用户头像图片                   |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 用户的ID               |  
  
* 说明  
    * 用户名不能出现重名，因此最好使用学生学号、教师教职工编号等  
  
#### 2.5.2. 查询用户列表  
* URI：/users  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、user、type、banji、name、avatar、createtime、kechengs、shanchu、xiugaitime、mac，默认只包含id  
    * Filter，支持的过滤字段包括：id、user、type、banji、name、createtime、shanchu、xiugaitime、mac
  
* 返回参数  
  
    | 名称         | 类型   | 说明                          |  
    |:------------ |:-------|:----------------------------- |  
    | users[count] | ARRAY  | count个用户信息组成的数组     |  
    
    其中“users[count]”数组的每个元素的结构为（Fields包含该字段时）：  
  
    | 名称             | 类型   | 说明                               |  
    |:--------------- |:-------|:----------------------------------- |  
    | id              | int    | 用户的ID                            |  
    | user            | string | 用户的登录用户名                    |  
    | type            | int    | 用户类型，2：老师；3：学生          |  
    | banji           | int    | 所属班级ID（仅学生）                |  
    | name            | string | 姓名或者昵称，用来显示在界面上      |  
    | avatar          | string | 用户头像图片                        |  
    | createtime      | date   | 用户帐号创建的时间                  |  
    | kechengs[count] | ARRAY  | count个课程信息组成的数组（仅老师） | 
    | mac             | int    | 用户设备mac地址，唯一标识每个用户   |
    | shanchu         | int    | 是否已删除                          |  
    | xiugaitime      | date   | 最后修改时间                        |     
    
    其中“kechengs[count]”数组的每个元素的结构为：  
  
    | 名称        | 类型   | 说明                   |  
    |:----------- |:-------|:---------------------- |  
    | banjiid     | int    | 班级的ID               |  
    | banjiname   | string | 班级名称               |  
    | kechengid   | int    | 班级课程的ID           |  
    | kechengname | string | 班级课程名称           |  
  
* 说明  
    无  
  
#### 2.5.3. 查询用户  
* URI：/users/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、user、type、banji、name、avatar、createtime、kechengs、dati、zuoye、pigai、shanchu、xiugaitime、mac，默认包含所有字段  
  
* 返回参数  
  
    | 名称            | 类型   | 说明                                |  
    |:--------------- |:-------|:----------------------------------- |  
    | user            | string | 用户的登录用户名                    |  
    | type            | int    | 用户类型，2：老师；3：学生          |  
    | banji           | int    | 所属班级ID（仅学生）                |  
    | name            | string | 姓名或者昵称，用来显示在界面上      |  
    | avatar          | string | 用户头像图片                        |  
    | createtime      | date   | 用户帐号创建的时间                  |  
    | kechengs[count] | ARRAY  | count个课程信息组成的数组（仅老师） |  
    | mac             | int    | 用户设备mac地址，唯一标识每个用户   |      
    | shanchu         | int    | 是否已删除                          |  
    | xiugaitime      | date   | 最后修改时间                        |     
    
    其中“kechengs[count]”数组的每个元素的结构为：  
  
    | 名称        | 类型   | 说明                   |  
    |:----------- |:-------|:---------------------- |  
    | banjiid     | int    | 班级的ID               |  
    | banjiname   | string | 班级名称               |  
    | kechengid   | int    | 班级课程的ID           |  
    | kechengname | string | 班级课程名称           |  
  
* 说明  
    无 
  
#### 2.5.4. 修改用户  
* URI：/users/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称       | 类型   | 必选 | 说明                             |  
    |:---------- |:-------|:----:|:-------------------------------- |  
    | user       | string | 否   | 登录用户名                       |  
    | passwd     | string | 否   | 登录密码的MD5字符串              |  
    | type       | int    | 否   | 用户类型，2：老师；3：学生       |  
    | banji      | int    | 否   | 所属班级ID                       |  
    | name       | string | 否   | 姓名或者昵称，用来显示在界面上   |
    | avatar     | string | 否   | 用户头像图片                     |  
    | mac        | int    | 否   | 用户设备mac地址，唯一标识每个用户|  
  
* 返回参数  
    无  
* 说明  
    * 每次登陆必须要调用put /users/:id这个接口，并只以mac作为参数；这个操作会把用户设备的mac地址传到数据库中；  
  
#### 2.5.5. 删除用户  
* URI：/users/:id  
* 请求方法：DELETE 
* 请求参数  
    无
* 返回参数  
    无  
* 说明  
    无  
  
<span id="login"></span>  
### 2.6. 登录   
  
#### 2.6.1. 登录  
* URI：/logins  
* 请求方法：POST  
* 请求参数  
    参见说明部分。  
* 返回参数
    
    | 名称            | 类型   | 说明                                           |  
    |:--------------- |:-------|:---------------------------------------------- |  
    | id              | int    | 用户的ID                                       |  
    | token           | string | 用来在后续请求中标识用户身份的令牌             |  
    | expire          | int    | 过期时间，从1970-01-01T00:00:00Z UTC开始的秒数 |  

* 说明  
    * 用户名、密码通过HTTP摘要认证机制来传递。用户名和密码对应于HTTP摘要认证的用户名和密码  
    * token包含此次登录的令牌，在调用其他接口时，必须携带此令牌，参见[1.2]  
  
#### 2.6.2. 查询登录用户列表  
* URI：/logins  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：token、id、ip、login、expire、last，默认只包含token  
    * Filter，支持的过滤字段同上  
  
* 返回参数  
  
    | 名称          | 类型   | 说明                          |  
    |:------------- |:-------|:----------------------------- |  
    | logins[count] | ARRAY  | count个用户信息组成的数组     |  
    
    其中“logins[count]”数组的每个元素的结构为：  
  
    | 名称    | 类型   | 说明                             |  
    |:------- |:-------|:-------------------------------- |  
    | token   | string | 系统为该登录签发的访问令牌       |  
    | id      | int    | 用户的ID                         |  
    | ip      | string | 用户登录的IP地址                 |  
    | login   | date   | 登录时间                         |  
    | expire  | date   | 访问令牌过期时间                 |  
    | last    | date   | 最后活动时间                     |  
  
* 说明  
    * 普通用户只能查询到本用户的登录；管理员账户可以查询所有登录  
    * 由于允许同一个帐号在多个终端上同时登录（可能会限制最大同时登录数），因此返回的列表中同一个用户ID可能会有多个记录，每一个记录对应于一个登录  
  
#### 2.6.3. 查询登录用户  
* URI：/logins/:token  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：token、id、ip、login、expire、last，默认包括所有字段  
  
* 返回参数  
  
    | 名称    | 类型   | 说明                             |  
    |:------- |:-------|:-------------------------------- |  
    | token   | string | 系统为该登录签发的访问令牌       |  
    | id      | int    | 用户的ID                         |  
    | ip      | string | 用户登录的IP地址                 |  
    | login   | date   | 登录时间                         |  
    | expire  | date   | 访问令牌过期时间                 |  
    | last    | date   | 最后活动时间                     |  
  
* 说明  
    * 普通用户只能查询到本用户帐号的登录；管理员账户可以查询所有登录  
  
#### 2.6.4. 登出（**不支持**）  
* URI：/logins/:token  
* 请求方法：DELETE 
* 请求参数 
    无
* 返回参数  
    无  
* 说明  
    * 登出需要服务端维护token白名单或者黑名单。因此暂**不支持**登出操作 
    
#### 2.6.5. 查询学生联网状态  
* URI：/xueshengstate/:bjid  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、ip、login、last，默认包括所有字段  
  
* 返回参数  

    | 名称             | 类型   | 说明                          |  
    |:---------------- |:-------|:----------------------------- |  
    | xueshengs[count] | ARRAY  | count个学生信息组成的数组     |  
    
    其中“xueshengs[count]”数组的每个元素的结构为：  
  
    | 名称    | 类型   | 说明                             |  
    |:------- |:-------|:-------------------------------- |  
    | id      | int    | 用户的ID                         |  
    | ip      | string | 用户登录的IP地址                 |  
    | login   | date   | 登录时间                         |  
    | last    | date   | 最后活动时间                     |  
  
* 说明  
    * 本接口提供查询学生联网状态的服务，限管理员和老师访问  
    * URI中的bjid为班级ID，访问本接口是将返回对应班级的所有学生的联网信息
  
### 2.7. 通知消息与数据更新  
  
#### 2.7.1 获取通知消息与数据更新  
* URI：/tongzhis/:uid  
* 请求方法：GET  
* 请求参数  
    
    | 名称       | 类型       | 说明                                          | 必选 |  
    |:---------- |:---------- |:--------------------------------------------- |:---- |  
    | timeout    | int        | 超时值，单位为毫秒                            |  否  |
    | dativer    | int        | 客户端当前答题通知消息的版本                  |  否  |  
    | zuoyever   | int        | 客户端当前作业通知消息的版本                  |  否  |  
    | pigaiver   | int        | 客户端当前批改通知消息的版本                  |  否  |  
    | timuver    | int        | 客户端当前题目通知消息的版本                  |  否  |  
    | renwuver   | int        | 客户端当前教学计划与教学任务更新通知消息的版本|  否  |  
    | kechengver | int        | 客户端当前课程更新通知消息的版本              |  否  |  
    | tixingver  | int        | 客户端当前题型更新通知消息的版本              |  否  |  
    | shoucangver| int        | 客户端当前收藏更新通知消息的版本              |  否  |  
    | fenzuver   | int        | 客户端当前分组更新通知消息的版本              |  否  |  
    
* 返回参数  
  
    | 名称       | 类型       | 说明                                          |  
    |:---------- |:---------- |:--------------------------------------------- |  
    | dativer    | int        | 服务端当前答题通知消息的版本                  |  
    | zuoyever   | int        | 服务端当前作业通知消息的版本                  |  
    | pigaiver   | int        | 服务端当前批改通知消息的版本                  |  
    | timuver    | int        | 服务端当前题目通知消息的版本                  |  
    | renwuver   | int        | 服务端当前教学计划与教学任务更新通知消息的版本|  
    | kechengver | int        | 服务端当前课程更新通知消息的版本              |  
    | tixingver  | int        | 服务端当前题型更新通知消息的版本              |  
    | shoucangver| int        | 服务端当前收藏更新通知消息的版本              |  
    | fenzuver   | int        | 服务端当前分组更新通知消息的版本              |  
  
* 说明  
    * 请求URI中的:uid参数表示用户ID  
    * 通知消息的获取采用[1.5.长轮询模式](#longpolling)  
    * 客户端需要请求某些通知消息的版本时，将本地的这些消息版本传递至服务端，服务端在接下来的一段时间(timeout)检测服务端的这些消息版本是否更新，检测到某一版本消息更新时返回所有该客户端请求的通知消息版本
    * 客户端根据收到新的消息版本时请求并更新相应数据
  
  
## 3. 题库管理  
  
### 3.1. 教学计划  
  
#### 3.1.1. 添加教学计划  
* URI：/jiaoxuejihuas  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                                 |  
    |:------ |:-------|:----:|:------------------------------------ |  
    | name   | string | 是   | 教学计划的名称，如“数学七年级下学期” |  
    | icon   | string | 否   | 教学计划图标                         |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 教学计划的ID           |  
  
* 说明  
    无  
  
#### 3.1.2. 查询教学计划列表  
* URI：/jiaoxuejihuas  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、icon、renwus、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的字段包括：id、name、shanchu、xiugaitime
  
* 返回参数  
  
    | 名称          | 类型   | 说明                          |  
    |:------------- |:-------|:----------------------------- |  
    | jihuas[count] | ARRAY  | count个教学计划信息组成的数组 |  
    
    其中“jihuas[count]”数组的每个元素的结构为：  
  
    | 名称       | 类型   | 说明                                       |  
    |:---------- |:-------|:------------------------------------------ |  
    | id         | int    | 教学计划的ID                               |  
    | name       | string | 教学计划的名称，如“数学七年级下学期”       |  
    | icon       | string | 教学计划图标                               |  
    | rewus      | ARRAY  | 该教学计划下的所有教学任务(嵌套的对象数组) |  
    | shanchu    | int    | 是否已删除                                 |  
    | xiugaitime | date   | 最后修改时间                               |     
  
* 说明  
    * 返回的每个教学计划是树状结构，由嵌套的对象表示；rewus包含其所有子节点（教学任务），每个子节点包含name和children两个字段，name为教学任务名称，children包含该子节点的所有子节点。
    * 一个教学计划示例如下：
    
            {
                id: 1,
                name: "数学七年级下",
                renwus: [
                    {
                        id: 1,
                        name: "整式的乘除",
                        children: [
                            {
                                id: 2,
                                name: "平方差公式",
                                children: []
                            },
                            {
                                id: 3,
                                name: "同底数幂相乘",
                                children: []
                            },
                            ...
                        ]
                    },
                    {
                        id: 18,
                        name: "相交线与平行线",
                        children: []
                    },
                    ...
                ]
            }

<span id="queryjihua"></span>
#### 3.1.3. 查询教学计划  
* URI：/jiaoxuejihuas/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、icon、renwus、shanchu、xiugaitime，默认包含所有字段  
    
* 返回参数  
  
    | 名称       | 类型   | 说明                                       |  
    |:---------- |:-------|:------------------------------------------ |  
    | id         | int    | 教学计划的ID                               |  
    | name       | string | 教学计划的名称，如“数学七年级下学期”       |  
    | icon       | string | 教学计划图标                               |  
    | rewus      | ARRAY  | 该教学计划下的所有教学任务(嵌套的对象数组) |  
    | shanchu    | int    | 是否已删除                                 |  
    | xiugaitime | date   | 最后修改时间                               |     
  
* 说明  
    返回的数据结构详见上一小节的说明
  
#### 3.1.4. 修改教学计划  
* URI：/jiaoxuejihuas/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                                 |  
    |:------ |:-------|:----:|:------------------------------------ |  
    | name   | string | 否   | 教学计划的名称，如“数学七年级下学期” |  
    | icon   | string | 否   | 教学计划图标                         |  
    
* 返回参数  
    无  
* 说明  
    无  
  
#### 3.1.5. 删除教学计划  
* URI：/jiaoxuejihuas/:id  
* 请求方法：DELETE  
* 请求参数
    无
* 返回参数  
    无  
* 说明  
    * 只有当一个教学计划下没有任何教学任务时，才可以删除这个教学计划。（同样，只有当一个教学任务没有被任何题库中的题目标记为所属时，才能删除教学任务。也就是说要删除一个教学计划，必须先删除所有属于它的题目，然后在删除所有属于他的教学任务）


### 3.2. 教学任务  
  
#### 3.2.1. 添加教学任务  
* URI：/jiaoxuejihuas/:jihuaid/jiaoxuerenwus
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                                                   |  
    |:------ |:-------|:----:|:------------------------------------------------------ |  
    | name   | string | 是   | 教学任务的名称，一般为具体的章节名称，如“平行线的性质” |  
    | jihua  | int    | 是   | 当前教学任务所属的教学计划的ID                       |  
    | parent | int    | 是   | 当前教学任务的父教学任务节点的ID，若无父教学任务节点，则此ID为-1 |  
    | pos    | int    | 是   | 一个表示插入位置的ID，当前教学任务要插入到pos所指的那个教学任务的前面，如果pos为-1则表示插入到末尾 |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 教学任务的ID           |  
  
* 说明  
    无  
  
#### 3.2.2. 查询教学任务列表  
* URI：/jiaoxuejihuas/:jihuaid/jiaoxuerenwus
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、jihua、parent、next、head、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段同上  
    
* 返回参数  
  
    | 名称          | 类型   | 说明                          |  
    |:------------- |:-------|:----------------------------- |  
    | renwus[count] | ARRAY  | count个教学任务信息组成的数组 |  
    
    其中“renwus[count]”数组的每个元素的结构为：  
  
    | 名称       | 类型   | 说明                                                                |  
    |:---------- |:-------|:------------------------------------------------------------------- |  
    | id         | int    | 教学任务的ID                                                        |  
    | name       | string | 教学任务的名称，如“平行线的性质”                                    |  
    | jihua      | int    | 所属教学计划的ID                                                    |  
    | parent     | int    | 当前教学任务的父教学任务节点的ID，若无父教学任务节点，则此ID为-1    |  
    | next       | int    | 当前教学任务的下一个教学任务节点的ID，若无则此ID为-1                |  
    | head       | int    | 当前教学任务的第一个子教学任务的ID，若无则此ID为-1                  |
    | shanchu    | int    | 是否已删除                                                          |  
    | xiugaitime | date   | 最后修改时间                                                        |     
  
* 说明  
    本接口用于针对教学任务的条件查询，获取一个教学计划的所有教学任务应通过[3.1.3.查询教学计划](#queryjihua)接口得到树状的JSON数据  
  
#### 3.2.3. 查询教学任务  
* URI：/jiaoxuejihuas/:jihuaid/jiaoxuerenwus/:id
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、jihua、parent、next、head、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称      | 类型   | 说明                                                                 |  
    |:--------- |:-------|:-------------------------------------------------------------------- |  
    | id        | int    | 教学任务的ID                                                         |  
    | name      | string | 教学任务的名称，如“平行线的性质”                                     |  
    | jihua     | int    | 所属教学计划的ID                                                     |  
    | parent    | int    | 当前教学任务的父教学任务节点的ID，若无父教学任务节点，则此ID为-1     |  
    | next      | int    | 当前教学任务的下一个教学任务节点的ID，若无则此ID为-1                 |  
    | head      | int    | 当前教学任务的第一个子教学任务的ID，若无则此ID为-1                   | 
    | shanchu   | int    | 是否已删除                                                           |  
    | xiugaitime| date   | 最后修改时间                                                         |     
  
* 说明  
    无  
  
#### 3.2.4. 修改教学任务  
* URI：/jiaoxuejihuas/:jihuaid/jiaoxuerenwus/:id
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                             |  
    |:------ |:-------|:----:|:-------------------------------- |  
    | name   | string | 否   | 教学任务名称，如：“平行线的性质” |  
    
* 返回参数  
    无  
* 说明  
    * 不能改变树结构，所以修改教学任务时不能修改jihua、parent、next和head；    
  
#### 3.2.5. 删除教学任务  
* URI：/jiaoxuejihuas/:jihuaid/jiaoxuerenwus/:id
* 请求方法：DELETE  
* 请求参数   
    无
* 返回参数  
    无  
* 说明  
    * 只有当题库里面没有任何题目标记为属于这个教学计划时，才可以删除教学计划
    * 当要删除的教学任务下没有子教学任务时才能删除，删除教学任务时必须要对指向该教学任务的的教学任务做修改；


### 3.3. 题型  
  
#### 3.3.1. 添加题型  
* URI：/tixings  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                                 |  
    |:------ |:-------|:----:|:------------------------------------ |  
    | name   | string | 是   | 题型名称，如：“选择题”               |  
    | jihua  | int    | 是   | 所属教学计划的ID                     |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 题型的ID               |  
  
* 说明  
    无  
  
#### 3.3.2. 查询题型列表  
* URI：/tixings  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、jihua、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段同上  
    
* 返回参数  
  
    | 名称           | 类型   | 说明                          |  
    |:-------------- |:-------|:----------------------------- |  
    | tixings[count] | ARRAY  | count个题型信息组成的数组     |  
    
    其中“tixings[count]”数组的每个元素的结构为：  
  
    | 名称       | 类型   | 说明                                 |  
    |:---------- |:-------|:------------------------------------ |  
    | id         | int    | 题型的ID                             |  
    | name       | string | 题型的名称，如“选择题”               |  
    | jihua      | int    | 所属教学计划的ID                     |  
    | shanchu    | int    | 是否已删除                           |  
    | xiugaitime | date   | 最后修改时间                         |    
  
* 说明  
    无  
  
#### 3.3.3. 查询题型  
* URI：/tixings/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、jihua、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称       | 类型   | 说明                                 |  
    |:---------- |:-------|:------------------------------------ |  
    | id         | int    | 题型的ID                             |  
    | name       | string | 题型的名称，如“选择题”               |  
    | jihua      | int    | 所属教学计划的ID                     |
    | shanchu    | int    | 是否已删除                           |  
    | xiugaitime | date   | 最后修改时间                         |      
  
* 说明  
    无  
  
#### 3.3.4. 修改题型  
* URI：/tixings/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                           |  
    |:------ |:-------|:----:|:------------------------------ |  
    | name   | string | 是   | 题型名称，如：“选择题”         |  
    | jihua  | int    | 是   | 所属教学计划的ID               |  
  
* 返回参数  
    无  
* 说明  
    无  
  
#### 3.3.5. 删除题型  
* URI：/tixings/:id  
* 请求方法：DELETE  
* 请求参数  
    无  
* 返回参数  
    无  
* 说明  
    无
 
  
### 3.4. 题目  
  
#### 3.4.1. 添加题目  
* URI：/timus  
* 请求方法：POST  
* 请求参数  
  
    | 名称          | 类型   | 必选 | 说明                                       |  
    |:------------- |:-------|:----:|:------------------------------------------ |  
    | tigantxt      | string | 是   | 题干的文字部分，例如：“计算(3x+7y)(3x-7y)” |  
    | tiganpic      | string | 否   | 题干的图片部分，具体格式参见说明           |  
    | jietiguocheng | string | 否   | 参考解题过程的图片                         |  
    | youdaan       | bool   | 是   | 客户端是否渲染答案区域                     |  
    | daanpic       | string | 否   | 图片版答案                                 | 
    | daantxt       | string | 否   | 文字版答案                                 |    
    | tixing        | int    | 是   | 题型的ID                                   |  
    | nandu         | int    | 是   | 难度：1(最简单）～5（最难）                |  
    | beizhutxt     | string | 否   | 备注的文字部分                             |  
    | beizhupic     | string | 否   | 备注的图片部分                             |  
    | jiaoxuejihua  | int    | 是   | 教学计划的ID                               |  
    | jiaoxuerenwu  | int    | 是   | 教学任务的索引号                           |  
    | user          | int    | 是   | 添加题目的用户的ID                         |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 题目的ID               |  
  
* 说明  
    * 图片是二进制数据，为了在HTTP协议中传输，使用BASE64编码算法编码为可见字符串；  
    * 如果有多张图片，请使用tar将图片打包为一个文件，然后用上面的BASE64方法转换成字符串。  
  
#### 3.4.2. 查询题目列表  
* URI：/timus  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、tigantxt、tiganpic、jietiguocheng、youdaan、daanpic、daantxt、tixing、nandu、beizhutxt、beizhupic、jiaoxuejihua、jiaoxuerenwu、shanchu、buzhicishu、time、user、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段包括：id、tigantxt、youdaan、tixing、nandu、beizhutxt、jiaoxuejihua、jiaoxuerenwu、buzhicishu、time、user、shanchu、xiugaitime  
  
* 返回参数  
  
    | 名称         | 类型   | 说明                          |  
    |:------------ |:-------|:----------------------------- |  
    | timus[count] | ARRAY  | count个题目信息组成的数组     |  
    
    其中“timus[count]”数组的每个元素可以包含下列字段（当Fields包含该字段时）：  
  
    | 名称          | 类型   | 说明                        |  
    |:------------- |:-------|:--------------------------- |  
    | id            | int    | 题目的ID                    |  
    | tigantxt      | string | 题干的文字部分              |  
    | tiganpic      | string | 题干的图片部分              |  
    | jietiguocheng | string | 参考解题过程的图片          |  
    | youdaan       | bool   | 客户端是否渲染答案区域      |  
    | daanpic       | string | 图片版的答案                |  
    | daantxt       | string | 文字版的答案                |  
    | tixing        | int    | 题型的ID                    |  
    | nandu         | int    | 难度：1(最简单）～5（最难） |  
    | beizhutxt     | string | 备注的文字部分              |  
    | beizhupic     | string | 备注的图片部分              |  
    | jiaoxuejihua  | int    | 教学计划的ID                |  
    | jiaoxuerenwu  | int    | 教学任务的索引号            |  
    | buzhicishu    | int    | 题目被布置的总次数          |  
    | time          | date   | 添加时间                    |  
    | user          | int    | 添加题目的用户的ID          |
    | shanchu       | int    | 是否已删除                  |  
    | xiugaitime    | date   | 最后修改时间                |  
  
* 说明  
    无  
  
#### 3.4.3. 查询题目  
* URI：/timus/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、tigantxt、tiganpic、jietiguocheng、youdaan、daanpic、daantxt、tixing、nandu、beizhutxt、beizhupic、jiaoxuejihua、jiaoxuerenwu、shanchu、buzhicishu、time、user、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称          | 类型   | 说明                        |  
    |:------------- |:-------|:--------------------------- |  
    | id            | int    | 题目的ID                    |  
    | tigantxt      | string | 题干的文字部分              |  
    | tiganpic      | string | 题干的图片部分              |  
    | jietiguocheng | string | 参考解题过程的图片          |  
    | youdaan       | bool   | 客户端是否渲染答案区域      |  
    | daanpic       | string | 图片版的答案                |
    | daantxt       | string | 文字版的答案                |  
    | tixing        | int    | 题型的ID                    |  
    | nandu         | int    | 难度：1(最简单）～5（最难） |  
    | beizhutxt     | string | 备注的文字部分              |  
    | beizhupic     | string | 备注的图片部分              |  
    | jiaoxuejihua  | int    | 教学计划的ID                |  
    | jiaoxuerenwu  | int    | 教学任务的索引号            |  
    | buzhicishu    | int    | 题目被布置的总次数          |  
    | time          | date   | 添加时间                    |  
    | user          | int    | 添加题目的用户的ID          | 
    | shanchu       | int    | 是否已删除                  |  
    | xiugaitime    | date   | 最后修改时间                |  
  
* 说明  
    无  
  
#### 3.4.4. 修改题目  
* URI：/timus/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称          | 类型   | 必选 | 说明                                       |  
    |:------------- |:-------|:----:|:------------------------------------------ |  
    | tigantxt      | string | 是   | 题干的文字部分，例如：“计算(3x+7y)(3x-7y)” |  
    | tiganpic      | string | 否   | 题干的图片部分，具体格式参见说明           |  
    | jietiguocheng | string | 否   | 参考解题过程的图片                         |  
    | youdaan       | bool   | 是   | 客户端是否渲染答案区域                     |  
    | daanpic       | string | 否   | 图片版的答案                               |
    | daantxt       | string | 否   | 文字版的答案                               |
    | tixing        | int    | 是   | 题型的ID                                   |  
    | nandu         | int    | 否   | 难度：1(最简单）～5（最难）                |  
    | beizhutxt     | string | 否   | 备注的文字部分                             |  
    | beizhupic     | string | 否   | 备注的图片部分                             |  
    | jiaoxuejihua  | int    | 是   | 教学计划的ID                               |  
    | jiaoxuerenwu  | int    | 是   | 教学任务的索引号                           |
    | user          | int    | 否   | 添加题目的用户的ID                         |     
  
* 返回参数  
    无  
* 说明  
    无  
  
#### 3.4.5. 删除题目  
* URI：/timus/:id  
* 请求方法：DELETE  
* 请求参数   
    无  
* 返回参数  
    无  
* 说明  
    无
  
  
## 4. 答题与批改  
  
### 4.1. 作业清单  
  
#### 4.1.1. 创建作业清单  
* URI：/zuoyeqingdans  
* 请求方法：POST  
* 请求参数  
  
    | 名称         | 类型   | 必选 | 说明                      |  
    |:------------ |:-------|:----:|:------------------------- |  
    | timus[count] | ARRAY  | 是   | count个题目信息组成的数组 |  
    | banji        | int    | 是   | 作业清单针对的班级ID      |  
    | kecheng      | int    | 是   | 作业清单针对的课程ID      |  
    | user         | int    | 是   | 发布作业清单的老师的ID    |  
    | fabutime     | date   | 是   | 作业清单的发布时间        |  
    | beizhu       | string | 否   | 作业清单的文字备注        |  
    
    其中timus[count]的每一个元素为题目信息对象，其结构为
    
    | 名称         | 类型   | 必选 | 说明                      |  
    |:------------ |:-------|:----:|:------------------------- |  
    | id           | int    | 是   | 题目的ID                  |  
    | description  | string | 是   | 该题的描述子              |  
    | values[count]| ARRAY  | 否   | count个分值组成的数组     |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                       |  
    |:------ |:-------|:-------------------------- |  
    | id     | int    | 返回成功创建作业清单的ID   |  
  
* 说明  
    * 题目id与描述子确定学生需要作答的空数，values数组（若指定）的长度应与其相等；不指定values数组时该清单按对错批改
    * 返回的ID用来在对已经创建的作业清单进行其他操作时唯一的标识一个作业清单  
  
#### 4.1.2. 查询作业清单列表  
* URI：/zuoyeqingdans  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、timus、banji、kecheng、user、fabutime、beizhu、yijiaoshu、tijiaotime、shanchu、xiugaitime，默认只包含id；
    * Filter，支持的过滤字段包含除timus的所有字段；  
  
* 返回参数  
  
    | 名称                 | 类型   | 说明                          |  
    |:-------------------- |:-------|:----------------------------- |  
    | zuoyeqingdans[count] | ARRAY  | count个作业清单信息组成的数组 |  
    
    其中“zuoyeqingdans[count]”数组的每个元素的结构为：  
  
    | 名称                  | 类型   | 说明                       |  
    |:-----------------     |:-------|:-------------------------  |  
    | id                    | int    | 作业清单的ID               |  
    | timus[count]          | ARRAY  | count个题目信息组成的数组  |  
    | banji                 | int    | 作业清单针对的班级ID       |  
    | kecheng               | int    | 作业清单针对的课程ID       |  
    | user                  | int    | 发布作业清单的老师的ID     |  
    | fabutime              | date   | 作业清单的发布时间         |  
    | beizhu                | string | 作业清单的文字备注         |  
    | yijiaoshu             | int    | 已交齐作业的学生人数       |  
    | tijiaotime            | date   | 最后提交作业的提交时间     |  
    | shanchu               | int    | 是否已删除                 |  
    | xiugaitime            | date   | 最后修改时间               |  
    
    其中timus[count]的每一个元素为题目信息对象，其结构为
    
    | 名称         | 类型   | 说明                      |  
    |:------------ |:-------|:------------------------- |  
    | id           | int    | 题目的ID                  |  
    | description  | string | 该题的描述子              |  
    | values[count]| ARRAY  | count个分值组成的数组     |  
  
* 说明    
    * 返回信息中的user字段是创建这个作业清单的用户的id
    * values字段可为空数组，表示该题按对错判题
    
#### 4.1.3. 查询作业清单  
* URI：/zuoyeqingdans/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、timus、banji、kecheng、user、fabutime、beizhu、yijiaoshu、tijiaotime、shanchu、xiugaitime，默认包含所有字段；   
  
* 返回参数  
  
    | 名称                  | 类型   | 说明                       |  
    |:-----------------     |:-------|:-------------------------  |  
    | id                    | int    | 作业清单的ID               |  
    | timus[count]          | ARRAY  | count个题目信息组成的数组  |  
    | banji                 | int    | 作业清单针对的班级ID       |  
    | kecheng               | int    | 作业清单针对的课程ID       |  
    | user                  | int    | 发布作业清单的老师的ID     |  
    | fabutime              | date   | 作业清单的发布时间         |  
    | beizhu                | string | 作业清单的文字备注         |  
    | yijiaoshu             | int    | 已交齐作业的学生人数       |  
    | tijiaotime            | date   | 最后提交作业的提交时间     |  
    | shanchu               | int    | 是否已删除                 |  
    | xiugaitime            | date   | 最后修改时间               |  
    
    其中timus[count]的每一个元素为题目信息对象，其结构为
    
    | 名称         | 类型   | 说明                      |  
    |:------------ |:-------|:------------------------- |  
    | id           | int    | 题目的ID                  |  
    | description  | string | 该题的描述子              |  
    | values[count]| ARRAY  | count个分值组成的数组     |  
  
* 说明  
    无  
  
#### 4.1.4. ~~修改作业清单~~(不可用)  
* 说明
    由于修改清单会造成批改过的此清单的答题记录难以处理，暂不可用
  
#### 4.1.5. ~~删除作业清单~~(不可用)  
* 说明
    由于删除清单会造成批改过的此清单的答题记录难以处理，暂不可用
  
### 4.2. 答题记录  
  
#### 4.2.1. 创建答题记录  
* URI：/datijilus  
* 请求方法：POST  
* 请求参数  
  
    | 名称         | 类型   | 必选 | 说明                      |  
    |:------------ |:-------|:-----|:------------------------- |  
    | xuesheng     | int    | 是   | 答题学生的ID              |  
    | timu         | int    | 是   | 题目的ID                  |  
    | zuoyeqingdan | int    | 否   | 作业清单的ID              |  
    | guocheng     | string | 否   | 解题过程的图片            |  
    | daan         | string | 否   | 答案的图片                |  
    | daantxt      | string | 否   | 选择题答案                |  
    | zuotitime    | date   | 是   | 做题时间                  |    
    | ver          | int    | 是   | 答题记录的版本号          |
* 返回参数  
  
    | 名称   | 类型   | 说明                       |  
    |:------ |:-------|:-------------------------- |  
    | id     | int    | 返回成功创建答题记录的ID   |  
  
* 说明  
    * 返回的ID用来在对已经创建的答题记录进行其他操作时唯一的标识一个答题记录  
    * 只有学生用户能够创建答题记录  
    * ver表示答题记录的版本号，由客户端维护；ver为0表示对作业清单的提交，否则表示对错题的提交，错题可以重做多次，每次重做ver值+1
  
#### 4.2.2. 查询答题记录列表  
* URI：/datijilus  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、xuesheng、timu、zuoyeqingdan、guocheng、daan、daantxt、zuotitime、pigai、pigaijieguo、pigaipizhu、pigaitime、pigaiyidu、shanchu、xiugaitime、ver，默认只包含id  
    * Filter，支持的过滤字段包括：id、xuesheng、timu、zuoyeqingdan、zuotitime、pigaijieguo、pigaitime、pigaiyidu、shanchu、xiugaitime、ver  
  
* 返回参数  
  
    | 名称             | 类型   | 说明                          |  
    |:---------------- |:-------|:----------------------------- |  
    | datijilus[count] | ARRAY  | count个答题记录信息组成的数组 |  
    
    其中“datijilus[count]”数组的每个元素的结构为：  
  
    | 名称         | 类型   | 说明                      |  
    |:------------ |:-------|:------------------------- |  
    | id           | int    | 答题记录的ID              |  
    | xuesheng     | int    | 答题学生的ID              |  
    | timu         | int    | 题目的ID                  |  
    | zuoyeqingdan | int    | 作业清单的ID              |  
    | guocheng     | string | 解题过程的图片            |  
    | daan         | string | 答案的图片                |  
    | daantxt      | string | 选择题答案                |  
    | zuotitime    | date   | 做题时间                  |  
    | pigai        | bool   | 老师是否已经批改          |  
    | pigaijieguo  | bool   | 老师的批改结果            |  
    | pigaipizhu   | string | 老师批注的图片            |  
    | pigaitime    | date   | 老师批改时间              |  
    | pigaiyidu    | bool   | 学生已读批改结果          |  
    | shanchu      | int    | 是否已删除                |  
    | xiugaitime   | date   | 最后修改时间              |  
    | ver          | int    | 答题记录的版本号          |  
  
* 说明  
    无  
  
#### 4.2.3. 查询答题记录  
* URI：/datijilus/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、xuesheng、timu、zuoyeqingdan、guocheng、daan、daantxt、zuotitime、pigai、pigaijieguo、pigaipizhu、pigaitime、pigaiyidu、shanchu、xiugaitime、ver，默认包含所有字段  
  
* 返回参数  
  
    | 名称         | 类型   | 说明                      |  
    |:------------ |:-------|:------------------------- |  
    | id           | int    | 答题记录的ID              |  
    | xuesheng     | int    | 答题学生的ID              |  
    | timu         | int    | 题目的ID                  |  
    | zuoyeqingdan | int    | 作业清单的ID              |  
    | guocheng     | string | 解题过程的图片            |  
    | daan         | string | 答案的图片                |  
    | daantxt      | string | 选择题答案                |  
    | zuotitime    | date   | 做题时间                  |  
    | pigai        | bool   | 老师是否已经批改          |  
    | pigaijieguo  | bool   | 老师的批改结果            |  
    | pigaipizhu   | string | 老师批注的图片            |  
    | pigaitime    | date   | 老师批改时间              |  
    | pigaiyidu    | bool   | 学生已读批改结果          |
    | shanchu      | int    | 是否已删除                |  
    | xiugaitime   | date   | 最后修改时间              |
    | ver          | int    | 答题记录的版本号          |
  
* 说明  
    无
  
#### 4.2.4. 修改答题记录  
* URI：/datijilus/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称         | 类型   | 必选 | 说明                       |  
    |:------------ |:-------|:---- |:-------------------------- |  
    | pigaijieguo  | bool   | 是1  | 老师的批改结果（仅老师）   |  
    | pigaipizhu   | string | 否1  | 老师批注的图片（仅老师）   |  
    | pigaitime    | date   | 是1  | 老师批改时间（仅老师）     |  
    | pigaiyidu    | bool   | 是2  | 学生已读批改结果（仅学生） |  
  
* 返回参数  
    无  
* 说明  
    * 1对于老师用户来说，这些参数是必选或非必选；2对于学生用户来说这些参数是必选或非必选  
  
#### 4.2.5. 删除答题记录  
* URI：/datijilus/:id  
* 请求方法：DELETE
* 请求参数
    无
* 返回参数  
    无  
* 说明  
    无  
  
  
## 5. 收藏夹

### 5.1 收藏目录 
  
#### 5.1.1. 添加收藏目录  
* URI：/users/:uid/shoucangmulus  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明         |  
    |:------ |:-------|:----:|:------------ |  
    | name   | string | 是   | 收藏目录名称 |  
    | type   | int    | 是   | 收藏目录类型 |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明                   |  
    |:------ |:-------|:---------------------- |  
    | id     | int    | 创建的收藏目录的ID     |  
  
* 说明  
    * type为目录类型，0：答题记录收藏目录, 1：题目收藏目录 
    * 用户可以建立多个收藏目录；老师可以建立两种类型的收藏目录，学生只能建立答题记录类型的收藏目录
    * 目前仅支持一级目录，多级目录如有需要再做支持
  
#### 5.1.2. 查询收藏目录列表  
* URI：/users/:uid/shoucangmulus  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、user、type、shoucangs、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的过滤字段包括：id、name、user、type、shanchu、xiugaitime
  
* 返回参数  
  
    | 名称                 | 类型   | 说明                          |  
    |:-------------------- |:-------|:----------------------------- |  
    | shoucangmulus[count] | ARRAY  | count个收藏目录组成的数组     |  
    
    其中“shoucangmulus[count]”数组的每个元素的结构为（Fields包含该字段时）：  
  
    | 名称             | 类型   | 说明                                 |  
    |:---------------- |:-------|:------------------------------------ |  
    | id               | int    | 收藏目录的ID                         |  
    | user             | string | 收藏目录所属用户的ID                 |  
    | name             | string | 收藏目录的名称                       |  
    | type             | int    | 收藏目录的类型，0：答题记录；1：题目 |  
    | shoucangs[count] | ARRAY  | count个收藏信息组成的数组            | 
    | shanchu          | int    | 是否已删除                           |
    | xiugaitime       | date   | 最后修改时间                         |     
    
    其中“shoucangs[count]”数组的每个元素的结构为：  
  
    | 名称        | 类型   | 说明                                                             |  
    |:----------- |:-------|:---------------------------------------------------------------- |  
    | id          | int    | 收藏的ID                                                         |  
    | item        | string | 收藏对象的ID(若收藏目录类型为题目，则为题目ID，否则为答题记录ID) |  
  
* 说明  
    无  
  
#### 5.1.3. 查询收藏目录  
* URI：/users/:uid/shoucangmulus/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、user、type、shoucangs、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称             | 类型   | 说明                                 |  
    |:---------------- |:-------|:------------------------------------ |  
    | id               | int    | 收藏目录的ID                         |  
    | user             | string | 收藏目录所属用户的ID                 |  
    | name             | string | 收藏目录的名称                       |  
    | type             | int    | 收藏目录的类型，0：答题记录；1：题目 |  
    | shoucangs[count] | ARRAY  | count个收藏信息组成的数组            | 
    | shanchu          | int    | 是否已删除                           |
    | xiugaitime       | date   | 最后修改时间                         |     
    
    其中“shoucangs[count]”数组的每个元素的结构为：  
  
    | 名称        | 类型   | 说明                                                             |  
    |:----------- |:-------|:---------------------------------------------------------------- |  
    | id          | int    | 收藏的ID                                                         |  
    | item        | string | 收藏对象的ID(若收藏目录类型为题目，则为题目ID，否则为答题记录ID) |  
  
* 说明  
    无 
  
#### 5.1.4. 修改收藏目录  
* URI：/users/:uid/shoucangmulus/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明         |  
    |:------ |:-------|:----:|:------------ |  
    | name   | string | 是   | 收藏目录名称 |  
  
* 返回参数  
    无  
* 说明  
    仅支持修改收藏目录的名称，修改目录的类型会造成其下的收藏id失效，故不支持 
  
#### 5.1.5. 删除收藏目录  
* URI：/users/:uid/shoucangmulus/:id    
* 请求方法：DELETE 
* 请求参数  
    无
* 返回参数  
    无  
* 说明  
    当删除一个收藏目录时，首先将其shanchu字段置为1，并将该目录下所有收藏shanchu字段置为1 

### 5.2 收藏  
  
#### 5.2.1. 添加收藏  
* URI：/users/:uid/shoucangmulus/:mlid/shoucangs  
* 请求方法：POST  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明                                                             |  
    |:------ |:-------|:----:|:---------------------------------------------------------------- |  
    | name   | string |  是  | 收藏的名称，方便用户标识收藏对象                                 |  
    | item   | int    |  是  | 收藏对象的ID(若收藏目录类型为题目，则为题目ID，否则为答题记录ID) |  
  
* 返回参数  
  
    | 名称   | 类型   | 说明           |  
    |:------ |:-------|:-------------- |  
    | id     | int    | 创建的收藏的ID |  
  
* 说明  
    * 同一目录下不能创建相同item字段的收藏  
    * 暂不支持收藏顺序的改变，服务端仅保存其创建顺序（即ID递增）
    * name字段可由客户端按某种规则生成，也可由用户写入
  
#### 5.2.2. 查询收藏列表  
* URI：/users/:uid/shoucangmulus/:mlid/shoucangs  
* 请求方法：GET  
* 请求参数  
    * Paging  
    * Fields，支持的字段包括：id、name、mulu、item、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的字段同上
  
* 返回参数  
  
    | 名称             | 类型   | 说明                          |  
    |:---------------- |:-------|:----------------------------- |  
    | shoucangs[count] | ARRAY  | count个收藏信息组成的数组     |  
    
    其中“shoucangs[count]”数组的每个元素的结构为：  
  
    | 名称        | 类型   | 说明                                                             |  
    |:----------- |:-------|:---------------------------------------------------------------- |  
    | id          | int    | 收藏的ID                                                         |  
    | name        | string | 收藏的名称，方便用户标识收藏对象                                 |  
    | mulu        | int    | 所属收藏目录的ID                                                 | 
    | item        | int    | 收藏对象的ID(若收藏目录类型为题目，则为题目ID，否则为答题记录ID) |  
    | shanchu     | int    | 是否已删除                                                       |
    | xiugaitime  | date   | 最后修改时间                                                     |     
  
* 说明  
    无  
  
#### 5.2.3. 查询收藏  
* URI：/users/:uid/shoucangmulus/:mlid/shoucangs/:id  
* 请求方法：GET  
* 请求参数  
    * Fields，支持的字段包括：id、name、mulu、item、shanchu、xiugaitime，默认包含所有字段  
  
* 返回参数  
  
    | 名称        | 类型   | 说明                                                             |  
    |:----------- |:-------|:---------------------------------------------------------------- |  
    | id          | int    | 收藏的ID                                                         |  
    | name        | string | 收藏的名称，方便用户标识收藏对象                                 |  
    | mulu        | int    | 所属收藏目录的ID                                                 | 
    | item        | int    | 收藏对象的ID(若收藏目录类型为题目，则为题目ID，否则为答题记录ID) |  
    | shanchu     | int    | 是否已删除                                                       |
    | xiugaitime  | date   | 最后修改时间                                                     |     
  
* 说明  
    无 
  
#### 5.2.4. 修改收藏  
* URI：/users/:uid/shoucangmulus/:mlid/shoucangs/:id  
* 请求方法：PUT  
* 请求参数  
  
    | 名称   | 类型   | 必选 | 说明             |  
    |:------ |:-------|:----:|:---------------- |  
    | name   | string | 否   | 收藏名称         |  
    | mulu   | int    | 否   | 所属收藏目录的ID | 
  
* 返回参数  
    无  
* 说明  
     收藏创建后item不能修改，但是可以修改mulu字段，表示移动收藏到对应目录下
  
#### 5.2.5. 删除收藏  
* URI：/users/:uid/shoucangmulus/:mlid/shoucangs/:id  
* 请求方法：DELETE 
* 请求参数  
    无
* 返回参数  
    无  
* 说明  
    无  
 
## 6. 设备信息

### 6.1 添加设备信息
* URI：/shebeis 
* 请求方法：POST
* 请求参数

    | 名称       | 类型       | 必选  | 说明                         |  
    |:---------- |:-----------|:----- |:---------------------------- |
    | mac        |  BIGINT    |  是   | 设备的mac地址                | 
    | info       |  TEXT      |  否   | 当前操作的应用程序的进程标识 | 
    | caozuotime |  BIGINT    |  是   | 当前的操作时间               | 
    
* 返回参数  
    | 名称   | 类型   | 说明                             |  
    |:------ |:-------|:-------------------------------- |  
    | id     | int    | 返回插入设备总表中的记录的ID     |   
* 说明  
   * 每次“心跳”都会发送该请求，该请求会查询设备总表寻找是否有该mac地址的设备，如果没有则插入纪录，如果有则不插但是会修改xiugaitime；
   * 该请求还会将mac、info、caozuotime插入设备信息表中；
   * 客户端可以从设备信息表中查找某一台设备（mac地址唯一标识）所运行过的应用程序，以及运行该应用程序的时间（caozuotime）；


### 6.2 查询设备信息
* URI：/shebeis 
* 请求方法：GET
* 请求参数
    * Paging  
    * Fields，支持的字段包括：id、mac、shanchu、xiugaitime，默认只包含id  
    * Filter，支持的字段同上

* 返回参数  

    | 名称       | 类型   | 说明               |  
    |:---------- |:-------|:------------------ |  
    | id         | int    | 返回设备总表中的ID |
    | mac        | int    | 设备的mac地址      |
    | shanchu    | int    | 是否已删除         |
    | xiugaitime | DATE   | 最后修改时间       |    


#### 6.3 查询设备操作信息  
* URI：/shebeis/:id  
* 请求方法：GET  
* 请求参数
    * Fields，支持的字段包括：id、caozuoxinxi、shanchu、xiugaitime，默认只包含所有字段
    
* 返回参数  

    | 名称               | 类型   | 说明                           |  
    |:------------------ |:-------|:------------------------------ |
    | id                 | int    | 设备总表中的ID                 |
    | caozuoxinxi[count] | ARRAY  | count个设备操作信息组成的数组  |     
    | xiugaitime         | DATE   | 设备总表中该设备的最后修改时间 |      
  
    其中“caozuoxinxi[count]”数组的每个元素的结构为：

    | 名称       | 类型   | 说明                             |  
    |:---------- |:------ |:-------------------------------- |   
    | mac        | int    | 设备的mac地址                    |  
    | info       | string | 设备的操作信息                   |  
    | caozuotime | DATE   | 设备操作信息表中该设备的操作时间 |        
  
* 说明  
    无
  
### 6.4 删除设备操作信息  
* URI：/shebeis/:id  
* 请求方法：DELETE
* 请求参数
    无
* 返回参数  
    无  
* 说明  
   * 该操作会先从设备总表中选出改id下的设备的mac地址，然后从设备操作信息表中将具有该mac地址的纪录全部删除；


## 7. 自动更新

### 7.1. 添加新的版本
* URL:/gengxin
* 请求方法：POST
* 请求参数：
    
    | 名称        | 类型     | 必选 | 说明               |
    |:------------|:---------|:----:|:------------------ |
    | clienttype  | int     |  是  | 0:学生端，1:老师端 |
    | versioncode | int     |  是  | 是否兼容版本号     |
    | apkcode     | int     |  是  | APK版本号          |
    | sourcecode  | int     |  是  | 资源文件版本号     |
    | apkurl      | string  |  是  | apk的下载地址      |
    | sourceurl   | string  |  是  | source的下载地址   |
    
* 返回参数

    | 名称 | 类型  | 说明               |
    |:---- |:----- |:------------------ |
    | id   | int   | 版本更新的ID       |

* 说明：
    * 这个接口是管理员端调用的，管理员把新的apk文件和打包文件传到服务器上并调用这个接口将版本号（versioncode、apkcode、sourcecode）和文件url地址传入数据库；    
    
### 7.2. 获取更新信息
* URL: /gengxin
* 请求方法：GET
* 请求参数

    | 名称            | 类型  | 必选 | 说明             |
    |:----------------|:------|:----:|:---------------- |
    | clienttype      |  int  |  是  | 0学生端 1老师端  |
    | versioncode     |  int  |  是  | 是否兼容版本号   |
    | apkcode         |  int  |  是  | APK版本号        |
    | sourcecode      |  int  |  是  | 资源文件版本号   |
    
* 返回参数

    | 名称             | 类型     | 说明               |
    |:----------------|:---------|:-------------------|
    |   updatetype    |  int     |  0 不更新  1 APK更新  2 资源更新 |
    |   url           |  string  |  更新文件的下载地址      |
    
* 说明：    
    * www.zip文件中包括
    * www文件夹
    * version.json的文件
    * version.json的文件中包括版本号例如：{versioncode：0，apkcode：0，sourcecode}分别代表兼容版本号, apk版本号, 资源版本号
    * www文件夹和version.json压缩的时候在同一级
* 客户端通过返回的update_type和url从服务器下载相应的更新安装包；

### 7.3. 查看版本信息
* URL:/gengxin/:clienttype
* 请求方法：GET
* 请求参数  
    * Fields，支持的字段包括：id、clienttype、versioncode、apkcode、sourcecode，默认只包含id  
    * Filter，支持的过滤字段包括：id、versioncode、apkcode、sourcecode

* 返回参数
    
    | 名称            | 类型     | 说明                             |
    |:----------------|:---------|:-------------------------------- |
    |   id            |  int     | 版本信息的ID                     |
    |   clienttype    |  int     | 客户端的类型（0学生端 1老师端）  |
    |   versioncode   |  int     | 代表是否兼容版本号               |
    |   apkcode       |  int     | 代表APK版本号                    |
    |   sourcecode    |  int     | 代表资源文件版本号               |
    
    
## 8. 杂项  
  
### 8.1. 服务器信息  
  
#### 8.1.1. 查询服务器信息  
* URI: /serverinfos  
* 请求方法: GET  
* 请求参数  
    无  
* 返回参数  
  
    | 名称             | 类型   | 说明                       |  
    |:---------------- |:-------|:-------------------------- |  
    | addresses[count] | ARRAY  | 由服务器监听地址组成的数组 |  
  
    其中“addresses[count]”数组的每个元素的结构为：  
  
    | 名称    | 类型   | 说明                                      |  
    |:------- |:-------|:----------------------------------------- |  
    | address | string | 服务器监听的IPv4地址，例如：192.168.0.100 |  
    | port    | int    | 服务器监听的TCP协议端口号，例如：8080     |  
  
* 说明  
    客户端使用此接口来发现服务端的局域网地址信息，从而优先通过局域网连接。建议客户端逻辑为： 
    * 客户端应该缓存服务端的局域网地址（初始时缓存为空）  
    * 客户端在每次登陆时判断应该连接局域网地址还是公网地址  
    * 在登陆时，客户端应该首先连接缓存的服务端局域网地址，如果连接成功，则当前登录的后续访问都通过局域网地址进行（注意局域网地址可能有多个，需要全部进行尝试）  
    * 如果缓存为空或者连接超时（建议超时值为2s），则通过服务端的域名来连接（使用域名连接时，端口为80），并登录获取token  
    * 通过域名请求 GET /serverinfos ，返回的结果中包含服务器的局域网地址和端口数组，客户端将其保存到本地缓存（刷新缓存）  
    * 客户端通过刷新的服务器局域网地址连接服务器。如果成功，则当前登录的后续请求全部通过局域网地址进行；否则，后续请求全部通过域名进行。  
  
  
  
## 9. 接口使用示例  

### 9.1. 登录和授权
下面是客户端登录获取授权Token，并使用Token来访问其他接口的例子：  
（大于号‘>’后面的内容是客户端发给服务器的HTTP报头，小于号‘<’后面的内容是服务器回应的HTTP报头）  
* 登录获取Token（使用HTTP摘要认证，这里用户admin的密码为admin）  
        > POST /logins HTTP/1.1  
        > User-Agent: curl/7.40.0  
        > Host: 192.168.1.61:3000  
        > Accept: */*  
        >   
        < HTTP/1.1 401 Unauthorized  
        < X-Powered-By: Express  
        < Access-Control-Allow-Origin: *  
        < WWW-Authenticate: Digest realm="sseservice", nonce="8yDNoEqNDPEFHPgmMPDsk2sKqzddhIiC", qop="auth"  
        < Date: Thu, 21 Jan 2016 07:30:31 GMT  
        < Connection: keep-alive  
        < Transfer-Encoding: chunked  
        <   
        > POST /logins HTTP/1.1  
        > Authorization: Digest username="admin", realm="sseservice", nonce="8yDNoEqNDPEFHPgmMPDsk2sKqzddhIiC", uri="/logins", cnonce="YTY3ZTcwNmY2NjFjNzQzNWE0ZDMzMGVjYTQyZDdlNzI=", nc=00000001, qop=auth, response="f5dfd2438705b7b67e518ab66633b1ed"  
        > User-Agent: curl/7.40.0  
        > Host: 192.168.1.61:3000  
        > Accept: */*  
        >   
        < HTTP/1.1 201 Created  
        < X-Powered-By: Express  
        < Access-Control-Allow-Origin: *  
        < Content-Type: application/json; charset=utf-8  
        < Content-Length: 204  
        < ETag: W/"cc-DyZN90fwHLjRb8hHbiIovQ"  
        < Date: Thu, 21 Jan 2016 07:30:31 GMT  
        < Connection: keep-alive  
        <   
        {  
          "expire": 1453447831,  
          "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpcCI6IjE5Mi4xNjguMS42MSIsImlhdCI6MTQ1MzM2MTQzMSwiZXhwIjoxNDUzNDQ3ODMxLCJzdWIiOjF9.iJNMWyz26zJ8HYkOOxrG5BDhzxXSBX7AmJBTQn-Jao4",  
          "id": 1  
        }  
* 使用Token来访问接口（使用HTTP报头`Authorization: Bearer <Token>`）  
        > GET /logins HTTP/1.1  
        > User-Agent: curl/7.40.0  
        > Host: 192.168.1.61:3000  
        > Accept: */*  
        > Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpcCI6IjE5Mi4xNjguMS42MSIsImlhdCI6MTQ1MzM2MTQzMSwiZXhwIjoxNDUzNDQ3ODMxLCJzdWIiOjF9.iJNMWyz26zJ8HYkOOxrG5BDhzxXSBX7AmJBTQn-Jao4  
        >   
        < HTTP/1.1 200 OK  
        < X-Powered-By: Express  
        < Access-Control-Allow-Origin: *  
        < Content-Type: application/json; charset=utf-8  
        < Content-Length: 295  
        < ETag: W/"127-smxrFiZcNOfa3fu539vLtw"  
        < Date: Thu, 21 Jan 2016 07:41:08 GMT  
        < Connection: keep-alive  
        <   
        [  
            {  
                "last": "2016-01-21T07:30:31.698Z",  
                "expire": 1453447831,  
                "login": "2016-01-21T07:30:31.698Z",  
                "ip": "192.168.1.61",  
                "id": 1,  
                "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpcCI6IjE5Mi4xNjguMS42MSIsImlhdCI6MTQ1MzM2MTQzMSwiZXhwIjoxNDUzNDQ3ODMxLCJzdWIiOjF9.iJNMWyz26zJ8HYkOOxrG5BDhzxXSBX7AmJBTQn-Jao4"  
            },  
            {  
                "last": "2016-01-21T07:47:44.938Z",  
                "expire": 1453448864,  
                "login": "2016-01-21T07:47:44.938Z",  
                "ip": "192.168.1.64",  
                "id": 1,  
                "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpcCI6IjE5Mi4xNjguMS42NCIsImlhdCI6MTQ1MzM2MjQ2NCwiZXhwIjoxNDUzNDQ4ODY0LCJzdWIiOjF9.1tFvHJoO9mPEZWbw4tMKCQk1DbKn2hz3yjKOkl-ALOA"  
            }  
        ] 