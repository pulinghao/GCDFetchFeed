# GCDFetchFeed

## Done

* RSS 解析成可用 model `SMFeedStore`

* dispatch_group 监听多 feeds 的 fetch

使用`dispatch_group`相关的接口，及并发队列实现。具体参看下面的接口：
这里，`dispatch_group`有实现了三个作用：
1. 控制网络请求回来
2. 确保数据被写入数据库，才释放了group
3. 在所有任务完成之后,即`dispatch_notify`,才把RAC的信号设置为Complete


```
- (RACSignal *)fetchAllFeedWithModelArray:(NSMutableArray *)modelArray {
    @weakify(self);
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        @strongify(self);
        //创建并行队列
        dispatch_queue_t fetchFeedQueue = dispatch_queue_create("com.starming.fetchfeed.fetchfeed", DISPATCH_QUEUE_CONCURRENT);
        dispatch_group_t group = dispatch_group_create();
        self.feeds = modelArray;
        
        for (int i = 0; i < modelArray.count; i++) {
            dispatch_group_enter(group);
            SMFeedModel *feedModel = modelArray[i];
            feedModel.isSync = NO;
            

            [self GET:feedModel.feedUrl parameters:nil headers:nil progress:nil success:^(NSURLSessionTask *task, id responseObject) {
                //                    NSString *xmlString = [[NSString alloc] initWithData:responseObject encoding:NSUTF8StringEncoding];
                //                    NSLog(@"Data: %@", xmlString);
                //                    NSLog(@"%@",feedModel);
                dispatch_async(fetchFeedQueue, ^{
                    @strongify(self);
                    //解析feed
                    self.feeds[i] = [self.feedStore updateFeedModelWithData:responseObject preModel:feedModel];
                    //入库存储
                    SMDB *db = [SMDB shareInstance];
                    @weakify(db);
                    [[db insertWithFeedModel:self.feeds[i]] subscribeNext:^(NSNumber *x) {
                        @strongify(db);
                        SMFeedModel *model = (SMFeedModel *)self.feeds[i];
                        model.fid = [x integerValue];
                        if (model.imageUrl.length > 0) {
                            NSString *fidStr = [x stringValue];
                            db.feedIcons[fidStr] = model.imageUrl;
                        }
                        //插入本地数据库成功后开始sendNext
                        [subscriber sendNext:@(i)];
                        //通知单个完成
                        dispatch_group_leave(group);
                    }];

                });//end dispatch async


            } failure:^(NSURLSessionTask *operation, NSError *error) {
                NSLog(@"Error: %@", error);
                dispatch_async(fetchFeedQueue, ^{
                    @strongify(self);
                    [[[SMDB shareInstance] insertWithFeedModel:self.feeds[i]] subscribeNext:^(NSNumber *x) {
                        SMFeedModel *model = (SMFeedModel *)self.feeds[i];
                        model.fid = [x integerValue];
                        dispatch_group_leave(group);
                    }];

                });//end dispatch async

            }];
            
        }//end for
        //全完成后执行事件
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            [subscriber sendCompleted];
        });
        return nil;
    }];
}
```


* Html convert to Core Text
使用了`DTCoreText`这个框架

* feed 详情页


* FMDB 本地存储 feed
* Atom 解析
* 阅读原文
* feed 列表的样式调整
* 首页 fetch 中的效果
* 支持点击文章点击后记录已看，显示已看的效果
* 显示列表滚动条
* 读取 feed 列表时取未读文章
* 支持标记全部已读，右上角添加按钮。同时清除该源下文章
* 首页提供一个 fetch 中的进度条
* 支持系统分享，长按链接，和点击右上角分享按钮
* 内置 web 浏览器
* 可判断 4g 和 wifi 环境，wifi 下可串行下载离线浏览图片

## Todo List
* 支持手动添加源

## 截图

![](https://github.com/ming1016/GCDFetchFeed/blob/master/GCDFetchFeed/resource/ScreenShot1.png?raw=true)|![](https://github.com/ming1016/GCDFetchFeed/blob/master/GCDFetchFeed/resource/ScreenShot2.png?raw=true)|![](https://github.com/ming1016/GCDFetchFeed/blob/master/GCDFetchFeed/resource/ScreenShot3.png?raw=true)  
:-------------------------:|:-------------------------:|:-------------------------:

## 评价
这个库整体上可学习的内容不是很多，主要使用了常见的三方框架：
* RAC
* FMDB
* DTCoreText
