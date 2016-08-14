#频率限制服务的设计与实现

##一. 背景
频率限制服务是可用于累计限制用户操作频率，防止用户恶意刷数据的方式之一，可以限制某一活动的总量，也可以按用户id限制在指定时间周期或者范围内的操作次数。   
部门很多地方都需要进行频率限制，比如限量活动/产品的秒杀活动等，因此需要将频率限制这个功能模块化，服务化，实现为公用的服务。为了讨论方便，在下文中，limit_id表示某一个活动的id，limit_key表示某一个用户的id


##二. 概述
频率限制服务分为两个子服务：  
1. **配置限制管理服务**：  
提供了添加/删除/查询/重置4个接口，管理限制策略；
```c
int AddLimitConfig(const LimitConfig& limit_config, SrfCurrentPtr curr);
int DelLimitConfig(const string& limit_id, SrfCurrentPtr curr);
int GetLimitConfig(const string& limit_id, LimitConfig& rsp, SrfCurrentPtr curr);
int SetLimitConfig(const LimitConfig& limit_config, SrfCurrentPtr curr);
```  
2.**频率限制服务**：  
查询接口，支持查询某个limit_d某个limit_key是否已经限制  
更新接口，对某个活动某个key进行参加操作
```c
int QueryLimit(const string& limit_id, const string& limit_key,  bool& is_limit, SrfCurrentPtr curr);
int UpdateLimit(const string& limit_id, const string& limit_key, const unsigned int count, bool& is_limit, SrfCurrentPtr curr);
```

目前服务支持三类限制策略：  
1. **总量限制**：针对limit_id来说，就是每个活动最多能被参加多少次；针对limit_key来说，就是每个用户最多能参加该活动多少次；
2. **自然时间限制，包括自然分钟(minute)、自然小时（hour）、自然天(day）、自然周(week)、自然月(month)**：可以理解为在某一个自然时间内的总量限制   
3. **时间间隔限制**：可以理解为在某一段时间间隔内的总量限制


##三. 主要数据结构
limit_id的限制策略保存在LimitConfig中，其中LimitItem是某个具体限制策略配置：
```c
struct LimitConfig
{
    0 require string            limit_id;
    ... ... .... ... 
    4 require vector<LimitItem>       items;//这个limit_id包含了哪些策略    
};

struct LimitItem
{
    0 require LIMIT_POLICY       policy; //限制策略（枚举变量）
    1 require long           limit_count; //限制的次数
    ... ... 
};
```
每个limit_id和limit_key都要保存自己最新的状态，即已经参加的次数，以及一些额外的信息；这些数据放在CurrentData中：
```c
struct CurrentData
{
    0 require int                    count;//limit_id和limit_key最新的已被访问次数
    1 require vector<PolicyStatus>        policy_status; //每个limit_id和limit_key的每一项限制策略的最新数据
};

struct PolicyStatus
{
    0 require LIMIT_POLICY    policy;
    1 require long            current_count; //这个策略最新的已被访问次数
    2 require long            last_time; //更新（策略已被访问次数）的时间
};
```
##四. 接入架构  

##五. 频率限制的详细算法
主要介绍一下频率限制中逻辑最复杂的接口：更新接口，对某个活动某个key进行参加操作；  
具体逻辑如下图:

##六. 压测工具的实现以及压测数据 
###先安利一波
因为之前没有接触过压测，所以花了些时间了解了下；在此安利一个超简单的单元测试框架：**Catch**;  
网上推荐的一些经典单元测试框架，比如GoogleTest等，配置十分麻烦，需要花很多时间和精力去学习使用；
而Catch简单到什么程度呢？？？你只需引入一个头文件和一个宏！！
```c
#define CATCH_CONFIG_MAIN  // This tells Catch to provide a main() - only do this in one cpp file
#include "catch.hpp"
```
具体的使用教程在：https://github.com/philsquared/Catch/blob/master/docs/tutorial.md
保证半个小时之内学会！！！

下面给出一个教程中的例子，让大家了解一下这个框架：
```c
#define CATCH_CONFIG_MAIN  // This tells Catch to provide a main() - only do this in one cpp file
#include "catch.hpp"

unsigned int Factorial( unsigned int number ) {
    return number <= 1 ? number : Factorial(number-1)*number;
}

TEST_CASE( "Factorials are computed", "[factorial]" ) {
    REQUIRE( Factorial(1) == 1 );
    REQUIRE( Factorial(2) == 2 );
    REQUIRE( Factorial(3) == 6 );
    REQUIRE( Factorial(10) == 3628800 );
}
```
以上就是测试一个斐波那契数列的全过程！！  整个使用教程也就250行English！！非常简单！  

###好，回归主题
我的压测工具就是基于Catch框架实现的，得到的压测数据如下：

从数据可以看出随着并发度的提高，QPS逐渐增加，但是成功率确急剧下降；这是什么原因呢？？因为是并发产生的失败，所以可以推断是对同一资源的互斥访问所造成的；
先去看一下服务失败产生的log文件，可以看到失败的原因都是因为调用setJceStructVer，而返回的失败码是-13104。查一下CKV的错误码，原来是CKV的cas机制！ckv的cas特性可以保证多用户并发时，get、set操作的原子性。

从数据还可以看出，即使不存在并发的时候，QPS值也比较低。因为程序中不涉及消耗CPU的大型计算，也没有IO操作，因此主要的时间是花在调用CKV服务上；每次请求都会涉及多次CKV的get操作和set操作；
##优化方案
因此针对前面的分析，提出了以下的优化方案及服务架构，优化的点主要是下面三个方面：  
1. 服务最终是分布式的，因此，这里采用hash算法，以活动limit_id为key，将同一个活动的请求集中在一台服务器处理；  
2. 将limit_id和limit_key的最新状态数据CurrentData存在本地;为了后续的容灾，需要定期写入CKV；  
3. 配置数据还是存在CKV，但不是每次都要访问CKV获取配置，而是存在本地缓存，定时去更新拉取；

服务架构图如下：

处理请求的流程图如下：
