#频率限制服务的设计与实现

##一. 背景
频率限制服务是可用于累计限制用户操作频率，防止用户恶意刷数据的方式之一，可以限制某一活动的总量，也可以按用户id限制在指定时间周期或者范围内的操作次数。后台服务很多都需要进行频率限制，因此频率限制这个功能模块化，服务化，将它实现为公用的服务。为了讨论方便，在下文中，limit_id表示某一个活动的id，limit_key表示某一个用户的id


##二. 概述
频率限制服务分为两个子服务：  
1. **配置限制管理服务**：  
提供了添加/删除/查询/重置4个接口，管理限制策略；  
2. **频率限制服务**：  
查询接口，支持查询某个limit_d某个limit_key是否已经限制  
更新接口，对某个活动某个key进行参加操作

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
检查用户频率限制的具体逻辑如下图:

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
##优化方案
