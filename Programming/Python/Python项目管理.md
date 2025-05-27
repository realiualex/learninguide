## 项目环境管理
现在推荐使用 uv 来管理项目环境，虽然也可以使用 virtualenv，venv，pyenv，但是uv是新一代项目环境管理软件，越来越普及

```
# 创建虚拟环境，虚拟环境目录为 .venv
uv venv
# 默认上面的命令创建出来的虚拟环境名字就是当前文件夹名字，如果名字太长，可以用下面的方法创建，则名字为.venv
uv venv .venv

# 激活虚拟环境
source .venv/bin/activate   # macOS / Linux
.\.venv\Scripts\activate    # Windows

# 添加软件
uv add requests
uv pip list
uv pip compile pyproject.toml > requirements.txt

```
## 类型提示

Pydantic 的 BaseModel，是比 dataclass 更强大的功能，BaseModel里如果定义了字段为 int，那么即使使用的时候为 string，也会转成 int。所以如果项目能用 BaseModel的话，最好用这个。除非是不太方便安装第三方库，就只能用自带的 dataclass了。
* 示例 （Pydantic V2，注意如果V1，语法和下面略有不同）
	* Field 是和 populate_by_name 一起配合使用的，主要用于将 type_ 和别名 type 可以一起使用，当我们在实例化 DNSModel 这个对象的时候，可以传 type，也可以传 type_
	* @property，这个是将方法给变成属性，比如 dns_model = DNSModel()，dns_model.name 可以获得name，使用 @property 属性之后，dns_model.type_name 就能获得type的名字。这个 type_name里做了一个映射，当 type_为1的时候，type_name就为 A，当type_没有定义，比如为99的时候， type_name就是 "99"
	* computed_field：在我们实例化 DNSModel 之后，在使用 .dict() 或 model_dump() 方法的时候，默认情况下，是没有 type_name 的，但是加了 computed_filed，就会有了
	* 第二个例子里，主要展示 RootModel如何定义和使用

```python
from pydantic import BaseModel, ConfigDict, RootModel, computed_field, Field


class DNSModel(BaseModel):
	name: str
	type_: int = Field(..., alias="type")
	TTL: int
	data: str
	
	@computed_field
	@property
	def type_name(self) -> str:
		DNS_TYPE_MAP = {
			1: "A",
            2: "NS",
            5: "CNAME",
            6: "SOA"
		}
		return DNS_TYPE_MAP.get(self.type_, str(self.type_))

	model_config = ConfigDict(extra='Allow', populate_by_name=True)

class SubdomainList(RootModel[List[str]]):
	pass
```

上述示例里，Field

## gunicorn 和 uvicorn
如果依赖python web server自带的listen功能（如 flask 或 fastapi 的 app.run），那么是单线程的，性能和稳定性都不足，Gunicorn 是专门为生产环境设计的高性能 WSGI 服务器(同步)，提供多进程、超时、日志、信号管理等功能

Uvicorn 是 ASGI (异步 + 同步) web 服务器，也会结合Gunicorn一起使用。将Gunicorn做主控（master + 多进程管理），每一个worker实际上都跑了一个 uvicorn 实例（异步支持）。

```
# 只使用 gunicorn
gunicorn myapp:app --workers 4 --bind 0.0.0.0:8000

# 只使用 uvicorn
uvicorn src.main:app --host=0.0.0.0 --port=8080 --reload --access-log

# 将 gunicorn 和 uvicorn 结合使用
gunicorn -w 2 -k uvicorn.workers.UvicornWorker src.main:app --bind 0.0.0.0:8080 --timeout 120

```
