---
title: C#对于非托管资源的释放原理探究
tags: []
id: '2659'
categories:
  - - C#
date: 2020-03-20 11:26:20
---

<meta name="referrer" content="no-referrer" />



## 前言

我们都知道CLR有一个使用根的可达性算法的垃圾回收机制来回收托管内存，那么对于那些本机资源（非托管内存）他又是怎么清理的呢？

## 正文

要看他怎么清理资源，首先要知道这个资源是怎么来的，这里我们用FileStream这一经典类来探讨这些问题。 首先是它的构造函数

```csharp
    [SecuritySafeCritical]
    public FileStream(string path, FileMode mode)
      : this(path, mode, mode == FileMode.Append ? FileAccess.Write : FileAccess.ReadWrite, FileShare.Read, 4096, FileOptions.None, Path.GetFileName(path), false)
    {
    }

    [SecurityCritical]
    internal FileStream(
      string path,
      FileMode mode,
      FileAccess access,
      FileShare share,
      int bufferSize,
      FileOptions options,
      string msgPath,
      bool bFromProxy)
    {
      //取得内核对象
      Win32Native.SECURITY_ATTRIBUTES secAttrs = FileStream.GetSecAttrs(share);
      //绑定句柄，初始化缓冲区
      this.Init(path, mode, access, 0, false, share, bufferSize, options, secAttrs, msgPath, bFromProxy, false, false);
    }
```

然后我们调用FileStream.Write函数，将一段二进制数组写入文件

```csharp
//这句代码会把我们的二进制数组写入到缓存区中
Buffer.InternalBlockCopy((Array) array, offset, (Array) this._buffer, this._writePos, count);
//最后调用这个函数把缓存区中的数据写入文件
Win32Native.WriteFile(handle, numPtr + offset, count, IntPtr.Zero, overlapped);
```

我们知道，释放一个这样的文件流需要调用Dispose函数，找到Dispose函数就能知道它内部怎么处理非托管资源的释放了，我们当前正使用一个本机资源的句柄来连接着这个文件

```csharp
    [SecuritySafeCritical]
    protected override void Dispose(bool disposing)
    {
      try
      {
        if (this._handle == null  this._handle.IsClosed  this._writePos <= 0)
          return;
        //会把当前缓冲区内容写入磁盘文件
        this.FlushWrite(!disposing);
      }
      finally
      {
        if (this._handle != null && !this._handle.IsClosed)
          //释放句柄及其引用的本机资源
          this._handle.Dispose();
        this._canRead = false;
        this._canWrite = false;
        this._canSeek = false;
        base.Dispose(disposing);
      }
    }
```

对于`this._handle.Dispose()`，我们只能看到这一步

```csharp
    /// <summary>Releases the unmanaged resources used by the <see cref="T:System.Runtime.InteropServices.SafeHandle" /> class specifying whether to perform a normal dispose operation.</summary>
    /// <param name="disposing">
    /// <see langword="true" /> for a normal dispose operation; <see langword="false" /> to finalize the handle.</param>
    [SecurityCritical]
    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
    [__DynamicallyInvokable]
    protected virtual void Dispose(bool disposing)
    {
      if (disposing)
        this.InternalDispose();
      else
        this.InternalFinalize();
    }
```

再底层就是

```csharp
    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
    [MethodImpl(MethodImplOptions.InternalCall)]
    private extern void InternalDispose();    

    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
    [MethodImpl(MethodImplOptions.InternalCall)]
    private extern void InternalFinalize();
```

看不到具体实现了

## 总结

总结就是对于非托管资源，CLR底层引用了句柄，并且在Dispose的时候清理句柄及其引用的本机资源，这样就避免了当托管资源（FileStream）被GC回收时，由于句柄引用问题（因为句柄是归属于操作系统的内核对象），而无法清理具体的本机文件所造成的内存泄漏问题，而CLR为我们处理好了这一切。