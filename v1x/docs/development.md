# 开发扩展

HttpRunner 除了作为命令行工具使用外，还可以作为软件包集成到你自己的项目中。

简单来说，HttpRunner 提供了运行 YAML/JSON 格式测试用例的能力，并能返回详细的测试结果信息。

## HttpRunner class

HttpRunner 以 `类（class）` 的形式对外提供调用支持，类名为`HttpRunner`，导入方式如下：

```python
from httprunner import HttpRunner
```

### 可用初始化参数

`HttpRunner` 内部用于驱动测试执行的是`unittest.TextTestRunner`，在初始化 `HttpRunner` 时可以使用 TextTestRunner 的所有可用参数（详情可阅读[官方文档][TextTestRunner]）。除此之外，`HttpRunner`还额外支持一个参数，`http_client_session`，可用于指定不同的客户端类型。

通常情况下，初始化 `HttpRunner` 时常用的参数有如下几个：

- `resultclass`: HtmlTestResult/TextTestResult，默认值为 HtmlTestResult
- `failfast`: 设置为 True 时，测试在首次遇到错误或失败时会停止运行；默认值为 False
- `http_client_session`: 传入`requests.Session()`时进行自动化测试（默认），传入`locust.client.Session()`时进行性能测试

例如，如需初始化 `HttpRunner` 时设置 `failfast` 为 False，初始化方式如下所示：

```python
from httprunner import HttpRunner

runner = HttpRunner(failfast=False)
```

### 可用调用方法

在 `HttpRunner` 中，对外提供了两个方法：

- `run`: 运行测试用例
- `gen_html_report`: 生成 HTML 测试报告

### 可用属性

在 `HttpRunner` 中，对外提供了一个属性：

- `summary`: 测试执行结果

该属性需要在调用 `run` 方法后获取。

## 运行测试用例

`HttpRunner` 的 run 方法有三个参数：

- `path_or_testcases`: 指定要运行的测试用例；支持传入两类参数，YAML/JSON 格式测试用例文件路径，或者标准的测试用例结构体；
- `dot_env_path`（可选）: 指定加载环境变量文件（.env）的路径，默认值为当前工作目录（PWD）中的 `.env` 文件
- `mapping`（可选）: 变量映射，可用于对传入测试用例中的变量进行覆盖替换。

### 传入测试用例文件路径

指定测试用例文件路径支持三种形式：

- YAML/JSON 文件路径，支持绝对路径和相对路径
- 包含 YAML/JSON 文件的文件夹，支持绝对路径和相对路径
- 文件路径和文件夹路径的混合情况（list/set）

```python
# 文件路径
runner.run("docs/data/demo-quickstart-2.yml")

# 文件夹路径
runner.run("docs/data/")

# 混合情况
runner.run(["docs/data/", "files/demo-quickstart-2.yml"])
```

如需指定加载环境变量文件（.env）的路径，或者需要对测试用例中的变量进行覆盖替换，则可使用 `dot_env_path` 和 `mapping` 参数。

```python
# dot_env_path
runner.run("docs/data/demo-quickstart-2.yml", dot_env_path="/path/to/.env")

# mapping
override_mapping = {
    "device_sn": "XXX"
}
runner.run("docs/data/demo-quickstart-2.yml", mapping=override_mapping)
```

### 传入标准的测试用例结构体

除了传入测试用例文件路径，还可以直接传入标准的测试用例结构体。

以 [demo-quickstart-2.yml](/data/demo-quickstart-2.yml) 为例，对应的数据结构体如下所示：

```json
[
  {
    "config": {
      "name": "testcase description",
      "request": {
        "base_url": "",
        "headers": {
          "User-Agent": "python-requests/2.18.4"
        }
      },
      "variables": [],
      "output": ["token"],
      "path": "/abs-path/to/demo-quickstart-2.yml",
      "refs": {
        "env": {},
        "debugtalk": {
          "variables": {
            "SECRET_KEY": "DebugTalk"
          },
          "functions": {
            "gen_random_string": <function gen_random_string at 0x108596268>,
            "get_sign": <function get_sign at 0x1085962f0>,
            "get_user_id": <function get_user_id at 0x108596378>,
            "get_account": <function get_account at 0x108596400>,
            "get_os_platform": <function get_os_platform at 0x108596488>
          }
        },
        "def-api": {},
        "def-testcase": {}
      }
    },
    "teststeps": [
      {
        "name": "/api/get-token",
        "request": {
          "url": "http://127.0.0.1:5000/api/get-token",
          "method": "POST",
          "headers": {"Content-Type": "application/json", "app_version": "2.8.6", "device_sn": "FwgRiO7CNA50DSU", "os_platform": "ios", "user_agent": "iOS/10.3"},
          "json": {"sign": "958a05393efef0ac7c0fb80a7eac45e24fd40c27"}
        },
        "extract": [
          {"token": "content.token"}
        ],
        "validate": [
          {"eq": ["status_code", 200]},
          {"eq": ["headers.Content-Type", "application/json"]},
          {"eq": ["content.success", true]}
        ]
      },
      {
        "name": "/api/users/1000",
        "request": {"url": "http://127.0.0.1:5000/api/users/1000", "method": "POST", "headers": {"Content-Type": "application/json", "device_sn": "FwgRiO7CNA50DSU", "token": "$token"},
        "json": {"name": "user1", "password": "123456"}},
        "validate": [
          {"eq": ["status_code", 201]},
          {"eq": ["headers.Content-Type", "application/json"]},
          {"eq": ["content.success", true]},
          {"eq": ["content.msg", "user created successfully."]}
        ]
      }
    ]
  },
  {...} # another testcase
]
```

传入测试用例结构体时，支持传入单个结构体（dict），或者多个结构体（list of dict）。

```python
# 运行单个结构体
runner.run(testcase)

# 运行多个结构体
runner.run([testcase1, testcase2])
```

### 加载 `debugtalk.py` && `.env`

通过传入测试用例文件路径运行测试用例时，HttpRunner 会自动以指定测试用例文件路径为起点，向上搜索 `debugtalk.py` 文件，并将 `debugtalk.py` 文件所在的文件目录作为当前工作目录（PWD）。

同时，HttpRunner 会在当前工作目录（PWD）下搜索 `.env` 文件，以及 `api` 和 `testcases` 文件夹，并自动进行加载。

最终加载得到的存储结构如下所示：

```json
{
  "env": {},
  "debugtalk": {
    "variables": {
      "SECRET_KEY": "DebugTalk"
    },
    "functions": {
      "gen_random_string": <function gen_random_string at 0x108596268>,
      "get_sign": <function get_sign at 0x1085962f0>,
      "get_user_id": <function get_user_id at 0x108596378>,
      "get_account": <function get_account at 0x108596400>,
      "get_os_platform": <function get_os_platform at 0x108596488>
    }
  },
  "def-api": {},
  "def-testcase": {}
}
```

其中，`env` 对应的是 `.env` 文件中的环境变量，`debugtalk` 对应的是 `debugtalk.py` 文件中定义的变量和函数，`def-api` 对应的是 `api` 文件夹下定义的接口描述，`def-testcase` 对应的是 `testcases` 文件夹下定义的测试用例。

通过传入标准的测试用例结构体执行测试时，传入的数据应包含所有信息，包括 `debugtalk.py`、`.env`、依赖的 api 和 测试用例等；因此也无需再使用 `dot_env_path` 和 `mapping` 参数，所有信息都要通过 `refs` 传入。

## 返回详细测试结果数据

运行完成后，通过 `summary` 属性可获取详尽的运行结果数据。

```python
# get result summary
summary = runner.summary
```

其数据结构为：

```json
{
  "success": true,
  "stat": {
    "testsRun": 2,
    "failures": 0,
    "errors": 0,
    "skipped": 0,
    "expectedFailures": 0,
    "unexpectedSuccesses": 0,
    "successes": 2
  },
  "time": {
    "start_at": 1538449655.944404,
    "duration": 0.03181314468383789
  },
  "platform": {
    "httprunner_version": "1.5.14",
    "python_version": "CPython 3.6.5+",
    "platform": "Darwin-17.6.0-x86_64-i386-64bit"
  },
  "details": [
    {
      "success": true,
      "name": "testcase description",
      "base_url": "",
      "stat": {"testsRun": 2, "failures": 0, "errors": 0, "skipped": 0, "expectedFailures": 0, "unexpectedSuccesses": 0, "successes": 2},
      "time": {"start_at": 1538449655.944404, "duration": 0.03181314468383789},
      "records": [
        {
          "name": "/api/get-token",
          "status": "success",
          "attachment": "",
          "meta_data": {
            "request": {
              "url": "http://127.0.0.1:5000/api/get-token",
              "method": "POST",
              "headers": {"User-Agent": "python-requests/2.18.4", "Accept-Encoding": "gzip, deflate", "Accept": "*/*", "Connection": "keep-alive", "Content-Type": "application/json", "app_version": "2.8.6", "device_sn": "FwgRiO7CNA50DSU", "os_platform": "ios", "user_agent": "iOS/10.3", "Content-Length": "52"},
              "start_timestamp": 1538449655.944801,
              "json": {"sign": "958a05393efef0ac7c0fb80a7eac45e24fd40c27"},
              "body": b'{"sign": "958a05393efef0ac7c0fb80a7eac45e24fd40c27"}'
            },
            "response": {
              "status_code": 200,
              "headers": {"Content-Type": "application/json", "Content-Length": "46", "Server": "Werkzeug/0.14.1 Python/3.6.5+", "Date": "Tue, 02 Oct 2018 03:07:35 GMT"},
              "content_size": 46,
              "response_time_ms": 12.87,
              "elapsed_ms": 6.955,
              "encoding": null,
              "content": b'{"success": true, "token": "CcQ7dBjZZbjIXRkG"}',
              "content_type": "application/json",
              "ok": true,
              "url": "http://127.0.0.1:5000/api/get-token",
              "reason": "OK",
              "cookies": {},
              "text": '{"success": true, "token": "CcQ7dBjZZbjIXRkG"}',
              "json": {"success": true, "token": "CcQ7dBjZZbjIXRkG"}
            },
            "validators": [
              {"check": "status_code", "expect": 200, "comparator": "eq", "check_value": 200, "check_result": "pass"},
              {"check": "headers.Content-Type", "expect": "application/json", "comparator": "eq", "check_value": "application/json", "check_result": "pass"},
              {"check": "content.success", "expect": true, "comparator": "eq", "check_value": true, "check_result": "pass"}
            ]
          }
        },
        {
          "name": "/api/users/1000",
          "status": "success",
          "attachment": "",
          "meta_data": {
            "request": {
              "url": "http://127.0.0.1:5000/api/users/1000",
              "method": "POST",
              "headers": {"User-Agent": "python-requests/2.18.4", "Accept-Encoding": "gzip, deflate", "Accept": "*/*", "Connection": "keep-alive", "Content-Type": "application/json", "device_sn": "FwgRiO7CNA50DSU", "token": "CcQ7dBjZZbjIXRkG", "Content-Length": "39"},
              "start_timestamp": 1538449655.958944,
              "json": {"name": "user1", "password": "123456"},
              "body": b'{"name": "user1", "password": "123456"}'
            },
            "response": {
              "status_code": 201,
              "headers": {"Content-Type": "application/json", "Content-Length": "54", "Server": "Werkzeug/0.14.1 Python/3.6.5+", "Date": "Tue, 02 Oct 2018 03:07:35 GMT"},
              "content_size": 54,
              "response_time_ms": 3.34,
              "elapsed_ms": 2.16,
              "encoding": null,
              "content": b'{"success": true, "msg": "user created successfully."}',
              "content_type": "application/json",
              "ok": true,
              "url": "http://127.0.0.1:5000/api/users/1000",
              "reason": "CREATED",
              "cookies": {},
              "text": '{"success": true, "msg": "user created successfully."}',
              "json": {"success": true, "msg": "user created successfully."}
            },
            "validators": [
              {"check": "status_code", "expect": 201, "comparator": "eq", "check_value": 201, "check_result": "pass"},
              {"check": "headers.Content-Type", "expect": "application/json", "comparator": "eq", "check_value": "application/json", "check_result": "pass"},
              {"check": "content.success", "expect": true, "comparator": "eq", "check_value": true, "check_result": "pass"},
              {"check": "content.msg", "expect": "user created successfully.", "comparator": "eq", "check_value": "user created successfully.", "check_result": "pass"}
            ]
          }
        }
      ],
      "in_out": {
        "in": {"SECRET_KEY": "DebugTalk"},
        "out": {"token": "CcQ7dBjZZbjIXRkG"}
      }
    }
  ]
}
```

## 生成 HTML 测试报告

如需生成 HTML 测试报告，可调用 `gen_html_report` 方法。

```python
# generate html report
runner.gen_html_report(
    html_report_name="demo",
    html_report_template="/path/to/custom_report_template"
)

# => reports/demo/demo-1532078874.html
```

关于测试报告的详细内容，请查看[测试报告](/basic/report/)部分。

[TextTestRunner]: https://docs.python.org/3.6/library/unittest.html#unittest.TextTestRunner