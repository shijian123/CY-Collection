## SDWebImage介绍

SDWebImage是iOS开发中十分流行的库，大多数的开发者在下载图片或者加载网络图片并且本地缓存的时候，都会用这个框架。官方给出的定义为：

```
//一个异步下载图片并且支持缓存的 UIImageView 分类
Asynchronous image downloader with cache support as a UIImageView category 
```
官方也给出了一个主调用流程图，如下：
![MainSequenceDiagram.png](https://user-gold-cdn.xitu.io/2019/5/28/16afd85545265d24?w=1240&h=454&f=png&s=50765)

结合上图概括说下`SDWebImage下载图片`的调用流程：

1、分配流程：我们常用的方法`sd_setImageWithURL()`，它实质调用的是`sd_internalSetImageWithURL()`，在这个方法里面用`SDWebImageManager`调用`loadImageWithURL(url, options, context, progressBlock, completedBlock)`函数,在`loadImageWithURL()`里则又先判断url格式是否正确，不正确直接调用`callCompletionBlockForOperation()`返回错误信息，正确进入下一步从缓存中寻找图片，调用`callCacheProcessForOperation`;

2、缓存流程：在`callCacheProcessForOperation `判断用户是否设置了仅下载，如果设置了直接进入到`callDownloadProcessForOperation`开始下载流程，如果没设置则先调用`queryImageForKey:(key, options, context, completionBlock)`适配`SDImageCacheOptions`，然后再调用`queryCacheOperationForKey`开始缓存查找，首先在内存中查找，如果有则返回；没有则在磁盘中查找，如果都没有查找到则进入下载流程

3、下载流程：其实`callDownloadProcessForOperation `函数的完整版是`callDownloadProcessForOperation(operation, url, options, context, cachedImage, cachedData, cacheType, progressBlock, completedBlock)`,通过参数我们也能知道其实这个方法里面有了上一步缓存查找的结果，如果上一步缓存查找到了，则调用`callCompletionBlockForOperation`返回结果，如果没有则还是先调用`loadImageWithURL:(url, options, context, progressBlock, completedBlock)`适配`SDWebImageDownloaderOptions`,然后调用`downloadImageWithURL`进行下载并生成一个`SDWebImageDownloadToken`，这个token可以用来取消下载操作

4、存储缓存流程：在下载完成后，调用`callStoreCacheProcessForOperation`开始存储缓存流程，在这个函数里调用`storeImage:(image, imageData, key, cacheType, completionBlock)`设置缓存的Data、key等参数，将图片存储到相应的地方，并调用`callCompletionBlockForOperation`函数返回结果


下面我们就具体看下以上流程所涉及函数的代码实现，代码里面有注释：

## 1、分配流程：
让我们从这个我们最常用的方法开始

```
[self.imgView sd_setImageWithURL:[NSURL URLWithString:@"url"] placeholderImage:[UIImage imageNamed:@"placeholderPic"]];
```

点击进入`UIImageView+WebCache.m`文件，会发现实际调用的是“另有其码”

```
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:0 progress:nil completed:nil];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options progress:(nullable SDImageLoaderProgressBlock)progressBlock completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:options context:nil progress:progressBlock completed:completedBlock];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
                   context:(nullable SDWebImageContext *)context
                  progress:(nullable SDImageLoaderProgressBlock)progressBlock
                 completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_internalSetImageWithURL:url
                    placeholderImage:placeholder
                             options:options
                             context:context
                       setImageBlock:nil
                            progress:progressBlock
                           completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL) {
                               if (completedBlock) {
                                   completedBlock(image, error, cacheType, imageURL);
                               }
                           }];
}

```

然后我们在`UIView+WebCache.m`文件里看看`sd_internalSetImageWithURL()：`

```
/**
	使用图片的 URL 和可选的 placeholder image 设置 imageView 的 image 
	
	下载是异步和缓存的
	
	@param url image的URL
	@param placeholder 初始图像，直至image数据请求完成
	@param options image下载的时候的选择项-SDWebImageOptions，下方有详细介绍
	@param context 为了补充options枚举没有的选择想，下方有详细介绍
	@param setImageBlock 用于自定义设置image
	@param progressBlock 在图片下载ing状态调用，注：在后台队列上执行
	@param completedBlock 在操作完成后调用，返回类型为
	completedBlock(image, data, error, cacheType, finished, url)
	
*/

- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock {
    //将可变对象copy为不可变的
    context = [context copy]; // copy to avoid mutable object
    
    //取出context设置中的operationKey
    NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
    //如果operationKey为nil，就采用他自身的类的字符串作为key
    if (!validOperationKey) {
        validOperationKey = NSStringFromClass([self class]);
    }
    //取消之前绑定的operation，保证没有当前正在进行的异步下载操作, 使它不会与即将进行的操作发生冲突
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    //为自身绑定一个URL
    self.sd_imageURL = url;
    //如果不是延迟显示placeholder的情况
    if (!(options & SDWebImageDelayPlaceholder)) {
    	 //dispatch_main_async_safe 异步线程安全，下面有源码介绍
        dispatch_main_async_safe(^{
        	 //在图片下载下来之前，添加临时的占位图
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:SDImageCacheTypeNone imageURL:url];
        });
    }
    
    if (url) {
        // reset the progress
        self.sd_imageProgress.totalUnitCount = 0;
        self.sd_imageProgress.completedUnitCount = 0;
        
#if SD_UIKIT || SD_MAC
        // check and start image indicator
        // 检查是否设置了image indicator，如果有则启动，下面有方法的源码
        [self sd_startImageIndicator];
        id<SDWebImageIndicator> imageIndicator = self.sd_imageIndicator;
#endif
        // 判断context中是否自定义了manager，如果没有则使用默认的
        SDWebImageManager *manager = context[SDWebImageContextCustomManager];
        if (!manager) {
            manager = [SDWebImageManager sharedManager];
        }
        
        // 设置image加载进度Block（已接收size，预计总size，image的URL）
        __weak __typeof(self)wself = self;
        SDImageLoaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
        	// 使用__strong __typeof是防止self在这个执行过程中释放，下方有详细介绍
            __strong __typeof (wself) sself = wself;
            NSProgress *imageProgress = sself.sd_imageProgress;
            imageProgress.totalUnitCount = expectedSize;
            imageProgress.completedUnitCount = receivedSize;
#if SD_UIKIT || SD_MAC
				//加载指示器是否实现了updateIndicatorProgress方法
            if ([imageIndicator respondsToSelector:@selector(updateIndicatorProgress:)]) {
                double progress = imageProgress.fractionCompleted;
                dispatch_async(dispatch_get_main_queue(), ^{
                    [imageIndicator updateIndicatorProgress:progress];
                });
            }
#endif			//返回image加载进度Block
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
        //下载图片
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options context:context progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            __strong __typeof (wself) sself = wself;
            if (!sself) { return; }
           
            // 如果progress没有更新，则标记其为完成状态
            if (finished && !error && sself.sd_imageProgress.totalUnitCount == 0 && sself.sd_imageProgress.completedUnitCount == 0) {
            //const int64_t SDWebImageProgressUnitCountUnknown = 1LL; （LL是 long long 类型的缩写）
                sself.sd_imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                sself.sd_imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
            }
            
#if SD_UIKIT || SD_MAC
            // 检查并停止 image indicator
            if (finished) {
                [self sd_stopImageIndicator];
            }
#endif
            // 下载完成后是否自动加载图片 (options & SDWebImageAvoidAutoSetImage)根据枚举名取枚举中的值，get了
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!sself) { return; }
                if (!shouldNotSetImage) {
                    [sself sd_setNeedsLayout];
                }
                // 设置completedBlock
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, data, error, cacheType, finished, url);
                }
            };
            
            // case 1a: we got an image, but the SDWebImageAvoidAutoSetImage flag is set
            // OR
            // case 1b: we got no image and the SDWebImageDelayPlaceholder is not set
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }
            
            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                targetImage = placeholder;
                targetData = nil;
            }
            
#if SD_UIKIT || SD_MAC
            // check whether we should use the image transition
            // 检查image的过渡动画效果
            SDWebImageTransition *transition = nil;
            if (finished && (options & SDWebImageForceTransition || cacheType == SDImageCacheTypeNone)) {
                transition = sself.sd_imageTransition;
            }
#endif
				// 设置image的过渡动画效果
            dispatch_main_async_safe(^{
#if SD_UIKIT || SD_MAC
                [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
#else
                [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:cacheType imageURL:imageURL];
#endif
                callCompletedBlockClojure();
            });
        }];
        //在操作缓存字典（operationDictionary）里添加operation，表示当前的操作正在进行，源码见下
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } else {
    		// 没有url，停止Image Indicator
#if SD_UIKIT || SD_MAC
        [self sd_stopImageIndicator];
#endif
        dispatch_main_async_safe(^{
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
                completedBlock(nil, nil, error, SDImageCacheTypeNone, YES, url);
            }
        });
    }
}

```		


PS 

### options参数：

SDWebImageOptions属性	|	说明
----------------------	| ------------------------
SDWebImageRetryFailed	|	默认情况下，如果一个url在下载的时候失败了,那么这个url会被加入黑名单并且library不会尝试再次下载,这个flag会阻止library把失败的url加入黑名单	
SDWebImageLowPriority	|	默认情况下，图片会在交互发生的时候下载(例如你滑动tableview的时候),这个flag会禁止这个特性,导致的结果就是在scrollview减速的时候,才会开始下载
SDWebImageProgressiveLoad	|	这个flag启动渐进式下载图像，类似浏览器加载图像那样逐步显示，默认情况下，图像仅是在下载完成后显示	
SDWebImageRefreshCached	|	一个图片缓存了，还是会重新请求.并且缓存侧略依据NSURLCache而不是SDWebImage。即在URL没变但是服务器图片发生更新时使用	
SDWebImageContinueInBackground	|	启动后台下载，实现原理是通过向系统询问后台的额外时间来完成请求的。 如果后台任务到期，则操作将被取消。	
SDWebImageHandleCookies	|	当设置了`NSMutableURLRequest.HTTPShouldHandleCookies = YES`时，可以控制存储NSHTTPCookieStorage中的cookie
SDWebImageAllowInvalidSSLCertificates	| 允许不安全的SSL证书，用于测试环境，在正式环境中谨慎使用	
SDWebImageHighPriority	|	默认情况下,image在装载的时候是按照他们在队列中的顺序装载的(就是先进先出)。这个flag会把他们移动到队列的前端,并且立刻装载,而不是等到当前队列装载的时候再装载。	
SDWebImageDelayPlaceholder	|	默认情况下,占位图会在图片下载的时候显示.这个flag开启会延迟占位图显示的时间,等到图片下载完成之后才会显示占位图.	
SDWebImageTransformAnimatedImage	|	通常不会在可动画的图像上调用 `transformDownloadedImage` 代理方法，因为大多数转换代码会破坏动画文件，这个flag为尝试转换	
SDWebImageAvoidAutoSetImage	|	图片在下载后被加载到imageView。但是在一些情况下，我们想要设置一下图片（引用一个滤镜或者加入透入动画）这个flag来手动的设置图片在下载图片成功后
SDWebImageScaleDownLargeImages	|	默认情况下，图像将根据其原始大小进行解码。 在iOS上，此flat会将图片缩小到与设备的受限内存兼容的大小。	但如果设置了`SDWebImageAvoidDecodeImage`则此flat不起作用。 如果设置了`SDWebImageProgressiveLoad`它将被忽略。
SDWebImageQueryMemoryData	|	默认情况下，当图像已缓存在内存中时，我们不会查询图像数据。 此flat则强制查询图像数据。 此查询是异步的，除非指定`SDWebImageQueryMemoryDataSync`
SDWebImageQueryMemoryDataSync	|	结合`SDWebImageQueryMemoryData`设置同步查询图像数据（一般不建议这么使用，除非是在同一个`runloop`里避免单元格复用时发生闪现）
SDWebImageQueryDiskDataSync	|	如果内存查询没有的时候，强制同步磁盘查询（这三个查询可以组合使用，一般不建议这么使用，除非是在同一个`runloop`里避免单元格复用时发生闪现）
SDWebImageFromCacheOnly		|	默认情况下，当缓存丢失时，SD将从网络下载图像。 此flat可以防止这样，使其仅从缓存加载。
SDWebImageFromLoaderOnly	|	默认情况下，SD在下载之前先从缓存中查找，此flat可以防止这样，使其仅从网络下载
SDWebImageForceTransition	|  默认情况下，SD在图像加载完成后使用`SDWebImageTransition`进行某些视图转换，此转换仅适用于从网络下载图像。 此flat可以强制为内存和磁盘缓存应用视图转换。
SDWebImageAvoidDecodeImage	| 默认情况下，SD在查询缓存和从网络下载时会在后台解码图像，这有助于提高性能，因为在屏幕上渲染图像时，需要首先对其进行解码。这发生在`Core Animation`的主队列中。然而此过程也可能会增加内存使用量。 如果由于过多的内存消耗而遇到问题，可以用此flat禁止解码图像。
SDWebImageDecodeFirstFrameOnly	|  默认情况下，SD会解码动画图像，该flat强制只解码第一帧并生成静态图。
SDWebImagePreloadAllFrames	| 默认情况下，对于`SDAnimatedImage`，SD会在渲染过程中解码动画图像帧以减少内存使用量。 但是用户可以指定将所有帧预加载到内存中，以便在大量imageView共享动画图像时降低CPU使用率。这实际上会在后台队列中触发`preloadAllAnimatedImageFrames`（仅限磁盘缓存和下载）。

### context参数：


```
typedef NSDictionary<SDWebImageContextOption, id> SDWebImageContext;
typedef NSMutableDictionary<SDWebImageContextOption, id> SDWebImageMutableContext;

```

可以看到`SDWebImageContext / SDWebImageMutableContext `其实就是以 `SDWebImageContextOption`为`key`,`id`(指定类型或者协议)为`value` 的`NSDictionary/NSMutableDictionary`

而 `SDWebImageContextOption `是一个可扩展的String枚举
```
typedef NSString * SDWebImageContextOption NS_EXTENSIBLE_STRING_ENUM;
```

SDWebImage定义的10个SDWebImageContextOption的如下

Key	|	 Value	         |         说明
----------------------	| ---------------------  | ----------------------
SDWebImageContextSetImageOperationKey | NSString | 作为view类别的 operation key使用，用来存储图像下载的operation，用于支持不同图像加载过程的视图实例。如果为nil，则使用类名作为操作键。
SDWebImageContextCustomManager | SDWebImageManager * | 可以传入一个自定义的`SDWebImageManager`，默认使用`[SDWebImageManager sharedManager]`	
SDWebImageContextImageTransformer | id<SDImageTransformer> | 可以传入一个id<SDImageTransformer>类型（👇有详细介绍）用于转换处理加载出来的图片，并将变换后的图像存储到缓存中。如果设置了，则会忽略`manager `中的`transformer`
SDWebImageContextImageScaleFactor | NSNumber | CGFloat原始值，为用于指定图像比例且这个数值应大于等于1.0 
SDWebImageContextStoreCacheType | NSNumber | `SDImageCacheType`原始值，用于刚刚下载图像时指定缓存类型，并将其存储到缓存中。 指定`SDImageCacheTypeNone`：禁用缓存存储; `SDImageCacheTypeDisk`：仅存储在磁盘缓存中; `SDImageCacheTypeMemory`：只存储在内存中；`SDImageCacheTypeAll`：存储在内存缓存和磁盘缓存中。如果没有提供或值无效，则使用`SDImageCacheTypeAll`。	
SDWebImageContextAnimatedImageClass | Class（`UIImage / NSImage`的子类并采用了`SDAnimatedImage`协议 ） | 用于使用`SDAnimatedImageView`来改善动画图像渲染性能（尤其是大动画图像上的内存使用）。	
SDWebImageContextDownloadRequestModifier | id<SDWebImageDownloaderRequestModifier> | 用于在加载图片前修改`NSURLRequest`，	
SDWebImageContextCacheKeyFilter | id<SDWebImageCacheKeyFilter> | 指定图片的缓存key	
SDWebImageContextCacheSerializer | id<SDWebImageCacheSerializer> | 转换需要缓存的图片格式，通常用于需要缓存的图片格式与下载的图片格式不相符的时候，如：下载的时候为了节约流量、减少下载时间使用了WebP格式，但是如果缓存也用WebP，每次从缓存中取图片都需要经过一次解压缩，这样是比较影响性能的，就可以使用id<SDWebImageCacheSerializer>



### SDImageTransformer的类型

SDImageTransformer的类型				|    作用
--------------------------------	| --------------------- 
SDImagePipelineTransformer 		|  可以传入一个NSArray<id<SDImageTransformer>>按顺序做转换
SDImageRoundCornerTransformer 	|  添加圆角
SDImageResizingTransformer 		|  调整大小
SDImageCroppingTransformer 		|  裁剪
SDImageFlippingTransformer 		|  翻转
SDImageRotationTransformer 		|  旋转
SDImageTintTransformer		 		|  添加色彩
SDImageBlurTransformer 				|  添加模糊
SDImageFilterTransformer	 		|  添加滤镜





### dispatch_main_async_safe：

用于异步线程安全，源码如下 

```
#ifndef dispatch_main_async_safe
#define dispatch_main_async_safe(block)\
    if (dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL) == dispatch_queue_get_label(dispatch_get_main_queue())) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
#endif
```

`dispatch_queue_get_label`用来取队列的名字，进而判断如果当前已经是主队列，那么直接执行，否则回调到主队列之后再执行。



### sd_startImageIndicator函数

```
- (void)sd_startImageIndicator {
    id<SDWebImageIndicator> imageIndicator = self.sd_imageIndicator;
    if (!imageIndicator) {
        return;
    }
    dispatch_main_async_safe(^{
        [imageIndicator startAnimatingIndicator];
    });
}
```

### 在Block中使用__strong __的原因

```
        __weak __typeof(self)wself = self;
        SDImageLoaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            __strong __typeof (wself) sself = wself;
		... 省略代码
        };
```

`weakSelf `是为了`block`不持有`self`，避免`Retain Circle`循环引用。在 `Block` 内如果需要访问 `self` 的方法、变量，建议使用 `weakSelf`。`strongSelf`的目的是因为一旦进入`block`执行，假设不允许`self`在这个执行过程中释放，就需要加入`strongSelf`。`block`执行完后这个`strongSelf` 会自动释放，没有不会存在循环引用问题。如果在 `Block` 内需要多次 访问` self`，则需要使用 `strongSelf`。

### sd_setImageLoadOperation函数

```
- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key {
    if (key) {
        [self sd_cancelImageLoadOperationWithKey:key];
        if (operation) {
        		// 字典 operationDictionary 专门用作存储操作的缓存，随时添加、删除操作任务
            SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
            @synchronized (self) {
                [operationDictionary setObject:operation forKey:key];
            }
        }
    }
}
```


再进入`SDWebImageManager.m`文件，查看`loadImageWithURL(url, options, context, progressBlock, completedBlock)`


```
- (SDWebImageCombinedOperation *)loadImageWithURL:(nullable NSURL *)url
                                          options:(SDWebImageOptions)options
                                          context:(nullable SDWebImageContext *)context
                                         progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                        completed:(nonnull SDInternalCompletionBlock)completedBlock {
   
    //首先设置断言completedBlock不为空，否则这个方法也就没有了意义
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    // 处理部分使用者传递过来url不是NSURL类型而是NSString类型，且Xcode不对类型不匹配发出警告
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // 防止因参数错误造成的app崩溃
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

	//用这个Operation管理下载和缓存的Operation，并把它加到runningOperations中
    SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    operation.manager = self;

    BOOL isFailedUrl = NO;
    if (url) {
    	//dispatch_semaphore_t failedURLsLock; 一个锁,用来保证线程安全的访问“failedURLs”
    	/* 上锁，开锁
    	#ifndef SD_LOCK
		#define SD_LOCK(lock) dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
		#endif

		#ifndef SD_UNLOCK
		#define SD_UNLOCK(lock) dispatch_semaphore_signal(lock);
		#endif
    	*/
    	
        SD_LOCK(self.failedURLsLock);
        // url是否在不成功的url列表中
        isFailedUrl = [self.failedURLs containsObject:url];
        SD_UNLOCK(self.failedURLsLock);
    }
	// url为nil 或者 在url失败黑名单里，则直接返回错误结果
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}] url:url];
        return operation;
    }
	// url正常则将Operation加到runningOperations中
    SD_LOCK(self.runningOperationsLock);
    [self.runningOperations addObject:operation];
    SD_UNLOCK(self.runningOperationsLock);
    
    // Preprocess the context arg to provide the default value from manager
    context = [self processedContextWithContext:context];
    
    // 开始从缓存中查找、加载image
    [self callCacheProcessForOperation:operation url:url options:options context:context progress:progressBlock completed:completedBlock];

    return operation;
}

```

## 2、进入查找缓存流程

`SDWebImageManager`管理`下载operation`和`缓存operation`，首先manage分配当前operation应该为`下载operation`还是`缓存operation`

```
// Query cache process
- (void)callCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                 url:(nullable NSURL *)url
                             options:(SDWebImageOptions)options
                             context:(nullable SDWebImageContext *)context
                            progress:(nullable SDImageLoaderProgressBlock)progressBlock
                           completed:(nullable SDInternalCompletionBlock)completedBlock {

    // 检查是否设置了不从缓存中获取
    BOOL shouldQueryCache = (options & SDWebImageFromLoaderOnly) == 0;
    if (shouldQueryCache) {// 从缓存中查找
    	
    		// 获取图片缓存的key
        id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
        NSString *key = [self cacheKeyForURL:url cacheKeyFilter:cacheKeyFilter];
        __weak SDWebImageCombinedOperation *weakOperation = operation;
        // 在SDImageCache里查询是否存在缓存的图片
        operation.cacheOperation = [self.imageCache queryImageForKey:key options:options context:context completion:^(UIImage * _Nullable cachedImage, NSData * _Nullable cachedData, SDImageCacheType cacheType) {
            __strong __typeof(weakOperation) strongOperation = weakOperation;
            // 如果Operation不存在或者取消，安全的从runningOperations中删除它
            if (!strongOperation || strongOperation.isCancelled) {
                [self safelyRemoveOperationFromRunning:strongOperation];
                return;
            }
            // 进入到下载环节
            [self callDownloadProcessForOperation:strongOperation url:url options:options context:context cachedImage:cachedImage cachedData:cachedData cacheType:cacheType progress:progressBlock completed:completedBlock];
        }];
    } else {
    	  // 进入下载环节
        [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
    }
}

```

在`SDImageCache.m`文件里面，查看缓存operation的具体查找过程

```
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options context:(nullable SDWebImageContext *)context done:(nullable SDImageCacheQueryCompletionBlock)doneBlock {
    if (!key) {
        if (doneBlock) {
            doneBlock(nil, nil, SDImageCacheTypeNone);
        }
        return nil;
    }
    //是否存在图片转换处理设置
    id<SDImageTransformer> transformer = context[SDWebImageContextImageTransformer];
    if (transformer) {
        // grab the transformed disk image if transformer provided
        //如果存在转换处理，则从抓取转换后的磁盘图像
        NSString *transformerKey = [transformer transformerKey];
        key = SDTransformedKeyForKey(key, transformerKey);
    }
    
    // 第一步在内存cache中查找
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryMemoryData));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    // 第二步在磁盘cache中查找
    NSOperation *operation = [NSOperation new];
    // Check whether we need to synchronously query disk
    // 1. in-memory cache hit & memoryDataSync
    // 2. in-memory cache miss & diskDataSync
    BOOL shouldQueryDiskSync = ((image && options & SDImageCacheQueryMemoryDataSync) ||
                                (!image && options & SDImageCacheQueryDiskDataSync));
    void(^queryDiskBlock)(void) =  ^{
        if (operation.isCancelled) {
            // do not call the completion if cancelled
            return;
        }
        
        @autoreleasepool {
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage;
            SDImageCacheType cacheType = SDImageCacheTypeNone;
            if (image) {
                // the image is from in-memory cache, but need image data
                diskImage = image;
                cacheType = SDImageCacheTypeMemory;
            } else if (diskData) {
                cacheType = SDImageCacheTypeDisk;
                // decode image data only if in-memory cache missed
                diskImage = [self diskImageForKey:key data:diskData options:options context:context];
                if (diskImage && self.config.shouldCacheImagesInMemory) {
                    NSUInteger cost = diskImage.sd_memoryCost;
                    [self.memCache setObject:diskImage forKey:key cost:cost];
                }
            }
            
            if (doneBlock) {
                if (shouldQueryDiskSync) {
                    doneBlock(diskImage, diskData, cacheType);
                } else {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        doneBlock(diskImage, diskData, cacheType);
                    });
                }
            }
        }
    };
    
    // Query in ioQueue to keep IO-safe
    if (shouldQueryDiskSync) {
        dispatch_sync(self.ioQueue, queryDiskBlock);
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}

```


## 3、进入下载流程

`SDWebImageManager`管理的`下载operation`

```
// Download process
- (void)callDownloadProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                    url:(nullable NSURL *)url
                                options:(SDWebImageOptions)options
                                context:(SDWebImageContext *)context
                            cachedImage:(nullable UIImage *)cachedImage
                             cachedData:(nullable NSData *)cachedData
                              cacheType:(SDImageCacheType)cacheType
                               progress:(nullable SDImageLoaderProgressBlock)progressBlock
                              completed:(nullable SDInternalCompletionBlock)completedBlock {
    // Check whether we should download image from network
    // 检查是否应该从网络下载
    BOOL shouldDownload = (options & SDWebImageFromCacheOnly) == 0;
    shouldDownload &= (!cachedImage || options & SDWebImageRefreshCached);
    shouldDownload &= (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
    shouldDownload &= [self.imageLoader canLoadWithURL:url];
    // 如果图像应该下载
    if (shouldDownload) {
    		// 存在缓存图片 && 即使有缓存图片也要下载更新图片
        if (cachedImage && options & SDWebImageRefreshCached) {
            // 如果在缓存中找到了图像但设置了SDWebImageRefreshCached，则通知缓存图像尝试重新下载，以便从服务器刷新缓存。
            [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            // 将缓存的图像传递给图像加载器。 图像加载器则检查远程图像和缓存图像是否相等
            SDWebImageMutableContext *mutableContext;
            if (context) {
                mutableContext = [context mutableCopy];
            } else {
                mutableContext = [NSMutableDictionary dictionary];
            }
            mutableContext[SDWebImageContextLoaderCachedImage] = cachedImage;
            context = [mutableContext copy];
        }
        
        // `SDWebImageCombinedOperation` -> `SDWebImageDownloadToken` -> `downloadOperationCancelToken`, 这是一个`SDCallbacksDictionary` 并且需要在保留完整的block, 所以我们需要将weak再次strong避免被提前释放
        __weak typeof(operation) weakOperation = operation;
        operation.loaderOperation = [self.imageLoader loadImageWithURL:url options:options context:context progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
            __strong typeof(weakOperation) strongOperation = weakOperation;
            if (!strongOperation || strongOperation.isCancelled) {
                // 如果为取消操作，则不执行任何操作
                // 如果我们调用了这个completedBlock，那么这个Block会跟同一个对象的其他completedBlock产生竞争，所以如果这个Block是second，我们就用新数据覆盖
                // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
            } else if (cachedImage && options & SDWebImageRefreshCached && [error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCacheNotModified) {
                // 图像刷新在NSURLCache缓存中，则不调用这个完成的Block
            } else if (error) {// 下载报错
                [self callCompletionBlockForOperation:strongOperation completion:completedBlock error:error url:url];
                BOOL shouldBlockFailedURL = [self shouldBlockFailedURLWithURL:url error:error];
                // 加入url失败黑名单
                if (shouldBlockFailedURL) {
                    SD_LOCK(self.failedURLsLock);
                    [self.failedURLs addObject:url];
                    SD_UNLOCK(self.failedURLsLock);
                }
            } else {// 下载成功
            		// 失败重新加载成功，则将url移除失败黑名单
                if ((options & SDWebImageRetryFailed)) {
                    SD_LOCK(self.failedURLsLock);
                    [self.failedURLs removeObject:url];
                    SD_UNLOCK(self.failedURLsLock);
                }
                // 将下载完成的图像保存到
                [self callStoreCacheProcessForOperation:strongOperation url:url options:options context:context downloadedImage:downloadedImage downloadedData:downloadedData finished:finished progress:progressBlock completed:completedBlock];
            }
            // 完成后将strongOperation从runningOperations中移除
            if (finished) {
                [self safelyRemoveOperationFromRunning:strongOperation];
            }
        }];
    } else if (cachedImage) {
    		// 如果有缓存图像，则返回cachedImage
        [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
        [self safelyRemoveOperationFromRunning:operation];
    } else {
        
        // 图像不在缓存中且delegate不允许下载则返回nil
        [self callCompletionBlockForOperation:operation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
        [self safelyRemoveOperationFromRunning:operation];
    }
}

```

在`SDWebImageDownloader.m`文件查看具体的下载代码

1、将`SDWebImageOptions`转换为`SDWebImageDownloaderOptions`后开始下载图片

```
- (id<SDWebImageOperation>)loadImageWithURL:(NSURL *)url options:(SDWebImageOptions)options context:(SDWebImageContext *)context progress:(SDImageLoaderProgressBlock)progressBlock completed:(SDImageLoaderCompletedBlock)completedBlock {
    UIImage *cachedImage = context[SDWebImageContextLoaderCachedImage];
    
    SDWebImageDownloaderOptions downloaderOptions = 0;
    if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
    if (options & SDWebImageProgressiveLoad) downloaderOptions |= SDWebImageDownloaderProgressiveLoad;
    if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
    if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
    if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
    if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
    if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
    if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
    if (options & SDWebImageAvoidDecodeImage) downloaderOptions |= SDWebImageDownloaderAvoidDecodeImage;
    if (options & SDWebImageDecodeFirstFrameOnly) downloaderOptions |= SDWebImageDownloaderDecodeFirstFrameOnly;
    if (options & SDWebImagePreloadAllFrames) downloaderOptions |= SDWebImageDownloaderPreloadAllFrames;
    
    if (cachedImage && options & SDWebImageRefreshCached) {
        // force progressive off if image already cached but forced refreshing
        // 如果图像已缓存却要求强制刷新，则强制关闭SDWebImageDownloaderProgressiveLoad
        downloaderOptions &= ~SDWebImageDownloaderProgressiveLoad;
        // 如果图像缓存但强制刷新，则忽略从NSURLCache读取图像
        downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
    }
    // 开始下载图像
    return [self downloadImageWithURL:url options:downloaderOptions context:context progress:progressBlock completed:completedBlock];
}

```

下载图片的实际代码，`SDWebImageDownloadToken`可以用来取消下载任务

```
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                   context:(nullable SDWebImageContext *)context
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    // 异常处理，url不能为nil否则立即调用completedBlock
    if (url == nil) {
        if (completedBlock) {
            NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
            completedBlock(nil, nil, error, YES);
        }
        return nil;
    }
    
    SD_LOCK(self.operationsLock);
    // 根据url去查找SDWebImageDownloaderOperation，如果没有，则调用createCallback block去生成downloadOperation，之后实现operation 完成操作的回调block，然后根据operation生成token。
    NSOperation<SDWebImageDownloaderOperation> *operation = [self.URLOperations objectForKey:url];
    
    // 有一种情况operation可能被标记为已完成或已取消，但未从“self.URLOperations”中删除
    if (!operation || operation.isFinished || operation.isCancelled) {
        operation = [self createDownloaderOperationWithUrl:url options:options context:context];
        if (!operation) {
            SD_UNLOCK(self.operationsLock);
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidDownloadOperation userInfo:@{NSLocalizedDescriptionKey : @"Downloader operation is nil"}];
                completedBlock(nil, nil, error, YES);
            }
            return nil;
        }
        __weak typeof(self) wself = self;
        operation.completionBlock = ^{
            __strong typeof(wself) sself = wself;
            if (!sself) {
                return;
            }
            SD_LOCK(sself.operationsLock);
            [sself.URLOperations removeObjectForKey:url];
            SD_UNLOCK(sself.operationsLock);
        };
        self.URLOperations[url] = operation;
        
        // 根据苹果文档完成所有的配置后将operation添加进operation queue
        // “addOperation”不是同步操作“operation.completionBlock”，所以不会造成死锁
        [self.downloadQueue addOperation:operation];
    }
    else if (!operation.isExecuting) { // 若operation没有执行
        if (options & SDWebImageDownloaderHighPriority) {
            operation.queuePriority = NSOperationQueuePriorityHigh;
        } else if (options & SDWebImageDownloaderLowPriority) {
            operation.queuePriority = NSOperationQueuePriorityLow;
        } else {
            operation.queuePriority = NSOperationQueuePriorityNormal;
        }
    }
    SD_UNLOCK(self.operationsLock);

    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    
    // 这个token关联每一个下载，可以用它来取消下载
    SDWebImageDownloadToken *token = [[SDWebImageDownloadToken alloc] initWithDownloadOperation:operation];
    token.url = url;
    token.request = operation.request;
    token.downloadOperationCancelToken = downloadOperationCancelToken;
    token.downloader = self;
    
    return token;
}

```

## 4、存储缓存流程

根据上面`SDWebImageManager.m`中的`callDownloadProcessForOperation`我们知道当下载完成时再次调用`callStoreCacheProcessForOperation`进行缓存存储

```
// Store cache process
- (void)callStoreCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                      url:(nullable NSURL *)url
                                  options:(SDWebImageOptions)options
                                  context:(SDWebImageContext *)context
                          downloadedImage:(nullable UIImage *)downloadedImage
                           downloadedData:(nullable NSData *)downloadedData
                                 finished:(BOOL)finished
                                 progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                completed:(nullable SDInternalCompletionBlock)completedBlock {
    // 根据单例中的缓存配置看是否应该将图片缓存在内存
    SDImageCacheType storeCacheType = SDImageCacheTypeAll;
    if (context[SDWebImageContextStoreCacheType]) {
        storeCacheType = [context[SDWebImageContextStoreCacheType] integerValue];
    }
    id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
    NSString *key = [self cacheKeyForURL:url cacheKeyFilter:cacheKeyFilter];
    id<SDImageTransformer> transformer = context[SDWebImageContextImageTransformer];
    id<SDWebImageCacheSerializer> cacheSerializer = context[SDWebImageContextCacheSerializer];
    if (downloadedImage && (!downloadedImage.sd_isAnimated || (options & SDWebImageTransformAnimatedImage)) && transformer) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
            @autoreleasepool {
                UIImage *transformedImage = [transformer transformedImageWithImage:downloadedImage forKey:key];
                if (transformedImage && finished) {
                    NSString *transformerKey = [transformer transformerKey];
                    NSString *cacheKey = SDTransformedKeyForKey(key, transformerKey);
                    BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                    NSData *cacheData;
                    // image转换后则为nil，所以我们重新计算图像的大小
                    if (cacheSerializer && (storeCacheType == SDImageCacheTypeDisk || storeCacheType == SDImageCacheTypeAll)) {
                        cacheData = [cacheSerializer cacheDataWithImage:transformedImage  originalData:(imageWasTransformed ? nil : downloadedData) imageURL:url];
                    } else {
                        cacheData = (imageWasTransformed ? nil : downloadedData);
                    }
                    // 缓存转换后的图像
                    [self.imageCache storeImage:transformedImage imageData:cacheData forKey:cacheKey cacheType:storeCacheType completion:nil];
                }
                
                [self callCompletionBlockForOperation:operation completion:completedBlock image:transformedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
            }
        });
    } else {
        if (downloadedImage && finished) {
            if (cacheSerializer && (storeCacheType == SDImageCacheTypeDisk || storeCacheType == SDImageCacheTypeAll)) {
                dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                    @autoreleasepool {
                        NSData *cacheData = [cacheSerializer cacheDataWithImage:downloadedImage originalData:downloadedData imageURL:url];
                        // 在磁盘中缓存图像，使用cacheData
                        [self.imageCache storeImage:downloadedImage imageData:cacheData forKey:key cacheType:storeCacheType completion:nil];
                    }
                });
            } else {
            		// 缓存图像，使用downloadedData
                [self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key cacheType:storeCacheType completion:nil];
            }
        }
        [self callCompletionBlockForOperation:operation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
    }
}
```

在`SDImageCache.m`文件里，查看缓存图像的实际实现

```
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
          toMemory:(BOOL)toMemory
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    if (!image || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    // 根据单例中的缓存配置看是否应该将图片缓存在内存
    if (toMemory && self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = image.sd_memoryCost;
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    // 判断是否存入磁盘，是的话在串行的ioQueue中执行存入磁盘操作
    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            @autoreleasepool {
                NSData *data = imageData;
                // 如果没有将图片转为data，会根据是否是包含alpha channel来辨别是PNG OR JPEG。然后调用SDWebImageCodersManager来encode image to NSData。之后写入磁盘
                if (!data && image) {
                    SDImageFormat format;
                    if ([SDImageCoderHelper CGImageContainsAlpha:image.CGImage]) {
                        format = SDImageFormatPNG;
                    } else {
                        format = SDImageFormatJPEG;
                    }
                    data = [[SDImageCodersManager sharedManager] encodedDataWithImage:image format:format options:nil];
                }
                [self _storeImageDataToDisk:data forKey:key];
            }
            
            if (completionBlock) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionBlock();
                });
            }
        });
    } else {
        if (completionBlock) {
            completionBlock();
        }
    }
}

```

## 一些感悟

* 代码就像女孩，代码格式如果很整齐、有规律让人眼前一亮，就会给人一种很舒服的感觉，不仅为了别人，也是为了自己之后阅读代码的时候能有一种很愉快的心态

* 代码要简洁明了，核心类`SDWebImageManager`使用单例模式，以操作查询、下载、存储等功能，其具体功能实现分配到各个相应的类中，运用单一模式原则让各类各司其职，同时其也方便了独立测试和维护

* 在处理复杂的回调时，使用Block显得简单直接，不拐弯抹角，便于阅读，例如上面源码的层层completionBlock回调

* 在开始一个功能前，将一些必要的异常首先排除出去，例如
	```
	 if (url == nil) {
        if (completedBlock) {
            NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
            completedBlock(nil, nil, error, YES);
        }
        return nil;
    }
	```

* 如果涉及线程，则需时时注意要保证线程的安全。例如dispatch_main_async_safe

## 参考资料：

[官方-SDWebImage](https://github.com/SDWebImage/SDWebImage)

