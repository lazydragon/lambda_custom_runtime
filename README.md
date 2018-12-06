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
4. 将以上这些逻辑打包成为一个bootstrap可执行文件，上传到lambda 

接下来的小实验会帮助大家动手理解runtime API的使用方式，大家也可以之后参考rust runtime的[实现方式](https://github.com/awslabs/aws-lambda-rust-runtime/blob/master/lambda-runtime-client/src/client.rs)


# 小实验
1. [用custom runtime跑bash脚本](#用custom-runtime跑bash脚本)
2. [用layer分离runtime和lambda方法](#用layer分离runtime和lambda方法)
3. [用custom runtime跑php脚本](#用custom-runtime跑php脚本)

## 用custom runtime跑bash脚本
### 1. 参考官方教程创建bootstrap和function代码

bootstrap
```bash
#!/bin/sh

set -euo pipefail

# Initialization - load function handler
source $LAMBDA_TASK_ROOT/"$(echo $_HANDLER | cut -d. -f1).sh"

# Processing
while true
do
  HEADERS="$(mktemp)"
  # Get an event
  EVENT_DATA=$(curl -sS -LD "$HEADERS" -X GET "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/next")
  REQUEST_ID=$(grep -Fi Lambda-Runtime-Aws-Request-Id "$HEADERS" | tr -d '[:space:]' | cut -d: -f2)

  # Execute the handler function from the script
  RESPONSE=$($(echo "$_HANDLER" | cut -d. -f2) "$EVENT_DATA")

  # Send the response
  curl -X POST "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/response"  -d "$RESPONSE"
done
```

function.sh
```bash
function handler () {
  EVENT_DATA=$1
  echo "$EVENT_DATA" 1>&2;
  RESPONSE="Echoing request: '$EVENT_DATA'"

  echo $RESPONSE
}
```

### 2. 打包bootstrap和function.sh到一个zip文件。
注意： bootstrap和function.sh都需要配置成为可执行文件
大家也可以直接使用我已经打包好的[zip](./bash_example/function.zip)

### 3. 在控制台创建lambda并上传zip文件

创建lambda

![](./images/create1.png)

上载zip包

![](./images/upload1.png)

### 4. 创建测试案例并测试
![](./images/test1.png)

测试

![](./images/result1.png)

## 用layer分离runtime和lambda方法
## 用custom runtime跑php脚本

# 参考文献
- runtime api: https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/runtimes-api.html
- 创建custom runtime官方教程: https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/runtimes-walkthrough.html
