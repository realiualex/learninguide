默认是不打印 logger 或print的日志，可以用下面的方法来打印
```shell
python -m pytest -s --capture=no --log-cli-level=INFO test_all.py
```
## Pytest 命名规范

* 类名必须Test开头
* 类里方法名必须 test_开头或者 _test 结尾
* 文件名必须 test_开头
* 如果想要修改默认的配置，则需要创建一个 pytest.ini 配置文件。无论是主函数模式运行，还是命令行运行，都会读这个配置文件。

## 入门示例

创建一个all.py文件，同时创建一个user文件夹

```python
import pytest  
   
if __name__ == "__main__":
    pytest.main(["-sv","./user", "-n=2"])
```

在user文件夹下，创建一个 test_get_user.py文件

```python
class TestGetUser:
    def test_get_user(self):
        print("test get user function")
```

此时pytest会执行 user 文件夹下的所有测试case，当然如果main 函数里不指定参数也是可以的，则会执行当前文件夹以及遍历其子文件夹下的所有测试案例。

如果使用pytest cli那么命令为 ``` pytest -sv ./user ```

如果想要用多线程的话，可以加上 -n 参数，比如 ``` pytest -sv ./user -n 2```

如果想要让测试案例失败的时候，重跑，那么main函数里可以加 "--reruns=2"，这样当失败的时候，会再重试2次(一共3次)



## 简化版

比如下面的目录结构，mymath.py里只有一个 add(num1, num2) 函数，在pycharm里，将 src 和test所在的目录，设置为 Source Root，

```shell
.
├── src
│   └── mymath.py
└── test
    └── all.py
```

all.py 的内容为

```python
from src.mymath import add

def test_add():
    result = add(3,4)
    assert result == 7
```

那么只需要执行下面的测试命令即可

```shell
python -m pytest test/all.py
```
## coverage
使用coverage可以轻松查看pytest的覆盖率
```python
# run test
python -m pytest test_all.py

# run report by coverage command
coverage run -m pytest test_all.py
coverage report

# get html report
coverage html
```

## pytest 使用

### 改变执行顺序
通过pytest.mark.run 来改变之行顺序，默认情况下，pytest是按函数名从上往下之行的。这个跟unitest不同，unitest是根据函数名ascii顺序执行。
```python
@pytest.mark.run(order=2)
def test_xxx():
	print("xxxx")
```

### 分组执行
如果我们的测试比较复杂，分不同的模块（文件夹），下面分别又有不同的测试样例，我们想测试某些文件夹下特定的某几个文件。比如执行冒烟测试，分接口测试等。

此时可以在需要进行测试的函数上面，通过 @pytest.mark.xxxx 做标记，注意xxx是可以自己随便写的，比如我们写smoke，usermanage, productmanage，即使分布在不同的模块下，不同的函数里，我们也可以在测试的时候，来执行那类标记被执行
```python

@pytest.mark.smoke
def test_aa():
	print("aa")

@pytest.mark.usermanage
def test_bb():
	...

@pytest.mark.productmanage
def test_cc():
	...
```
另外，还需要创建一个 pytest.ini 文件，将这些自定义marker注册进去
```toml
[pytest]  
markers =  
    smoke: anything you can input here
    usermanage: xxxx
    productmanage: xxxx
```

执行的时候输入 pytest -m "smoke" test.py 则只执行 smoke里的样例。如果想一次性执行两个，则是 pytest -m "smoke or usermanage" test.py

### 跳过
如果想要跳过某一个测试用例，则用 pytest.mark.skip 就可以了，这样默认就不执行了 
```python
@pytest.mark.skip
def xxx():
	...
```
在跳过的时候，还可以指定条件

```python
bb=16
@pytest.mark.skipif(bb < 18, reason='小于18')  
def test_bb():  
    print("bb")
```


### pytest.fixture 与setup/teardown
#### setup/teardown 与 setup_class 与 teardown_class
这个方法比较直接，适合简单测试场景
在pytest里面，如果想要在执行测试前，和测试后，做一些事情。比如在测试前，第一步要打开浏览器，测试完成，最后一步关闭浏览器，那么我们就要用到 setup_method和 teardown_method的函数（对每一个测试用例，即每一个函数都执行前后都会执行这个操作），注意函数名必须这么写（小写），如果函数在类里(比如日志对象，数据库连接等)，则是 setup_class 和 teardown_class（在类初始化以及执行完成只做一次）。示例：
```python
import pytest  
  
bb = 20  
  
  
class TestAA:  
  
    def setup_class(self):  
        print("\nsetup class")  
  
    def setup_method(self, method):  
        print("\nsetup class===")  
  
    def test_aa(self):  
        print("aa")  
  
    @pytest.mark.skipif(bb < 18, reason='大于18')  
    def test_bb(self):  
        print("bb")  
  
    def test_cc(self):  
        print("cc")  
  
    def teardown_method(self, method):  
        print("\nteardown method+++")  
  
    def teardown_class(self):  
        print("\nteardown")
```

#### pytest.fixture
pytest.fixture适合更灵活，需要复用参数的复杂测试场景。在fixture里，函数名可以随便起，如果后面的参数想自动引用，则 autouse=True，如果没有，需要将这个函数名作为参数，传给后面真正的方法。
在fixture里，yield相当于会把函数分成两部分，上面是setup，后面是 teardown

```python
class TestMain:

    @pytest.fixture(scope="session", autouse=True)
    def set_env(self):
        os.environ["ES_HOST"] = "mydomain.com"
        os.environ['REGION'] = 'ap-northeast-1'
        yield
        del os.environ["ES_HOST"]
        del os.environ['REGION']

```
不过对于改环境变量，更好的办法是通过 monkeypatch的方法。如果用上面的方法，需要显性的删除 os.environ，并且所有的测试单元里，用的是同一份环境变量，这意味着，如果某一个单元测试改了环境变量的值，会影响另外一个单元测试。如果用下面的 monkeypatch，则所有的单元测试的环境变量是独立的。monkeypatch除了能改环境变量外，还能改某一个函数的返回值，某一个json的key或value等
```python
class TestAll:
    @pytest.fixture(autouse=True)
    def setup_env(self, monkeypatch):
        monkeypatch.setenv("TEST_ENV", "1")

```

#### 测试函数参数
在 pytest 中，测试函数都是以 test_ 开头的，如果测试函数要传入参数，这个参数往往是一个 fixture (可以是 pytest 自带的 fixture，如 tmp_path，也可以是自定义的fixture)。

比如我们需要创建一个测试文件，并向这个文件里写点东西，那么我们可以用 tmp_path 这个python自带的fixture，它会自动创建这个文件，并在测试完成后删除。由于是自带的fixture，所以 test_bucket_ecr_mapping 函数可以直接使用这个作为参数

```python
def test_bucket_ecr_mapping(tmp_path):
    mapping_file = tmp_path / "mapping.json"
    mapping_file.write_text('{"bucket1": "ecr1"}')
    with patch('lambda_function.open', create=True) as mock_open:
        mock_open.return_value.__enter__.return_value.read.return_value = '{"bucket1": "ecr1"}'
        result = bucket_ecr_mapping()
        assert result == {"bucket1": "ecr1"}
```

但如果我们传入的是一个自定义的值，那么就需要把这个变成一个fixture。示例我们把 tmp_path 改成 temp_path，则就需要把 temp_path 变成 fixture.
```python
@pytest.fixture
def temp_path(tmp_path):
    return tmp_path

def test_bucket_ecr_mapping(temp_path):

    mapping_file = tmp_path / "mapping.json"
    mapping_file.write_text('{"bucket1": "ecr1"}')
    with patch('lambda_function.open', create=True) as mock_open:
        mock_open.return_value.__enter__.return_value.read.return_value = '{"bucket1": "ecr1"}'
        result = bucket_ecr_mapping()
        assert result == {"bucket1": "ecr1"}
```

在上面的示例中，注意看 `patch('lambda_function.open', create=True) ` 这一部分，虽然 open() 函数是python builtin的，在 lambda_function里并没有直接定义，且 lambda_function里的子函数才会用到，可是在做 from lambda_function import ... 或者 import_funciton 的时候，整个顶层代码都会被执行，所有的函数和类都会被定义，所以即使子函数里用了 open()，也会生效。create=True这个参数，是因为 patch 的 open 并不是模块的属性，所以要允许 mock 创建一个不存在的属性

 
#### 使用 mock 方法来patch 函数返回值
pytest 可以和 unittest 结合使用，比如当我们在 lambda_function.py 文件里，有一个 get_string_data 的函数，这个函数需要访问外部数据，但我们每次测试的时候，不想让这个接口访问外部数据，那么就可以用 mock的 patch方法，来模拟返回值。在模拟的时候，可以用with上下文管理器结合yield方法，也可以直接用装饰器。两种用法都是可以的
```python
import pytest
from unittest.mock import patch

    @pytest.fixture(autouse=True)
    def test_get_string_data(self):
        with patch('lambda_function.get_string_data') as mock_get_string_data:
            mock_get_string_data.return_value = "this is mock return"
            yield mock_get_string_data
            
    @patch("lambda_function.my_list", new=["a","b"])
    def test_main(self):
        from lambda_function import lambda_handler
        lambda_handler(None, None)
```

#### 使用 MagicMock 来指定mock一个类函数或方法

当我们需要mock一个类或函数的时候，我们想要模拟这个类方法的行为，比如requests.Session().get()的这个方法，我们模拟的 requests.Session()是有 get()的这个方法，此时我们可以用 MagicMock() 来模拟一个方法，之后用 return_value来表示这个方法的返回值，如果返回值也是一个方法，比如是一个get()方法，就可以用 return_value.get.return_value=MagicMock()的写法。
在@patch("requests.Session")的装饰器里，patch方法，会默认就将 requests.Session 创建一个 MagicMock()，然后在执行这个装饰器下面函数的时候，会将这个创建的 MagicMock()作为第一个参数，传给下面的装饰器函数，比如我下面的例子，则会自动传给 mock_session。而mock_response本身就是 pytest.fixture指定的一个 MagicMock()，其有两个属性，一个是 text，一个是status_code。我们在 mock_session_instance.get.return_value = mock_response的这行代码，就将 mock_session 的 get()方法(即 requests.Session().get()方法)和 mock_response进行了绑定
```python
@pytest.fixture
def mock_response():
    mock = MagicMock()
    mock.text = "Example Domain"
    mock.status_code = 200
    return mock

@patch("requests.Session")
def test_connect_to_somewhere(mock_session, mock_response):
    mock_session_instance = mock_session.return_value
    mock_session_instance.get.return_value = mock_response
    
    url = "https://www.example.com"
    connection = ConnectToSomewhere(url)
    
    assert connection.url == url
    assert connection.response.status_code == 200
    assert connection.response.text == "Example Domain"
    
    mock_session_instance.get.assert_called_once_with(url)

```
#### conftest.py
整个项目中所有需要使用的 fixture 固件，都会放到 conftest.py 文件里，pytest会自动引用，无需手动指定

#### 使用parametrize进行参数化测试
##### 示例一
当需要执行pytest，有多个mock的数据，希望对于这些数据进行测试，可以用 parametrize 来做，比如:

```python
messages = [
    "message #1",
    "message #2",
    "message #3"
]

def my_print(i):
    print(f"print {i}")

class TestAll:
    @pytest.mark.parametrize("msg", messages)
    def test_print(self, msg):
        my_print(msg)
```

注意：上述示例中， parametrize 会将 messages 列表里的每一个元素，在每一次测试的时候都传给 msg，下面的 test_print 函数里的参数，也必须叫 msg才行。

在真实的使用场景中，有可能 messages 列表是某一个方法的返回值，此时我们结合 unittest.mock 的 patch方法，将这个 messages 列表，作为方法的返回值，然后 test_print()里，就能用这个mock的返回值了

##### 示例二
比如我的代码是这样的，此时如果对每一个场景都单独写一个test函数，就会很麻烦，比如年纪大于18写一个，小于18写一个，年龄格式不对的多种场景(输入的是文本，年龄是负数，是小数等)，那么要写太多的test_is_audit函数，会很麻烦
```python
def is_adult(age: int):
    if not isinstance(age, int) or age < 0:
        raise ValueError("Age must be an integer and greater than 0")
    if age >= 18:
        return True
    else:
        return False

```

此时我们可以用 pytest.mark.parametrize 来进行参数化设置。在pytest.mark.parametrize里，第一个参数里存的是要给测试函数传的参数，如果有多个，用逗号分隔。第二个参数是一个list，里面存的是第一个参数对应的值，每一个list代表了一组值，可以测试多组。
由于上面的测试，包含函数成功返回，和异常抛出错误，所以我们分成两个test function进行测试.
```python
import pytest
from data import is_adult

@pytest.mark.parametrize("age, expected", [
    (20, True),
    (18, True),
    (17, False),
    (0, False),
])
def test_is_adult(age, expected):
    assert is_adult(age) == expected

@pytest.mark.parametrize("invalid_age", [
    "xxx",
    -1,
    3.14,
])
def test_is_adult_invalid_input(invalid_age):
    with pytest.raises(ValueError, match="Age must be an integer and greater than 0"):
        is_adult(invalid_age)

if __name__ == '__main__':
    pytest.main()

```


## Pycharm debug

在 debug python代码的时候，程序会卡在断点的那一样的位置，那一行的代码处于未被执行的状态。pycharm 对查看断点有几个按钮:

* step over: 直接执行这一行的代码
* step into：进入到这一行代码内部看具体执行过程
* Step into my code：跳过系统包和第三方包，只看自己的代码的执行过程。由IDE判断哪些包是第三方。
* step out：在 step into 的过程中，将 step into的剩余部分一次性执行完。step into + step out 相当于 step over
