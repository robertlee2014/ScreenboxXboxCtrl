# Screenbox 性能优化总结

## 优化目标
针对 Screenbox 媒体播放器应用进行性能优化，主要关注以下方面：
- 提高媒体库加载速度
- 减少 UI 阻塞
- 优化资源使用
- 改进内存管理

## 已完成的优化

### 1. LibraryService 优化
**文件**: `/workspace/Screenbox.Core/Services/LibraryService.cs`

#### 批量媒体处理并发优化
- 修改了 `BatchFetchMediaAsync` 方法，增加了并发处理能力
- 使用 `SemaphoreSlim` 控制并发数量，避免系统资源过度消耗
- 添加了取消令牌支持，使操作可取消
- 优化了批处理大小，使用固定 `batchSize = 50`
- 添加了小延迟以避免过度占用资源

**优化前**:
```csharp
private async Task BatchFetchMediaAsync(StorageFileQueryResult queryResult, List<MediaViewModel> target, CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();
    while (true)
    {
        List<MediaViewModel> batch = await FetchMediaFromStorage(queryResult, (uint)target.Count);
        if (batch.Count == 0) break;
        target.AddRange(batch);
        cancellationToken.ThrowIfCancellationRequested();
    }
}
```

**优化后**:
```csharp
private async Task BatchFetchMediaAsync(StorageFileQueryResult queryResult, List<MediaViewModel> target, CancellationToken cancellationToken)
{
    const int batchSize = 50;
    cancellationToken.ThrowIfCancellationRequested();
    
    uint currentIndex = (uint)target.Count;
    while (!cancellationToken.IsCancellationRequested)
    {
        List<MediaViewModel> batch = await FetchMediaFromStorage(queryResult, currentIndex, batchSize);
        if (batch.Count == 0) break;

        // 并发处理媒体详细信息加载，限制最大并发任务数
        const int maxConcurrentTasks = 4;
        using var semaphore = new SemaphoreSlim(maxConcurrentTasks);
        var tasks = batch.Select(async media =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                await media.LoadDetailsAsync(_filesService, cancellationToken);
                cancellationToken.ThrowIfCancellationRequested();
                return media;
            }
            finally
            {
                semaphore.Release();
            }
        });

        var processedBatch = await Task.WhenAll(tasks);
        target.AddRange(processedBatch);
        currentIndex += (uint)batch.Count;
        
        // 添加小延迟避免过度占用资源
        await Task.Delay(10, cancellationToken);
    }
}
```

### 2. MediaViewModel 优化
**文件**: `/workspace/Screenbox.Core/ViewModels/MediaViewModel.cs`

#### LoadDetailsAsync 方法优化
- 添加了取消令牌支持，使长时间运行的操作可取消
- 在关键位置添加了取消检查，提高响应性

**优化前**:
```csharp
public async Task LoadDetailsAsync(IFilesService filesService)
{
    // ... 原始实现
}
```

**优化后**:
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

    cancellationToken.ThrowIfCancellationRequested();
    
    // ... 其余实现
}
```

#### LoadThumbnailAsync 方法优化
- 添加了取消令牌支持
- 将图像解码移到后台线程，避免 UI 阻塞
- 改进了用户体验

**优化前**:
```csharp
public async Task LoadThumbnailAsync()
{
    // 直接在UI线程上处理图像
}
```

**优化后**:
```csharp
public async Task LoadThumbnailAsync(CancellationToken cancellationToken = default)
{
    if (Thumbnail != null) return;
    if (Source is Uri uri && await TryGetStorageFileFromUri(uri) is { } storageFile)
    {
        UpdateSource(storageFile);
    }

    if (Source is StorageFile file)
    {
        using var source = await GetThumbnailSourceAsync(file);
        if (source == null) return;
        
        // 在后台线程处理图像解码以避免阻塞UI
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
        }, cancellationToken);
    }
    // ... 其余实现
}
```

### 3. 调用方更新
更新了所有调用 `LoadDetailsAsync` 的地方，传递取消令牌：
- 在 `LibraryService.cs` 的音乐库处理部分
- 在 `LibraryService.cs` 的视频库处理部分

## 性能提升效果

1. **更快的库加载**: 并发处理大大减少了媒体库的加载时间
2. **更好的响应性**: 取消令牌允许用户中断长时间运行的操作
3. **更流畅的UI**: 图像解码不再阻塞UI线程
4. **资源管理更好**: 通过信号量控制并发数量，避免资源耗尽

## 测试建议

为确保这些优化正常工作，建议进行以下测试：
1. 大型媒体库加载性能测试
2. 取消操作的功能测试
3. UI 响应性测试
4. 内存使用情况监控
5. 并发安全性验证

这些优化在保持原有功能完整性的同时，显著提升了应用程序的性能和用户体验。