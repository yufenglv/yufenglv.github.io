# 采用最新的Objective-C特性

### instancetype
 使用instancetype来替换id.

####Why

1. instancetype返回的对象类型是`MyObject *`. id返回的对象类型是id.

		@interface MyObject : NSObject
		+ (instancetype)factoryMethodA;
		+ (id)factoryMethodB;
		@end
		 
		@implementation MyObject
		+ (instancetype)factoryMethodA { return [[[self class] alloc] init]; }
		+ (id)factoryMethodB { return [[[self class] alloc] init]; }
		@end
		 
		void doSomething() {
		    NSUInteger x, y;
		 
		    // 编译错误： Return type of +factoryMethodA is taken to be "MyObject *"
		    x = [[MyObject factoryMethodA] count]; 
		    
		    //正确： Return type of +factoryMethodB is "id"
	    	 y = [[MyObject factoryMethodB] count]; 
		    
		    //警告： Incompatible pointer types initializing 'NSString *' with an expression of type 'MyObject *'
		    NSString *aString = [MyObject factoryMethodA]; 
		    
		    //正确
			NSString *bString = [MyObject factoryMethodB];

		}
#####Tips: 编译器会将以 `alloc`,`init`或者`new`开头的,并且返回类型是id的函数的返回类型转换成`instancetype`。但是不会转换其他的函数。 Objective-C 的代码规范是显式的使用`instancetype`. 

### Properties

### Enumeration Macros

1. 使用`NS_ENUM`来定义枚举。

		typedef NS_ENUM(NSInteger, UITableViewCellStyle) {
		        UITableViewCellStyleDefault,
		        UITableViewCellStyleValue1,
		        UITableViewCellStyleValue2,
		        UITableViewCellStyleSubtitle
		};
	
2. 使用`NS_OPTIONS`来定义选项。
	
		typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
	        UIViewAutoresizingNone                 = 0,
	        UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
	        UIViewAutoresizingFlexibleWidth        = 1 << 1,
	        UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
	        UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
	        UIViewAutoresizingFlexibleHeight       = 1 << 4,
	        UIViewAutoresizingFlexibleBottomMargin = 1 << 5
		};
		
### Object Initialization
对Designated Initializer进行标记区分，添加`NS_DESIGNATED_INITIALIZER`宏定义，限制：
1.  Designated Initializer必须调用父类的Designated Initializer，比如`[super init...]`
2.  非Designated Initializer必须调用Designated Initializer，比如`[self init...]`
3.  如果有多个Designated Initializer，每个都必须调用父类的Designated Initializer。
	
	- (instancetype)init NS_DESIGNATED_INITIALIZER;


