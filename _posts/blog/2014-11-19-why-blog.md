
layout:     post
title:      我为什么写博客？
category: blog

##AsyncDisplayKit分析
### 需要iOS7，why?
### ASDisplayNode分析
	
	the node will participate in the current asyncdisplaykit_async_transaction and 
	 * do its rendering on the displayQueue instead of the main thread.
	 
异步渲染的过程:

  * 当view加入图层树中，-needsDisplay为true 
  * 在layout之后， Core Animation 调用 _ASDisplayLayer的 -display
  * -display 将一个渲染操作压入队列 displayQueue
  * 当render block执行时，才调用代理对象的display方法(-drawRect: 或者 display)
  * 代理对象通过display方法生成内容，并且在asyncdisplaykit_async_transaction中添加一个操作
  * 当asyncdisplaykit_async_transaction里面的所有渲染操作执行完成，block的completion回调把所有的内容在同一帧中赋给layers
  
非异步渲染过程:

  * 当view加入图层树中，-needsDisplay为true
  * 在layout之后， Core Animation 调用 _ASDisplayLayer的 -display
  * -display立刻调用代理对象的display方法（-drawRect 或者 -display）
  * display立刻将内容赋给layer
 
###_ASDisplayLayer

`displayQueue` 并行执行队列，队列的优先权为`DISPATCH_QUEUE_PRIORITY_HIGH`。

### ASTableView

ASTableView.m - nodeForRowAtIndexPath在异步线程执行
	
	- (ASCellNode *)tableView:(ASTableView *)tableView nodeForRowAtIndexPath:(NSIndexPath *)indexPath;
	在下面的函数中被调用
	- (ASCellNode *)rangeController:(ASRangeController *)rangeController nodeForIndexPath:(NSIndexPath *)indexPath
	{
	  ASDisplayNodeAssertNotMainThread();
	  return [_asyncDataSource tableView:self nodeForRowAtIndexPath:indexPath];
	}
在ASRangeController.m中的- (void)sizeNextBlock函数在异步线程中调用了nodeForIndexPatch.
###ASAsyncTransaction
ASAsyncTransaction provides lightweight transaction semantics for asynchronous operations.

ASAsyncTransaction特性:

 - Transactions包括一组`operations`, 每个`operation`包括一个`execution block` 和一个`completion block`.
 -  `execution block` 返回一个对象，然后这个对象作为参数传递给`completion block`.
 -  被加入到`transaction`的`Execution blocks` 在 `global background dispatch queues`中并行执行。`completion blocks`进入callback queue.
 - 每个操作的`completion block` 都会被执行，cancelation不起作用.
   但是如果 `transaction`被取消，那么`execution blocks` 将不会执行
 - Operation completion blocks are always executed in the order they were added to the transaction, assuming the callback queue is serial of course.
 
	 	- (void)addOperationWithBlock:(asyncdisplaykit_async_transaction_operation_block_t)block
	 			queue:(dispatch_queue_t)queue 
	 			completion:(asyncdisplaykit_async_transaction_operation_completion_block_t)completion
 
将displayBlock加入到displayQueue中，并且立即开始执行。将displayBlock的返回值是UIImage, 赋值给ASDisplayNodeAsyncTransactionOperation的value属性，completeBlock的参数之一即为返回的UIImage，completeBlock在主线程执行，将image赋值给layer的contents.`_layer.contents = (id)image.CGImage;`
		
###_ASAsyncTransactionGroup
 A group of transaction container layers, for which the current transactions are committed together at the end of the next runloop tick.
 
`mainTransactionGroup`在每个main runloop将group中的所有`_ASAsyncTransaction`提交。
###CALayer (ASDisplayNodeAsyncTransactionContainer)
	 - (_ASAsyncTransaction *)asyncdisplaykit_asyncTransaction
返回的`_ASAsyncTransaction`存放在NSHashTable(要求iOS6.0)。

### ASTextNodeRenderer.mm
	- (void)_initializeTextKitComponentsWithAttributedString:(NSAttributedString *)attributedString{
	...
	//NSTextContainer只有iOS7支持
	  _textContainer = [[NSTextContainer alloc] initWithSize:_constrainedSize];
	...
	}	


