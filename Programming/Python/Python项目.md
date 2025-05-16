## 类型提示

Pydantic 的 BaseModel，是比 dataclass 更强大的功能，BaseModel里如果定义了字段为 int，那么即使使用的时候为 string，也会转成 int。所以如果项目能用 BaseModel的话，最好用这个。除非是不太方便安装第三方库，就只能用自带的 dataclass了。

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