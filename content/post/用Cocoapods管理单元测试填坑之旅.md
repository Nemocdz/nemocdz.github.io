---
title: "用Cocoapods管理单元测试填坑之旅"
date: 2017-04-13T03:02:13+08:00
draft: false
---

上周接到了个需求，老大要我把项目代码里某个库覆盖上单元测试。而那个库没有Demo，平时都是集成在工程里开发的。为啥没有Demo，因为那个库依赖很重，说是个库，实际只是把代码用cocoapods拆分罢了……平时开发的时候，大家都是把库集成在主工程里运行。我想，单测写在主工程的target里，这样会显得很杂，给人感觉是给整个工程做单元测试。那能不能弄一个Pod，专门把单测代码写在里面呢，既能git管理，又可以分类管理。殊不知，这个过程，坑如此多。

<!--more-->

### 让Pod的target类型变为XCTest

像往常一样我在命令行里，敲下一个熟悉的命令

```shell
pod lib create UnitTestPod
```

经过简单的四个问题后，一个Pod生成了。

并在Demo里的Podfile熟练的写下

```ruby
target 'UnitTestPodDemo' do
	pod 'UnitTestPod' , :path => 'UnitTestPod/UnitTestPod.podspec'
end
```

经过一番pod install后，打开了xcworkspace。

![屏幕快照 2017-04-12 23.38.20](http://ww2.sinaimg.cn/large/006tNc79ly1fekhgrb3ocj308g08cwes.jpg)

？？？这个Target的icon，有点不对劲啊…原来Cocoapods默认生成的Pod是作为一个Target集成进Pods这个XcodeProject里的，而且Target的默认类型是Static Libaray，也就是一个静态库。咋办呢？我要的是XCTest的Target类型。好吧，那看看Cocoapods的[源码](https://github.com/CocoaPods/CocoaPods)吧。怎么找呢，有点Ruby基础的人知道，Podfile里面写东西实际上都是用Ruby语法实现的DSL（Domain Specific Language 领域特定语言）。也就是相当于实现了一套语法规则，比如target do，比如pod，比如:path=>这些，在Cocoapods都有对应的语法实现。在执行pod install的过程中，podfile中的信息被解析，然后把pod的信息进行处理，并生成target并生成集成后的project文件。那对target的类型的写入，也肯定也在生成Pods.xcodeproj的过程中。查看cocoapods的installer.rb，发现里面有个intall方法如下：

```ruby
def install!
    prepare
    resolve_dependencies
    download_dependencies
    verify_no_duplicate_framework_and_library_names
    verify_no_static_framework_transitive_dependencies
    verify_framework_usage
    generate_pods_project
    if installation_options.integrate_targets?
       integrate_user_project
    else
       UI.section 'Skipping User Project Integration'
    end
    perform_post_install_actions
end
```

看起来这个过程可能是install的过程，里面的``generate_pods_project``方法，应该就是生成pod_project的方法了。这个方法实现如下：

```ruby
def generate_pods_project(generator = create_generator)
    UI.section 'Generating Pods project' do
    	generator.generate!
        @pods_project = generator.project
        run_podfile_post_install_hooks
        generator.write
        generator.share_development_pod_schemes
        write_lockfiles
     end
end
```

再查看其类pods_project_generator.rb的generate方法(相当于初始化方法)

```ruby
def generate!
  prepare
  install_file_references
  install_libraries
  set_target_dependencies
end
```

这里面大概就是将依赖的文件和library加入工程，设置目标依赖。再看看``install_libraries``方法

```ruby
def install_libraries
	UI.message '- Installing targets' do
		pod_targets.sort_by(&:name).each do |pod_target|
		 target_installer = PodTargetInstaller.new(sandbox, pod_target)
		 target_installer.install!
        end
		aggregate_targets.sort_by(&:name).each do |target|
		 target_installer = AggregateTargetInstaller.new(sandbox, target)
		 target_installer.install!
        end
		 add_system_framework_dependencies
	 end
end
```

在看看之类这个target_Installer实例，就是PodTargetInstaller类生成的。看来这个类，应该是生成target的类。再看看这个pod_target_installer.rb，里面搜索一下type，居然没有？？？难道找错了？我又仔细看了一下，发现开始处有怎么一行代码：

```ruby
class PodTargetInstaller < TargetInstaller
```

看到这行代码，我感觉和OC里的``AClass : BClass``，这大概是继承吧。又找到了target_installer.rb里面的TargetInstaller类，终于发现了一行蛛丝马迹。

```ruby
def add_target
	product_type = target.product_type
```

也就是说，target的类型，在cocoapods里，是target的product_type属性。剩下的就是要把target的product_type，设置成cocoapods里的XCTest类型了。那么接下来，就有两个问题要解决

* Cocoapods里的XCTest类型，是怎么表示的

* 在哪里插入这个改变类型的操作

看Cocoapods源码里发现，实际上对于Xcodeprojcet相关文件的写入和修改，是通过Cocoapods另一个单独的库Xcodeproj实现的，再看看它的源码。搜索Unit Test，在constants.rb里发现这里定义了所有ProductType的别名。![屏幕快照 2017-04-13 00.53.09](http://ww3.sinaimg.cn/large/006tNc79ly1fekhgv0x7uj311o0o8gsp.jpg)

也就是说，product_type属性其实是个String类型，而XCTest的类型，就是``'com.apple.product-type.bundle.unit-test'``那么第一个问题就解决了。

剩下就是在哪里改变它了。还记得刚才的``generate_pods_project``方法么

```ruby
def generate_pods_project(generator = create_generator)
    UI.section 'Generating Pods project' do
    	generator.generate!
        @pods_project = generator.project
        run_podfile_post_install_hooks
        generator.write
        generator.share_development_pod_schemes
        write_lockfiles
     end
end
```

里面有个``run_podfile_post_install_hooks``方法，难道，官方提供了podfile里的hook方法？查看Cocoapods[官网](https://guides.cocoapods.org/)，发现官方还真提供了hooks！ 

有三种类型，plugin是加上插件，pre_install是提供hooks在下载好pod但还没被install的时候，而post_install是东西都生成好了，还没被写入磁盘的时候。看来在生成好后改变就可以了。

根据官网的例子和前面的探索，可以在podfile里加上hook方法如下

```ruby
post_install do |installer|
	installer.pods_project.targets.each do |target|
		 if target.name == "UnitTestPod"
             target.product_type = 'com.apple.product-type.bundle.unit-test'
             end
             puts "`#{target.name}` change type to `#{target.product_type}`"
         end
	end
end
```

就是对installer的pod_project这个project对象的targerts数组每一个执行一个循环，当找到我们需要的target时，改变target的类型。puts是ruby里的log，相当于NSLog，printf。

好了，重新pod install，可以发现，target已经变成了test的type了，而Xcode里的test栏也有了库里的Test了。![屏幕快照 2017-04-13 01.18.45](http://ww3.sinaimg.cn/large/006tNc79ly1fekhgt6s3rj30ea070mxj.jpg)

但这个时候，我们是无法引入并识别其它第三方库的方法的，也找不到其它第三方库的符号表。

## 让Pod识别到别的静态库

这个时候，我们其它普通静态库的方法，在podspec里写上`s.dependency 'CDZPicker'`某个库，再次pod install。这次，ide识别了，再次运行test，发现编译报错了，找不到符号表。如下图：

![屏幕快照 2017-04-13 01.26.59](http://ww4.sinaimg.cn/large/006tNc79ly1fekhgnopmsj30uo05q769.jpg)

同时可以发现，实际上运行test的时候，把依赖的库编译了一遍。也就是说，现在这个test的target没办法找到编译的产物（.o）。而在Cocoapods里生成的主工程的target里，为啥就能找到第三方库的编译后产物呢？先讲讲正常引用静态库的因素，一个是告诉Xcode去哪里找，也就是Target里Building Setting里的Library SearchPaths，里面指定了找编译产物的路径，另一个是告诉Xcode，要链接的静态库的名字，也就是Building Setting里的Other Linker Flags里用-l"静态库编译产物名（去掉前面lib）"标识。查看其BuildingSetting，发现其两个决定的因素，都被Cocoapods配置好了，也就是target xxx do里做的。

![屏幕快照 2017-04-13 01.31.41](http://ww3.sinaimg.cn/large/006tNc79ly1fekhgsapsbj30vu044js1.jpg)

![屏幕快照 2017-04-13 01.32.05](http://ww1.sinaimg.cn/large/006tNc79ly1fekhgqec57j315404q75f.jpg)

而默认导入的Pod的target，这两个参数都是""，也就是空的。这个时候，我们看看把这两个设置设成默认值会怎么样呢？在post_install的hook方法里加入下面的代码

```ruby
target.build_configurations.each do |config|
	config.build_settings['OTHER_LDFLAGS'] = '$(inherited)'
	config.build_settings['LIBRARY_SEARCH_PATHS'] = '$(inherited)'
end
```

怎么知道这些设置对应的键值是这些呢？Cocoapods实际上是通过生成XCConfig文件还配置这些的，在项目里搜索后缀名是xcconifg的文件，发现了Cocoapods写的那些部分。

![屏幕快照 2017-04-13 01.45.19](http://ww4.sinaimg.cn/large/006tNc79ly1fekhgplxufj31kw07vagu.jpg)

完整的XCConfig编写可以参考Github上这个[仓库](https://github.com/jspahrsummers/xcconfigs)。而为啥设成默认就可以了呢？在Building Setting里点击Level模式查看，发现对于每一项设置，都有5个地方可以设置，层级从左到右，如果在左边的层级设置了，就取最左边的设置覆盖右边的，而绿色的框代表每行的设置正在应用的来源是来自哪里，也就是绿色框的代表的是最终的设置。

![屏幕快照 2017-04-13 01.51.44](http://ww4.sinaimg.cn/large/006tNc79ly1fekhgwwu54j31kw09ste8.jpg)

而加上'$(inherited)'的作用，就是让值从下往上透上来，也就是，会把右边的一栏的设置也加进来，右边的如果也有'$(inherited)'，那么依次类推。上面五栏里分别是Resolve，Target，ConfigFile，Project，Default。第一层Resolve我不太清楚，可能是编译最终修复之类的，第二层就是target里修改的，第三层就是读取对应的xcconfig后缀文件里的配置，，第四层是Project里修改的，最后是系统默认的。而这也是为什么一个Project可以对应多个Target的原因，target只是覆盖了设置而已。而Cocoapods是把xcconfig写好，并让上层设置为'$(inherited)'，从而达到把配置写进Xcode。而在hook方法里，把target的config更改，相当于把上层设置好，取下层XCConfig里的值。而我们看到Cocoapods默认帮我们生成的Pod的XCConfig里，里面已经写好了正确的Library Search Path。

![屏幕快照 2017-04-13 01.59.08](http://ww2.sinaimg.cn/large/006tNc79ly1fekhgom01ij31kw073tg2.jpg)

剩下就是Other Link Flag了，这个也很好解决，在podspec里写上

```ruby
s.pod_target_xcconfig = {'OTHER_LDFLAGS' => '$(inherited) -l"CDZPicker"'}
```

就好了。重新pod install一下，写上单元测试代码，点击test，OK，一切顺利。

等等，点击主工程的Run，好像跑不起来了……

![屏幕快照 2017-04-13 02.05.31](http://ww4.sinaimg.cn/large/006tNc79ly1fekhgu5x1nj30xq03eab6.jpg)

### 曲线救国解决Test库编译问题

看看报错，找不到库的编译产物。思考了一下，应该是因为Test的target比较特殊，并不会编译自己并产生编译产物。而记得刚才主工程的Other Linker Flag吗，里面因为Cocoapods认为这个库是个静态库，是有编译产物的且要链接的，所以有"-l'UnitTestPod'"。去掉之后，果然就可以编译成功了。

但是这样不够优雅，每次Pod install之后，难道都要这样删掉吗？

这时我想起Podfile里一个参数configurations，一般我们会指定例如``:configurations => ['Debug']``这样来说明这个库在Debug下才被编译并链接进主工程。既然这样Release模式下就不会被链接，我们就可以利用这个特性，新建一个没用的Configuration,让Podfile指定就可以了。在Project的Info里新建一个Test的Configuration，并在Podfile里的Pod指定``:configurations => ['Test']``。

![屏幕快照 2017-04-13 02.23.54](http://ww1.sinaimg.cn/large/006tNc79ly1fekhgvz3jgj31ac0dcdhs.jpg)

重新Pod install，可以看到Debug的Other Linker Flag里已经没有"-l'UnitTestPod'"来指定链接Pod了。编译自然也就成功了。

## 最后

最后的结果是，因为我们工程里有很多预编译的库，而预编译的库通过我们公司的一套方案来设置，可以根据每个人的设置，可以选择源码或预编译，而生成的编译产物预编译的有一个前缀。因为没办法确定每个人某个库的依赖的库是否是预编译的，所以静态库的名字是不确定的，可能是有前缀可能没有，和每个人开发环境有关。在podspec里也没办法通过“-l‘静态库编译产物名’”来链接到正确的编译产物，也就没办法用这种方式进行下去管理单元测试。没想到最后填坑的结果，却和公司另一套别的方案冲突了。虽然有些难过，但是在研究这个问题的几天里，我一个完全没看过Ruby也不了解链接，Cocoapods也是按着例子写的小白，开始到了解一些Ruby，Cocoapods做了什么，Bulding Setting，XCConfig，静态库怎么被链接的知识，还是感觉很开心的。

所有源码和[Demo](https://github.com/Nemocdz/UnitTestPodDemo)
如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

##### 参考链接

* [细聊 Cocoapods 与 Xcode 工程配置 ]( https://bestswifter.com/cocoapods/)
* [CocoaPods都做了什么](http://draveness.me/cocoapods.html)
* [使用代码为Xcode工程添加文件](http://draveness.me/bei-xcodeproj-keng-de-zhe-ji-tian.html)
* [Cocoapods官网](https://cocoapods.org/)


