---
title: python修饰符
date: 2018-07-08 23:19:47
tags: 机器学习，图像处理
categories: 学习
---
本来分类应该分类到编程这一类下，但是想想所有关于python的学习都放在机器学习下，另外当时学习python的目的就是搞机器学习，所以也就放在了这一类下面，其实python的语法我就不再进行介绍了，面向对象的语言的语法大致上都差不多只是略有点区别而已，但是以前一直有一点不明确，那就是关于修饰符的作用，在做项目的时候看到了并且胡乱使用了一通，今天特意就这个问题进行一下总结。
## 内置修饰符
python的类内置修饰符有三个，分别为staticmethod，classmethod以及property，其作用分别为1.把类中定义的实例方法变成静态方法；2.把类中定义的实例方法变成类方法；3.将类成员修改为类属性。由于Python模块中可以定义函数，因此静态方法和类方法的用处并不多；下面对于staticmethod和classmethod两种方法进行说明，我们知道在定义类和类的成员函数后对于成员函数的调用首先要实例化一个类对象，然后通过类对象来调用成员函数，但是对于静态成员函数的调用则不需要对类进行实例化，实际上以上两种修饰符的作用都类似，但是又有一些区别。我们看几个实例：  
```python
class testclassmethod:
	@classmethod
	def printd(cls,a,b):
		print(a+b)
if __name__=='__main__':
	testclassmethod.printd(3,5)
```
从上面这个例子我们可以看出通过classmethod修饰符的作用，通过修饰符修饰之后成员方法不需要通过实例化就可以对方法进行调用，但是通过修饰符修饰的方法中第一个参数并不是我们常见的self了，变成了cls，这个参数实际上表示类自身，通过这个参数可以调用类的属性和方法，相当于类的一个实例吧；下面我们看看staticmethod：
```python
class teststaticmethod:
	@staticmethod
	def printd(a,b):
		print(a+b)
if __name__=='__main__':
	teststaticmethod.printd(3,5)
	teststaticmethod().printd(3,5)
```
比较两部分代码我们可以看到，最主要的区别在于staticmethod进行修饰后不需要再定义一个代表类本身的参数。好了以上两种修饰符就介绍到这里，下面介绍装饰器的作用：
## 装饰器
在python中我们经常可以看到如下写法：
```python
def test1(f):
	f()
	print('test1')

@test1
def test2():
	print('test2')
	
if __name__=='__main__':
	test2()
```
这个简单的例子说明了装饰器的作用，实际上我们的调用顺序是test1，然后在test1中调用传入函数参数f，然后调用print，我们可以将装饰器理解成函数指针，将装饰器后的函数作为参数输入到装饰器函数中进行调用。  
P.S. 以上代码都在python3.7 下编译通过，不同版本的python可能略有区别