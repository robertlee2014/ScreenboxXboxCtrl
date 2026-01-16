# Screenbox 性能优化报告

## 项目概述
Screenbox 是一个 UWP 媒体播放器应用程序，支持音乐和视频库管理、播放控制等功能。

## 性能分析发现的问题

### 1. 文件服务中的性能问题
- 在 `FilesService.cs` 中，获取相邻文件时可能存在效率问题
- 缺少对大目录的分页处理优化

### 2. 库服务中的性能问题
- `LibraryService.cs` 中批量处理媒体文件时，每次都要加载详细信息
- 没有并发处理，导致加载时间过长
- 缓存机制可以进一步优化

### 3. 媒体视图模型中的性能问题
- `MediaViewModel.cs` 中的异步操作没有充分并发化
- 缩略图加载过程可能阻塞 UI

## 性能优化建议

### 1. 并发处理优化
```csharp
// 优化 LibraryService 中的批量处理
private async Task BatchFetchMediaAsync(StorageFileQueryResult queryResult, List<MediaViewModel> target, CancellationToken cancellationToken)
{
    const int batchSize = 50;
    uint currentIndex = (uint)target.Count;
    
    while (!cancellationToken.IsCancellationRequested)
    {
        // 使用并行处理来提高效率
        var batch = await FetchMediaFromStorage(queryResult, currentIndex, batchSize);
        if (batch.Count == 0) break;
        
        target.AddRange(batch);
        currentIndex += (uint)batch.Count;
        
        // 可选：添加小延迟避免过度占用资源
        await Task.Delay(10, cancellationToken);
    }
}
```

### 2. 异步操作优化
在 MediaViewModel 中，可以对缩略图加载等操作进行优化，避免不必要的重复工作：

```csharp
public async Task LoadThumbnailAsync()
{
    if (Thumbnail != null) return;
    
    // 实现缓存和取消令牌
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
    
    if (Source is Uri uri && await TryGetStorageFileFromUri(uri) is { } storageFile)
    {
        UpdateSource(storageFile);
    }

    if (Source is StorageFile file)
    {
        using var source = await GetThumbnailSourceAsync(file);
        if (source == null) return;
        
        // 在后台线程处理图像解码
        await Task.Run(async () =>
        {
            BitmapImage image = new()
            {
                DecodePixelType = DecodePixelType.Logical,
                DecodePixelHeight = 300
            };

            try
            {
                await image.SetSourceAsync(source);
                Thumbnail = image;
            }
            catch (Exception)
            {
                return;
            }
        }, cts.Token);
    }
}
```

### 3. 内存管理优化
- 使用对象池减少内存分配
- 改进弱引用管理以防止内存泄漏
- 优化图片缓存策略

### 4. 数据库查询优化
- 优化存储查询参数，避免不必要的属性预取
- 使用更有效的索引策略

## 具体优化实现

### 优化 LibraryService 中的媒体加载
修改 `BatchFetchMediaAsync` 方法，添加并发处理能力：

```csharp
private async Task BatchFetchMediaAsync(StorageFileQueryResult queryResult, List<MediaViewModel> target, CancellationToken cancellationToken)
{
    const int batchSize = 50;
    const int maxConcurrentTasks = 4; // 控制并发数
    uint currentIndex = (uint)target.Count;
    
    while (!cancellationToken.IsCancellationRequested)
    {
        // 获取一批文件路径
        var files = await queryResult.GetFilesAsync(currentIndex, batchSize);
        if (files.Count == 0) break;
        
        // 并发处理这批文件，限制最大并发任务数
        var semaphore = new SemaphoreSlim(maxConcurrentTasks);
        var tasks = files.Select(async file =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                return _mediaFactory.GetSingleton(file);
            }
            finally
            {
                semaphore.Release();
            }
        });
        
        var mediaBatch = await Task.WhenAll(tasks);
        target.AddRange(mediaBatch);
        currentIndex += (uint)files.Count;
    }
}
```

### 优化 MediaViewModel 的详细信息加载
```csharp
public async Task LoadDetailsAsync(IFilesService filesService, CancellationToken cancellationToken = default)
{
    cancellationToken.ThrowIfCancellationRequested();
    
    switch (Source)
    {
        case StorageFile file:
            MediaInfo = await filesService.GetMediaInfoAsync(file);
            break;
        case Uri uri when await TryGetStorageFileFromUri(uri) is { } uriFile:
            UpdateSource(uriFile);
            MediaInfo = await filesService.GetMediaInfoAsync(uriFile);
            break;
    }
    
    // 继续处理其他逻辑...
}
```

这些优化将显著提升应用程序的性能，特别是在处理大量媒体文件时。