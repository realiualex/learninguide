## 运算符优先级
`in` 的优先级，高于 `or`，比如下面的这个语句，永远会打印出来 `xx`
```python
if "BB" or "CC" in ["AA"]:
	print("xx")
```

原因是上述等于
```python
if bool("BB") or bool("CC" in ["AA"])
```

为了严谨，要这么写
```python
if "BB" in ["AA"] or "CC" in ["AA"]
```
如果判断的条件比较多，可以这么写

```python
list1=["AA"]
list2=["BB", "CC"]
if any([x for x in list2 if x in list1]):

```
## 初始化参数的可变对象

在定义一个方法的时候，如果给了默认参数，默认参数的数据类型是一个可变类型，一定要小心。因为python会将这个参数的值保存下来，在另外一次调用的时候与其赋值

```python
def get_user(name, users=[]):
    users.append(name)
    print(users)


if __name__ == "__main__":
    get_user("a")
    get_user("b")
    print(get_user.__defaults__)
```

解决办法是：使用不可变参数，如将其设置为None，然后判断如果是None，就使其变成list就可以了

```python
def get_user(name, users=None):
    if users is None:
        users = []
    users.append(name)
    print(users)


if __name__ == "__main__":
    get_user("a")
    get_user("b")
    print(get_user.__defaults__)
```

上述示例，使用class的时候，在__init__方法里定义也是一样要注意这个问题
```python
class Person:  
    def __init__(self, items=[]):  
        self.items = items  
  
  
if __name__ == "__main__":  
    p1 = Person()  
    p1.items.append(1)  
    print(p1.items)  
  
    p2 = Person()  
    p2.items.append(2)  
  
    print(p2.items)
```
解决方法
```python
class Person:  
    def __init__(self, items=None):  
        if items is None:  
            items = []  
        self.items = items  
  
  
if __name__ == "__main__":  
    p1 = Person()  
    p1.items.append(1)  
    print(p1.items)  
  
    p2 = Person()  
    p2.items.append(2)  
  
    print(p2.items)
```