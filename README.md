# YXMLParserDemo
使用XMLParser对XML数据进行解析

博客原文:[iOS开发------XML原生解析(NSXMLParser篇)](http://blog.csdn.net/runintolove/article/details/51567502)

由于项目需要，最近就研究了一些与视频流相关的知识，在学习的过程中发现，JSON作为轻量级的数据传输格式就显的非常不便.当然，这句话的意思就是说在学习过程中碰到的XML格式数据居多了呗，这时掌握一些XML的解析方法就显得重要了。
     
尽管之前转发过一篇介绍XML解析的博文[网络数据的XML解析](http://blog.csdn.net/RunIntoLove/article/details/50233379)，但总觉得千转不如一写，所以就花时间研究了一下。

很多第三方代码其实能够很好的帮助我们解析数据，但由于楼主的个性，在认可第三方代码的同时，还是要掌握一下它的原理的(当然这个原理，只是比较肤浅的原理了)，这一篇就记录一下iOS解析XML的原生类`NSXMLParser`.

本人用的是Mac自带的本地服务器，将php文件放置服务器文件夹即可，XML.php文件内容如下:
```PHP
<?php

//声明一个XML文本，并声明版本和编码
$xmlParse = new DOMDocument('1.0','utf-8');
$xmlParse->formatOutput = true;

//创建头结点
$rootElement = $xmlParse->createElement("Persons");
$xmlParse->appendChild($rootElement);


//添加测试数据-添加三组数据
for ($i=0; $i < 3; $i++) 
{ 
	$element = $xmlParse->createElement("Person");

	//姓名
	$name = $xmlParse->createElement("name");
	$name->appendChild($xmlParse->createTextNode("RITL".$i));//生成结点内容
	$element->appendChild($name);

	//年龄
	$age = $xmlParse->createElement("age");
	$age->appendChild($xmlParse->createTextNode($i * 3));
	$element->appendChild($age);

	//添加该结点
	$rootElement->appendChild($element);
}

//返回
echo $xmlParse->saveXML();

?>
```
<br>
为了比较直观，全部用的iOS原生框架(当然，也没有使用第三方的必要0.0)
下面是用到的所有延展属性:
```Objective-C
@import Foundation;

@interface ViewController ()<NSXMLParserDelegate>

/// @brief 存储xml对象的字典
@property (nonatomic, strong)NSMutableArray * persons;

/// @brief 记录当前编辑的xml结点对象
@property (nonatomic, strong)NSMutableDictionary * currentDictionary;

/// @brief 记录当前编辑的xml结点对象的key值
@property (nonatomic, copy)NSString * currentDictionaryKey;

@end
```
<br>
在ViewDidLoad中进行网络请求以及使用方法解析收到的XML信息：
```Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
    
    //拼接字符串路径
    NSURL * requestURL = [NSURL URLWithString:@"http://localhost/XML.php"];
    
    //网络请求对象
    NSURLSession * session = [NSURLSession sharedSession];
    
    //避免强引用,属性值需要变化，用__block修饰
    __block __weak typeof(self) copy_self = self;
    
    //初始化请求任务
    NSURLSessionDataTask * dataTask = [session dataTaskWithURL:requestURL completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        
        //获得请求返回的xml字符串
        NSString * dataString = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
        
        NSLog(@"xml = %@",dataString);
        
        //进行解析
        [copy_self decodeXML:dataString];
        
        //打印出结果:
        NSLog(@"result = %@",copy_self.persons);

    }];
    
    //开始请求
    [dataTask resume];
}
```
<br>
这里打印的xml格式为:
```XML
2016-06-02 15:32:03.773 YXMLParse[1069:27997] xml = <?xml version="1.0" encoding="utf-8"?>
<Persons>
  <Person>
    <name>RITL0</name>
    <age>0</age>
  </Person>
  <Person>
    <name>RITL1</name>
    <age>3</age>
  </Person>
  <Person>
    <name>RITL2</name>
    <age>6</age>
  </Person>
</Persons>
```
<br>
本博文的干货开始，首先是解析XML字符串的解析方法:`decodeXML：`:
```Objective-C
/**
 *  解析XML字符串
 *
 *  @param xmlString 解析的XML字符串
 *
 *  @return 是否解析成功，true表示成功，false表示失败
 */
- (BOOL)decodeXML:(NSString *)xmlString
{
    //初始化NSXMLParse对象
    NSXMLParser * xmlParse = [[NSXMLParser alloc]initWithData:[xmlString dataUsingEncoding:NSUTF8StringEncoding]];
    
    //设置代理
    xmlParse.delegate = self;
    
    //开始解析
    return [xmlParse parse];
}
```
<br>
通过协议方法不断的进行解析，同时我也按照自己的理解，尽可能的进行了注释:
```Objective-C
#pragma mark - *************** <NSXMLParserDelegate>

/**
 *  开始解析进行回调的Delegate Function
 *
 *  @param parser 执行回调方法的NSXMLParser方法
 */
- (void)parserDidStartDocument:(NSXMLParser *)parser
{
    //初始化存放xml数据字典的数组
    self.persons = [NSMutableArray arrayWithCapacity:0];
}
```
```Objective-C
/**
 *  准备解析结点进行的回调
 *  在此处可以获取每个xml结点所传递的信息，如(xmlns--类似命名空间)
 *
 *  @param parser        执行回调方法的NSXMLParser对象
 *  @param elementName   结点的字符串描述(如name..)
 *  @param namespaceURI  命名空间的统一资源标志符字符串描述
 *  @param qName         命名空间的字符串描述
 *  @param attributeDict 参数字典
 */
- (void)parser:(NSXMLParser *)parser didStartElement:(NSString *)elementName
  namespaceURI:(nullable NSString *)namespaceURI
 qualifiedName:(nullable NSString *)qName attributes:(NSDictionary<NSString *, NSString *> *)attributeDict
{
    //表示开始遍历子结点了
    if ([elementName isEqualToString:@"Person"])//对Person标签进行比对
    {
        //新建一个索引为elementName的key字典
        NSMutableDictionary * elementNameDictionary = [NSMutableDictionary dictionary];
        
        //将数据添加到当前数组
        [self.persons addObject:elementNameDictionary];
        
        //记录当前进行操作的结点字典
        self.currentDictionary = elementNameDictionary;
    }
   
    //表示不是Persons的子节点,因为是else if,也保证
    else if (![elementName isEqualToString:@"Persons"])//排除Person以及Persons标签
    {
        //记录当前的结点值
        self.currentDictionaryKey = elementName;
        
        //设置当前element的值,先附初始值
        [self.currentDictionary addEntriesFromDictionary:@{elementName:@""}];
    }
}
```
```Objective-C
/**
 *  获得首尾结点间内容信息的内容
 *
 *  @param parser 执行回调方法的NSXMLParse对象
 *  @param string 结点间的内容
 *  如果结点之间的内容是结点段，那么返回的string首字符为unichar类型的‘\n’
 *  如果不是结点段，那么直接返回之间的信息内容
 */
-(void)parser:(NSXMLParser *)parser foundCharacters:(NSString *)string
{
    //如果结点内容的第一个字符是‘\n’,表示是包含其他结点的结点
    if (string.length > 0 && [string characterAtIndex:0] != '\n')
    {
        //对当前字典进行初始化赋值
        [self.currentDictionary setValue:string forKey:self.currentDictionaryKey];
    }   
}
```
```Objective-C
/**
 *  某个结点解析完毕进行的回调
 *
 *  @param parser       执行回调方法的NSXMLParse对象
 *  @param elementName  结点的字符串描述(如name..)
 *  @param namespaceURI 命名空间的统一资源标志符字符串描述
 *  @param qName        命名空间的字符串描述
 */
-(void)parser:(NSXMLParser *)parser didEndElement:(NSString *)elementName namespaceURI:(NSString *)namespaceURI qualifiedName:(NSString *)qName
{
    //只有结束Person的时候清除相关数据
    if ([elementName isEqualToString:@"Person"])
    {
        //cleat data
        self.currentDictionary = nil;
        self.currentDictionaryKey = nil;
    }
}
```
```Objective-C
/**
 *  XML解析完成进行的回调
 *
 *  @param parser 执行回调方法的NSXMLParse对象
 */
- (void)parserDidEndDocument:(NSXMLParser *)parser
{
    
}
```
最后输出一下解析完毕之后添加到数组中的数据，打印如下：
```Ruby
2016-06-02 15:32:03.774 YXMLParse[1069:27997] result = (
        {
        age = 0;
        name = RITL0;
    },
        {
        age = 3;
        name = RITL1;
    },
        {
        age = 6;
        name = RITL2;
    }
)//完成解析
```

