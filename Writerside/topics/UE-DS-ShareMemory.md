# UE DS 内存共享方案

## 概述

## 预加载地图资源

UE 中的地图资源被打包成了 Package，通过 LoadPackage 可以加载地图的 Package 到全局缓存中。接下来分析一下 LoadPackage
的流程:

```c++
UPackage* LoadPackage(UPackage* InOuter, const TCHAR* InLongPackageNameOrFilename, uint32 LoadFlags, FArchive* InReaderOverride, const FLinkerInstancingContext* InstancingContext)
{
    // InLongPackageNameOrFilename: 地图的资源地址
    FPackagePath PackagePath;
    // 将地图资源地址放入 PackagePath 中
    FPackagePath::TryFromMountedName(InLongPackageNameOrFilename, PackagePath);
    // 调用重载方法
    return LoadPackage(InOuter, PackagePath, LoadFlags, InReaderOverride, InstancingContext, DiffPackagePathPtr);
}
```

被上面的方法调用

```c++
UPackage* LoadPackage(UPackage* InOuter, const FPackagePath& PackagePath, uint32 LoadFlags, FArchive* InReaderOverride, const FLinkerInstancingContext* InstancingContext, const FPackagePath* DiffPackagePath)
{
    // PackagePath: 保存了地图资源地址
    // 调用下面的方法
    return LoadPackageInternal(InOuter, PackagePath, LoadFlags, /*ImportLinker =*/ nullptr, InReaderOverride, InstancingContext, DiffPackagePath);
}
```

```c++
UPackage* LoadPackageInternal(UPackage* InOuter, const FPackagePath& PackagePath, uint32 LoadFlags, FLinkerLoad* ImportLinker, FArchive* InReaderOverride,
	const FLinkerInstancingContext* InstancingContext, const FPackagePath* DiffPackagePath)
{
    UPackage* Result = nullptr;
    
    FLinkerLoad* Linker = nullptr;
    FUObjectSerializeContext* InOutLoadContext = LoadContext;
    // 调用下面的方法
    Linker = GetPackageLinker(InOuter, PackagePath, LoadFlags, nullptr, InReaderOverride, &InOutLoadContext, ImportLinker, InstancingContext);
    Result = Linker->LinkerRoot;
    
    return Result;
}
```

```c++
FLinkerLoad* GetPackageLinker
(
	UPackage*		InOuter,
	const FPackagePath& InPackagePath,
	uint32			LoadFlags,
	UPackageMap*	Sandbox,
	FArchive*		InReaderOverride,
	FUObjectSerializeContext** InOutLoadContext,
	FLinkerLoad*	ImportLinker,
	const FLinkerInstancingContext* InstancingContext
)
{
	UPackage* TargetPackage = nullptr;
	
	UPackage* CreatedPackage = nullptr;
	if (!TargetPackage)
	{
        // 创建 Package 对象 
        CreatedPackage = CreatePackage(*PackageNameToCreate);
        TargetPackage = CreatedPackage;
	}
    TRefCountPtr<FUObjectSerializeContext> LoadContext(InExistingContext ? InExistingContext : FUObjectThreadContext::Get().GetSerializeContext());
    // 调用了 CreateLinker 方法
    FLinkerLoad* Result = FLinkerLoad::CreateLinker(LoadContext, TargetPackage, PackagePath, LoadFlags, InReaderOverride, InstancingContext);
    return Result;	
}
```

```c++
FLinkerLoad* FLinkerLoad::CreateLinker(FUObjectSerializeContext* LoadContext, UPackage* Parent, const FPackagePath& PackagePath, uint32 LoadFlags, FArchive* InLoader /*= nullptr*/, const FLinkerInstancingContext* InstancingContext /*= nullptr*/)
{
    // 调用了 CreateLinkerAsync 方法
    FLinkerLoad* Linker = CreateLinkerAsync(LoadContext, Parent, PackagePath, LoadFlags, InstancingContext,
                TFunction<void()>([](){}));
    TGuardValue<FLinkerLoad*> SerializedPackageLinkerGuard(LoadContext->SerializedPackageLinker, Linker);            
    if (Linker->Tick(0.f, false, false, nullptr) == LINKER_Failed)
    {
        return nullptr;
    }
    return Linker;
}
```

```c++
FLinkerLoad::ELinkerStatus FLinkerLoad::Tick( float InTimeLimit, bool bInUseTimeLimit, bool bInUseFullTimeLimit, TMap<TPair<FName, FPackageIndex>, FPackageIndex>* ObjectNameWithOuterToExportMap)
{
	ELinkerStatus Status = LINKER_Loaded;
    Status = CreateLoader(TFunction<void()>([]() {}));
    return Status;
}
```

```c++
FLinkerLoad::ELinkerStatus FLinkerLoad::CreateLoader(
	TFunction<void()>&& InSummaryReadyCallback
	)
{
    FOpenPackageResult OpenResult;
    // 调用了 OpenReadPackage 方法，用于 open file
    OpenResult = IPackageResourceManager::Get().OpenReadPackage(GetPackagePath());
    Loader = OpenResult.Archive.Release();
    SetLoader(Loader, bLoaderNeedsEngineVersionChecks);
    int64 Size = Loader->TotalSize();
    bExecuteNextStep = Loader->Precache(0, PrecacheSize);
    return ;
}
```

### open file

```c++
FOpenPackageResult IPackageResourceManager::OpenReadPackage(const FPackagePath& PackagePath, FPackagePath* OutUpdatedPath)
{
    return OpenReadPackage(PackagePath, EPackageSegment::Header, OutUpdatedPath);
}
```

```c++
FOpenPackageResult FPackageResourceManagerFile::OpenReadPackage(const FPackagePath& PackagePath,
	EPackageSegment PackageSegment, FPackagePath* OutUpdatedPath)
{
    FOpenPackageResult Result{ nullptr, EPackageFormat::Binary, true /* bNeedsEngineVersionChecks */};
    IteratePossibleFiles(PackagePath, PackageSegment, OutUpdatedPath,
		[FileManager, &Result](const TCHAR* Filename, EPackageExtension Extension)
	{
	    // 打开文件
		Result.Archive = TUniquePtr<FArchive>(FileManager->CreateFileReader(Filename));
		if (Result.Archive)
		{
		    Result.Format = EPackageFormat::Binary;
		    return true;
		}
		return false;
	});
    return Result;
}
```

### read file

```c++
bool FArchiveFileReaderGeneric::Precache(int64 PrecacheOffset, int64 PrecacheSize)
{
    InternalPrecache(PrecacheOffset, PrecacheSize);
    return true;
}
```

```c++
bool FArchiveFileReaderGeneric::InternalPrecache( int64 PrecacheOffset, int64 PrecacheSize )
{
    ReadLowLevel( BufferArray.GetData() + WriteOffset, ReadCount, Count );
    return true;
}
```

```c++
void FArchiveFileReaderGeneric::ReadLowLevel( uint8* Dest, int64 CountToRead, int64& OutBytesRead )
{
    // 将文件内容加载到 Dest byte数组中
    if( Handle->Read( Dest, CountToRead ) )
    {
        OutBytesRead = CountToRead;
    } 
    else 
    {
        OutBytesRead = 0;
    }
}
```
