# 关于这个项目
这是一个帮助你学习Lambda custom runtime的小项目。

内容包括了：
- [lambda custom runtime工作原理的解释](#什么是lambda-custom-runtime)
- [3个动手小实验](#小实验)

# 什么是lambda custom runtime
## 背景
lambda原来只能支持有限的语言种类，包括node.js, python, .Net, Go, java, ruby等。 

如果需要在Lambda上运行不支持的语言或者二进制文件该怎么办呢？原先有一种比较有趣的方案，其实就是用已经支持的语言来写一个代理，包装在不被支持的语言的二进制运行文件之上。（参考 [在lambda上运行其他语言](https://github.com/lazydragon/asap/tree/master/other_language)）

而lambda custom runtime就是正统的这个问题的解决方案。

AWS新出的官方对于c++和rust的支持其实都是基于custom runtime来实现的，底层都使用了runtime API技术。
- [rust runtime](https://github.com/awslabs/aws-lambda-rust-runtime)
- [cpp runtime](https://github.com/awslabs/aws-lambda-cpp)

## 什么是runtime API
runtime API 是aws lambda所提供的http API, 帮助custom runtime监听lambda的触发事件，和返回处理结果。

runtime API一共有4个API接口：
### 触发事件监听
HTTP请求类型： GET

HTTP请求路径： /runtime/invocation/next

```bash
  curl "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/next"
```
### 返回正常处理结果
HTTP请求类型： POST

HTTP请求路径： /runtime/invocation/AwsRequestId/response

```bash
REQUEST_ID=156cb537-e2d4-11e8-9b34-d36013741fb9
curl -X POST  "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/response"  -d "SUCCESS"
```
### 返回处理异常
HTTP请求类型： POST

HTTP请求路径： /runtime/invocation/AwsRequestId/error

```bash
REQUEST_ID=156cb537-e2d4-11e8-9b34-d36013741fb9
ERROR="{\"errorMessage\" : \"Error parsing event data.\", \"errorType\" : \"InvalidEventDataException\"}"
curl -X POST "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/error"  -d "$ERROR"
```
### 返回初始化错误
HTTP请求类型： POST

HTTP请求路径： /runtime/init/error

```bash
REQUEST_ID=156cb537-e2d4-11e8-9b34-d36013741fb9
ERROR="{\"errorMessage\" : \"Failed to load function.\", \"errorType\" : \"InvalidFunctionException\"}"
curl -X POST "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/init/error"  -d "$ERROR"
```

## runtime API的使用
runtime API的使用流程一般是:
1. 循环监听[触发事件监听API](#触发事件监听)
2. 对每次事件，使用相对应自定义代码处理
3. 根据处理的成功和失败，使用相对应的返回API返回结果

接下来的小实验会帮助大家动手理解runtime API的使用方式，大家也可以之后参考rust runtime的[实现方式](https://github.com/awslabs/aws-lambda-rust-runtime/blob/master/lambda-runtime-client/src/client.rs)


# 小实验
1. [用custom runtime跑bash脚本](#用custom-runtime跑bash脚本)
2. [用layer分离runtime和lambda方法](#用layer分离runtime和lambda方法)
3. [用custom runtime跑php脚本](#用custom-runtime跑php脚本)

## 用custom runtime跑bash脚本
## 用layer分离runtime和lambda方法
## 用custom runtime跑php脚本

# 参考文献
- runtime api: https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/runtimes-api.html
