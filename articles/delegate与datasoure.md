## 问题

作为iOS开发, 对于tableView那套写法应该烂熟于心了吧.  
那现在来问自己两个问题:   

**1. datasource和delegate分别是什么?**  
**2. 两者有什么区别?**

本篇文章主要简单翻译下引用的文章,觉得蛮浅显易懂的.可以看下原文.

## delegate

delegate主要用于传递数据或者在类和结构体之间通信.  
举个栗子:

```swift
protocol FirstVCDelegate {
  func passData(data: String)
}

class FirstVC {
 var delegate: FirstVCDelegate?
}

class SecondVC: FirstVCDelegate {
 func passData(data: String) {
  print("Something happened")
 }
}

let firstVC = FirstVC()
let secondVC = SecondVC()
firstVC.delegate = secondVC
firstVC.delegate?.passData(data: "a bunch of contracts”)
// "Something happened"
```
为了便于理解, 这边将委托的参与双发比作CEO/秘书.  
例子中的FirstVC就是CEO, SecondVC就是秘书.CEO这边每天都会收到很多的合约文件,没有那么多精力来处理. 
所以将文件都委托给(passData)给秘书来处理.

## datasource

delegate这个概念比较简单,就是简单的让别人帮你做某些事情.那datasource又承担什么角色呢?  
继续上面的例子.秘书处理完了合约总得告诉CEO结果吧? 
```
//我们更改一下协议, 让他支持结果的返回
protocol FirstVCDelegate {
 func passData(data: String) -> String
}

class FirstVC {
 var dataFromSecretary: String?
 var delegate: FirstVCDelegate?
}


class SecondVC: FirstVCDelegate {
 func passData(data: String) -> String {
  print("The CEO gave me \(data)")
  return "Too much work"
 }
}

let firstVC = FirstVC() // CEO: delegator
let secondVC = SecondVC() // Secretary: delegate
firstVC.delegate = secondVC
firstVC.dataFromSecretary = firstVC.delegate?.passData(data: "a bunch of contracts") // "The CEO gave me a bunch of contracts"
print(firstVC.dataFromSecretary) // "Too much work"
```

现在我们再回到我们最熟悉的tableviewDatasource
```
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
 // calculate the rows
 return rows
}
```
协议最后的Int就相当于例子中秘书给CEO的回应.  
对应到tableView, 它就能根据我们返回的Int来创建行等等.

## 总结

delegate的职能上面提过了,主要理解下DataSource.  
从名字可以看出, DataSouce主要是针对数据的!!!使用场景上面的例子很形象了, 委托别人准备数据, 数据准备好了再传递回来.

## 引用

[The Complete Understanding of Swift Delegate and Data Source](https://www.bobthedeveloper.io/blog/the-complete-understanding-of-swift-delegate-and-data-source)
