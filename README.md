# iOS_Dynamic_Plug-in
iOS 动态插件，实时模块更新
#动态库和静态库
静态库和动态库是相对编译期和运行期:静态库在程序编译时会被链接到目标代码中，程序运行时将不再需要改静态库；而动态库在程序编译时并不会被链接到目标代码中，只是在程序运行时才被载入，因为在程序运行期间需要动态库的存在。

####静态库的好处
- 模块化，分工合作，提高了代码的复用及核心技术的保密程度；
- 避免少量改动经常导致大量的重复编译链接；
- 也可以重用，注意不是专享使用。
####动态库的好处
- 可以将最终可执行文件体积缩小，将整个应用程序分模块，团队合作，将进行分工，影响比较小；
- 多个应用程序共享内存中得同一份库文件，节省资源；
- 可以不重新编译链接可执行文件程序的前提下，更新动态库文件达到更新应用程序的目的；
- 应用插件化。
####软件版本实时模块升级
共享执行可执行文件，在其他大部分平台上，动态库都可以用于不同应用间共享，这就大大节省了内存。从目前来看，iOS仍然不允许进程间共享动态库，己iOS上的动态库只能是私有的，因为我们仍然不能将动态库文件放置在除了自身沙盒以为的其他任何地方。不过iOS8上开发了App Extension功能，可以为一个应用创建插件，这样主app和插件之间共享动态库还是可以行的。

#动态库和主工程的创建
本文章只针对动态库创建，软件版本实时模块升级进行说明，静态库不做详细解释说明。

####动态库创建
- 创建工程类型为Framework & Library 下的Cocoa Touch Framework工程，工程命名DynamicLink;
- 创建继承UIViewController命名为ViewController的控制器，设置背景颜色；
  // 动态库视图颜色
  self.view.backgroundColor = [UIColor greenColor];
- 创建继承NSObject命名为DynamicOpenMenth文件，在DynamicOpenMenth.h中

// 动态链对外开发方法
 - (void)startWithObject:(id)object withBundle:(NSBundle *)bundle;
在DynamicOpenMenth.m中方法实现

  - (void)startWithObject:(id)object withBundle:(NSBundle *)bundle
  {
    // 初始化第一个controller
    // 这里的重点是资源文件的加载,通常我们在初始化的时候并不是很在意bundle:这个参数，
    // 其实我们所用到的图片、xib等资源文件都是在程序内部中获取的，也就是我们常用的[NSBundle mainBundle]中获取，所谓的NSBundle本质上就是一个路径，mainBundle指向的是.app下。
   // 而如果我们不指定bundle，则会默认从.app路径下去寻找资源。
    // 不过很显然，我们的动态库是放到“主程序”的document文件下的，所以资源文件是不可能在[NSbundle mainBundle]中获取到的，所以这里我们需要指定bundle参数，这也是传递framework的路径的意义所在

    ViewController *vc = [[ViewController alloc] init];
   vc.root_bundle = bundle;
    //转换传递过来的mainCon参数，实现界面跳转
    UIViewController *viewCon = (UIViewController *)object;
    [viewCon presentViewController:vc animated:YES completion:^{
    NSLog(@"跳转到动态更新模块成功!");
  }];

}
在Build Only Device下编译程序，生成DynamicLink.Framework文件，文件可在工程目录下Products文件夹下Show In Finder中找到；
主工程创建
创建Single View Application的工程，命名为DynamicLibrary。

在ViewController.m中实现以下方法

  - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
  {
    // 动态库测试
    [self performSelector:@selector(dynamicLibraryClick) withObject:nil];
  }

  - (void)dynamicLibraryClick
  {
    // document路径
    NSArray* paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES);
    NSString *documentDirectory = nil;
    if ([paths count] != 0)
      documentDirectory = [paths objectAtIndex:0];
    NSLog(@"\nDocuments 路径 = %@\n",documentDirectory);
    //拼接我们放到document中的framework路径
    NSString *libName = @"DynamicLink.framework";
    NSString *destLibPath = [documentDirectory stringByAppendingPathComponent:libName];

    //判断一下有没有这个文件的存在　如果没有直接跳出
    NSFileManager *manager = [NSFileManager defaultManager];
    if (![manager fileExistsAtPath:destLibPath]) {
      NSLog(@"There isn't have the file");
      return;
   }

    //复制到程序中
    NSError *error = nil;

    //加载方式二：使用NSBundle加载动态库
    NSBundle *frameworkBundle = [NSBundle bundleWithPath:destLibPath];

    if (frameworkBundle) {
      NSLog(@"bundle load framework success.");
    }else {
      NSLog(@"bundle load framework err:%@",error);
      return;
    }

  //通过NSClassFromString方式读取类,FrameWorkStart为动态库中入口类
  Class pacteraClass = NSClassFromString(@"DynamicOpenMenth");
  if (!pacteraClass) {
      NSLog(@"Unable to get TestDylib class");
      return;
  }

  // 1.初始化方式采用下面的形式,alloc　init的形式是行不通的,同样，直接使用PacteraFramework类初始化也是不正确的
  // 2.通过- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
  // 3.方法调用入口方法（startWithObject:withBundle:），并传递参数（withObject:self withObject:frameworkBundle）

  NSObject *pacteraObject = [pacteraClass new];
  [pacteraObject performSelector:@selector(startWithObject:withBundle:) withObject:self withObject:frameworkBundle];
}
将动态库生成的DynamicLibrary.Framework动态库文件导入到主工程运行下的Documents文件下；

注意：模拟器运行下，直接讲DynamicLibrary.Framework动态库文件拖拽到打印出的文件夹中即可，真机运行需打开iTunes倒入到DynamicLibrary应用中。
以上为动态库加载主要实现，存在一些的问题需要感兴趣的朋友一起讨论，比如 Class pacteraClass = NSClassFromString(@"DynamicOpenMenth");读取类的时候为什么值为nil等。

技术交流群：193996724
