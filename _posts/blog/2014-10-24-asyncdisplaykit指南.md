---
layout: post
title: Asyncdisplaykit指南
description: AsyncDisplayKit的基本单元是node. ASDisplayNode是UIView和CALayer的抽象。ASDisplayNode是线程安全的，可以在工作线程中并行地初始化和配置整个node树。
category: blog
---

#Asyncdisplaykit指南
[Asyncdisplaykit github地址](https://github.com/facebook/AsyncDisplayKit)
##基本概念
&#8195;&#8195;AsyncDisplayKit的基本单元是node. ASDisplayNode是UIView和CALayer的抽象。ASDisplayNode是线程安全的，可以在工作线程中并行地初始化和配置整个node树。

&#8195;&#8195;如果保证帧率到60fps，那么所有的layout和drawing需要在16ms内完成。由于系统的开销，留给我们的只有大概10ms。

&#8195;&#8195;AsyncDisplayKit能够将image decoding, text sizing和rendering以及其他耗时的UI操作剥离主线程。
        
##nodes替代view
 
 &#8195;&#8195;Node的API与UIView相似，而且可以直接访问CALayer属性。
 添加view和layer，可以直接使用node.view 和 node.layer.
 
 AsyncDisplayKit包括几个强大的模块：
 
 * ASDisplayNode	- 相当于UIView，子类化实现自定义nodes
 * ASControlNode	- 相当于UIControl,子类化实现buttons
 * ASImageNode  	- 相当于UIimageView，异步decode image
 * ASTextNode   	- 相当于UITextView，基于TextKit实现，支持富文本 
 * ASTableView	- 相当于UITableView  	
    
&#8195;&#8195; 我们可以直接用node替换UIKit，全部基于node实现的图层树的ASDK效率更高，但即使仅仅替换一个view也会提高效率。

&#8195;&#8195;首先我们在主线程 同步的使用node.

&#8195;&#8195;代码示例：

		_imageView = [[UIImageView alloc] init];
		_imageView.image = [UIImage imageNamed:@"hello"];
		_imageView.frame = CGRectMake(10.0f, 10.0f, 40.0f, 40.0f);
		[self.view addSubview:_imageView];
&#8195;&#8195;使用Node替换：

		_imageNode = [[ASImageNode alloc] init];
		_imageNode.backgroundColor = [UIColor lightGrayColor];
		_imageNode.image = [UIImage imageNamed:@"hello"];
		_imageNode.frame = CGRectMake(10.0f, 10.0f, 40.0f, 40.0f);
		[self.view addSubview:_imageNode.view];

&#8195;&#8195;这里我们没有利用ASDK的异步 sizing 和 layout，但是性能已经有所提高。第一段代码在主线程decode image，第二段代码则在工作线程decode，而且可能在不同的CPU核心。
> 这里我们先展示了一个placeholder,然后再展示真实图片。但是对于text，这种延迟加载的方法不可行，后面会进行讨论。

##Botton nodes

&#8195;&#8195;ASImageNode 和 ASTextNode 都继承自ASControlNode,可以当做button使用，比如我们想做一个音乐播放器，先添加一个shuffle按钮：

![image](https://github.com/yufenglv/yufenglv.github.io/blob/master/images/blog/shuffle.png)

&#8195;&#8195;view controller的代码:

	- (void)viewDidLoad
	{
	  [super viewDidLoad];
	
	  // attribute a string
	  NSDictionary *attrs = @{
	                          NSFontAttributeName: [UIFont systemFontOfSize:12.0f],
	                          NSForegroundColorAttributeName: [UIColor redColor],
	                          };
	  NSAttributedString *string = [[NSAttributedString alloc] initWithString:@"shuffle"
	                                                               attributes:attrs];
	
	  // create the node
	  _shuffleNode = [[ASTextNode alloc] init];
	  _shuffleNode.attributedString = string;
	
	  // configure the button
	  _shuffleNode.userInteractionEnabled = YES; // opt into touch handling
	  [_shuffleNode addTarget:self
	                   action:@selector(buttonTapped:)
	         forControlEvents:ASControlNodeEventTouchUpInside];
	
	  // size all the things
	  CGRect b = self.view.bounds; // convenience
	  CGSize size = [_shuffleNode measure:CGSizeMake(b.size.width, FLT_MAX)];
	  CGPoint origin = CGPointMake(roundf( (b.size.width - size.width) / 2.0f ),
	                               roundf( (b.size.height - size.height) / 2.0f ));
	  _shuffleNode.frame = (CGRect){ origin, size };
	
	  // add to our view
	  [self.view addSubview:_shuffleNode.view];
	}
	
	- (void)buttonTapped:(id)sender
	{
	  NSLog(@"tapped!");
	}

上述代码可以正常运行，但是text的点击区域太小，解决方法：

	  // size all the things
	  /* ... */
	
	  // make the tap target taller
	  CGFloat extendY = roundf( (44.0f - size.height) / 2.0f );
	  _shuffleNode.hitTestSlop = UIEdgeInsetsMake(-extendY, 0.0f, -extendY, 0.0f);

所有的nodes都可以使用Hit-test slops。