### 隐式转换与隐式参数

#### 1、隐式转换

实现隐式转换需要在程序可见范围内定义隐式转换函数。Scala会自动是使用隐式转换函数。隐式转换函数与普通函数的唯一语法区别是，以`implicit`开头，并且一定要定义返回类型。

案例，特殊售票窗口：

```shell
scala> :paste
// Entering paste mode (ctrl-D to finish)

class SpecialPerson(val name:String)
class Student(val name:String)
class Older(val name:String)

implicit def object2SpecialPerson(obj:Object):SpecialPerson = {
 if(obj.getClass == classOf[Student]){val stu = obj.asInstanceOf[Student];new SpecialPerson(stu.name)}
 else if(obj.getClass == classOf[Older]){val older = obj.asInstanceOf[Older];new SpecialPerson(older.name)}
 else Nil
}
var ticketNumber = 0
def buySpecialTicket(p:SpecialPerson) = {
 ticketNumber+=1
 "T-"+ticketNumber
}

// Exiting paste mode, now interpreting.

<pastie>:15: warning: implicit conversion method object2SpecialPerson should be enabled
by making the implicit value scala.language.implicitConversions visible.
This can be achieved by adding the import clause 'import scala.language.implicitConversions'
or by setting the compiler option -language:implicitConversions.
See the Scaladoc for value scala.language.implicitConversions for a discussion
why the feature should be explicitly enabled.
implicit def object2SpecialPerson(obj:Object):SpecialPerson = {
             ^
defined class SpecialPerson
defined class Student
defined class Older
object2SpecialPerson: (obj: Object)SpecialPerson
ticketNumber: Int = 0
buySpecialTicket: (p: SpecialPerson)String

scala> val s = new Student("leo")
s: Student = Student@4d266391

scala> buySpecialTicket(s)
res0: String = T-1

scala> val o = new Older("Jike")
o: Older = Older@6afbe6a1

scala> buySpecialTicket(o)
res1: String = T-2
```

其中，`Student`和`Older`对象被隐式地转换为了`SpecialPerson`对象。

#### 2、使用隐式转换加强现有类型

隐式转换还可以加强现有类型的功能。即，为某个类定义一个加强版的类，并定义相互之间的隐式转换，从而让源类在使用加强版的类的方法时，由Scala自动地将源类隐式地转换为加强类，然后调用该方法。

案例，超人变身：

```shell
scala> :paste
// Entering paste mode (ctrl-D to finish)

class Man(val name:String)
class Superman(val name:String){
 def emitLaser = println("emit a laster!")
}

implicit def man2superman(man:Man):Superman = new Superman(man.name)


// Exiting paste mode, now interpreting.

defined class Man
defined class Superman
man2superman: (man: Man)Superman

scala> val leo = new Man("leo")
leo: Man = Man@618e7761

scala> leo.emitLaser
emit a laster!
```

`Man`类本没有`emtiLaser`函数，执行过程中，`Man`对象被隐式地转换为了`Superman`对象。

#### 3、隐式转换函数

Scala 会使用两种隐式转换，一种是源类型或者目标类型的伴生对象内的隐式转换函数；一种是当前程序作用域内的可以用唯一标识符表示的隐式转换函数。除了这两种情况，必须使用`import`语法引入某个包下的隐式转换函数。通常建议，仅在需要进行隐式转换的地方，比如某个函数或者方法内，用`import`导入隐式转换函数，这样可以缩小隐式转换函数的作用域，避免发生没必要的隐式转换。

#### 4、隐式转换的发生时机

- 调用某个函数，但是传给函数的参数类型与函数定义的参数类型不匹配

- 使用某个类型的对象，调用某个方法，而该类型没有该方法

- 使用某个类型的对象，调用某个方法，虽然该类型有这个方法，但是传给方法的参数类型与方法定义的参数类型不匹配

  案例，特殊售票窗口加强版：

  ```shell
  scala> :paste
  // Entering paste mode (ctrl-D to finish)
  
  class TicketHouse {
   var ticketNumber= 0
   def buySpecialTicket(p:SpecialPerson) = {
    ticketNumber += 1
    "T-"+ticketNumber
   }
  }
  
  // Exiting paste mode, now interpreting.
  
  defined class TicketHouse
  
  scala> val leo = new Student("leo")
  leo: Student = Student@217dc48e
  
  scala> val ticket = new TicketHouse
  ticket: TicketHouse = TicketHouse@7a5a26b7
  
  scala> ticket.buySpecialTicket(leo)
  res1: String = T-1
  ```

#### 5、隐式参数

隐式参数，指的是函数或者方法中，定义一个`implicit`修饰的参数，Scala 会尝试找到一个指定类型的`implicit`修饰的对象（即隐式值）作为参数。

Scala 查找隐式参数的范围：一种是当前作用域内可见的`val`或`var`定义的隐式变量；一种是隐式参数类型的伴生对象内的隐式值。

案例，考试签到：

```shell
scala> :paste
// Entering paste mode (ctrl-D to finish)

class SignPen{
 def write(content:String) = println(content)
}
implicit val signPen = new SignPen

def signForExam(name:String)(implicit signPen:SignPen){
 signPen.write(name+" come to exam in time.")
}

// Exiting paste mode, now interpreting.

defined class SignPen
signPen: SignPen = SignPen@6c4d0224
signForExam: (name: String)(implicit signPen: SignPen)Unit

scala> signForExam("leo")(signPen)
leo come to exam in time.
```

