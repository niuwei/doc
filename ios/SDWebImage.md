# SDWebImage

这个库提供了一个UIImageView的别类（category），以获取支持远端web图片。

特点：

* 为Cocoa Touch framework的UIImageView添加别类（category）以获取和缓存web图片。
* 异步下载图片。
* 异步内存+磁盘缓存图片，过期缓存自动处理。
* 支持GIF动画。
* 支持WebP格式。
* 后台图像解压缩
* 确保相同URL只会下载一次。
* 确保伪URL重试次数限制。
* 确保不阻塞主线程。
* 效率！
* 使用GCD和ARC。
* 支持Arm64.

[How is SDWebImage better than X?](https://github.com/rs/SDWebImage/wiki/How-is-SDWebImage-better-than-X%3F)

## 这个库的优点

### 自从iOS 5.0以来，NSURLCache处理磁盘缓存，想比普通（over plain）NSURLRequest，SDWebImage的优点在哪？

iOS NSURLCache做原生（row）HTTP response的内存和磁盘缓存。每次命中缓存，APP都要把原生缓存数据转化为UIImage。涉及到了HTTP数据解析和内存拷贝等操作。

另一方面，SDWebImage在内存上缓存UIImage数据，在磁盘上缓存压缩的UIImage数据。使用NSCache缓存的UIImage数据不需要调用copy直接使用缓存数据。

此外，在主线程的UIImageView第一次使用UIImage，由SDWebImageDecoder在后台线程负责图片解压缩。

最后但并非最不重要，SDWebImage完全绕过复杂和会频繁错配的HTTP缓存控制协商，查找缓存大大加速。

[AFNetworking link]:https://github.com/AFNetworking/AFNetworking

### 既然[AFNetworking][AFNetworking link]为UIImageView提供了相似的功能，SDWebImage还有用吗？

毫无质疑不会，AFNetworking得益于使用NSURLCache缓存的基础URL加载系统（Foundation URL Loading System），就像用NSCache为UIImageView和UIButton提供的可配置内存缓存。可以进一步在对应NSURLRequest的缓存策略上配置缓存行为。AFNetworking也提供了SDWebImage的后台解压图片特性。

如果你已经在用AFNetworking了，仅仅想要个轻松异步加载图片的分类，内置的UIKit扩展可以满足需求。

## 如何使用

### 在UITableView中使用

只要导入头文件UIImageView+WebCache.h, 在 tableView:cellForRowAtIndexPath: UITableViewDataSource 方法中调用sd_setImageWithURL:placeholderImage: 方法. 为您处理从异步下载到缓存管理的所有工作。

```objc
#import <SDWebImage/UIImageView+WebCache.h>

...

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    static NSString *MyIdentifier = @"MyIdentifier";

    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:MyIdentifier];
    if (cell == nil) {
        cell = [[[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault
                                       reuseIdentifier:MyIdentifier] autorelease];
    }

    // Here we use the new provided sd_setImageWithURL: method to load the web image
    [cell.imageView sd_setImageWithURL:[NSURL URLWithString:@"http://www.domain.com/path/to/image.jpg"]
                      placeholderImage:[UIImage imageNamed:@"placeholder.png"]];

    cell.textLabel.text = @"My Text";
    return cell;
}
```

### Using blocks

使用 blocks, 会通知你图片的下载进度和成功与否的信息:

```objc
// Here we use the new provided sd_setImageWithURL: method to load the web image
[cell.imageView sd_setImageWithURL:[NSURL URLWithString:@"http://www.domain.com/path/to/image.jpg"]
                      placeholderImage:[UIImage imageNamed:@"placeholder.png"]
                             completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
                                ... completion code here ...
                             }];
```

Note: 如果在完成之前取消请求，无论成功或失败block都不会被调用。

### Using SDWebImageManager

SDWebImageManager 连接异步下载与图像缓存。You can use this class directly to benefit from web image downloading with caching in another context than a UIView (ie: with Cocoa).

下面是一个如何使用 SDWebImageManager 的简单例子:

```obcj
SDWebImageManager *manager = [SDWebImageManager sharedManager];
[manager downloadImageWithURL:imageURL
                      options:0
                     progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                         // progression tracking code
                     }
                     completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
                         if (image) {
                             // do something with image
                         }
                     }];
```

### Using Asynchronous Image Downloader Independently

单独使用异步图像下载：

```objc
SDWebImageDownloader *downloader = [SDWebImageDownloader sharedDownloader];
[downloader downloadImageWithURL:imageURL
                         options:0
                        progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                            // progression tracking code
                        }
                       completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                            if (image && finished) {
                                // do something with image
                            }
                        }];
```

### Using Asynchronous Image Caching Independently

可以单独使用异步图像缓存存储。SDImageCache包含内存缓存和可选磁盘缓存。磁盘缓存写操作是异步的，不会让UI产生延迟。

方便起见，SDImageCache 类以单例方式提供，但如果想创建分离的缓存命名空间，也可以创建自己的实例。

使用 queryDiskCacheForKey:done: 方法搜索缓存。方法返回nil意味着图片不在缓存中。就由你负责生成和缓存它。图片在缓存中拥有唯一标识，通常是图片的绝对URL。

```objc
SDImageCache *imageCache = [[SDImageCache alloc] initWithNamespace:@"myNamespace"];
[imageCache queryDiskCacheForKey:myCacheKey done:^(UIImage *image) {
    // image is not nil if image was found
}];
```

通常，SDImageCache在搜索内存缓存失败后，会搜索磁盘缓存。方法imageFromMemoryCacheForKey: 可以阻止方法磁盘缓存。

在缓存中存储图片，使用 storeImage:forKey: 方法：

```objc
[[SDImageCache sharedImageCache] storeImage:myImage forKey:myCacheKey];
```

图片默认会存储到内存缓存和磁盘缓存。如果只想存储到内存，使用 storeImage:forKey:toDisk: 方法，并将第三个参数设置为NO。

### Using cache key filter

有时，你可能不想使用图像的URL作为缓存的key，因为部分的网址是动态的（即：用于访问控制的目的）。SDWebImageManager提供了缓存key的过滤器，输入NSURL，输出NSString形式的缓存key。

下面的例子给SDWebImageManager设置一个过滤器，从URL里移除查询字符串，作为缓存的key：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    SDWebImageManager.sharedManager.cacheKeyFilter = ^(NSURL *url) {
        url = [[NSURL alloc] initWithScheme:url.scheme host:url.host path:url.path];
        return [url absoluteString];
    };

    // Your app init code...
    return YES;
}
```

## Common Problems

### Using dynamic image size with UITableViewCell

通过给Cell设置第一个图片，UITableView便决定了图片的尺寸。如果远端图片和你预设的占位图片（placeholder image）尺寸不一致，会产生尺寸变换问题。下面的文章给你一个解决这个问题的方法：

[http://www.wrichards.com/blog/2011/11/sdwebimage-fixed-width-cell-images/](http://www.wrichards.com/blog/2011/11/sdwebimage-fixed-width-cell-images/)

### Handle image refresh

SDWebImage默认是主动缓存。它会忽略所有HTTP服务器返回的缓存控制头，缓存没有时间限制。这意味着图片URL始终指向这个图片不会改变。如果指向发生了变化，URL应该因此改变。

如果你不控制图片服务器，当内容更新时就不能改变URL。这是一个Facebook头像（avatar）URL的实例。在这种情况下可以使用 SDWebImageRefreshCached 标记，来使用HTTP缓存控制头，但会稍稍降低性能：

```objc
[imageView sd_setImageWithURL:[NSURL URLWithString:@"https://graph.facebook.com/olivier.poitrey/picture"]
                 placeholderImage:[UIImage imageNamed:@"avatar-placeholder.png"]
                          options:SDWebImageRefreshCached];
```

### Add a progress indicator

参见: https://github.com/JJSaccolo/UIActivityIndicator-for-SDWebImage

## Installation