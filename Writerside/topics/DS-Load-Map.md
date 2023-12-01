# DS 加载地图流程

## 概述

- DS 启动之后，默认会创建一个 World，该 World 会默认加载一个地图。
- 如果想切换一个地图（非Level）的话，需要新创建一个 World，并指定一个新地图。
- 有没有一种方式是在当前 World 的基础上，直接加载一个新的地图呢？

回答这个问题的前提是先了解一下 DS 加载地图的过程，也就是 `UEngine::LoadMap` 这个方法。

## UEngine::LoadMap 解析

该方法一共有 700 行，实在太大，只能分成几部分来解析

### 第一部分-准备加载阶段 {collapsible="true"}

```C++
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{
	// ---------- 第一部分开始 ---------------
    // 会在 Unreal Insights 工具的 Timing 面板上显示出来，标识一下系统当前处于什么状态下
    // 由于 URL.Map 记录了当前需要加载的地图名称，所以追踪一下该变量的使用位置
    TRACE_BOOKMARK(TEXT("LoadMap - %s"), *URL.Map);
    
    // 移除 PIE 模式加的前缀
    const FString OriginalURLMap = URL.Map;
    URL.Map = UWorld::RemovePIEPrefix(URL.Map);
    //.......... 省略此处无关代码 ..............
    // 通知准备加载场景
    FCoreUObjectDelegates::PreLoadMapWithContext.Broadcast(WorldContext, URL.Map);
	FCoreUObjectDelegates::PreLoadMap.Broadcast(URL.Map);
    //.......... 省略此处无关代码 ..............
	if (WorldContext.World() && WorldContext.World()->PersistentLevel)
	{
	    // 如果存在一个持久化的 Level，则清理已有的地图信息
	    // 首次加载的话，里面的信息为空
		CleanupPackagesToFullyLoad(WorldContext, FULLYLOAD_Map, WorldContext.World()->PersistentLevel->GetOutermost()->GetName());
	}
	// ---------- 第一部分结束 ---------------
}	
```

### 第二部分-卸载World {collapsible="true" id="world_1"}

```c++
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{
	// ---------- 第二部分开始 ---------------
	// 以下流程是卸载 World
	if( WorldContext.World() )
	{
	    // 通知准备将 Level 从 World 离开
	    FWorldDelegates::PreLevelRemovedFromWorld.Broadcast(nullptr, WorldContext.World());
	    WorldContext.World()->BeginTearingDown();
        //.......... 省略此处无关代码 ..............
        // 关闭网络监听
	    ShutdownWorldNetDriver(WorldContext.World());    
        //.......... 省略此处无关代码 ..............
        // 通知 Level 移除成功
	    FWorldDelegates::LevelRemovedFromWorld.Broadcast(nullptr, WorldContext.World());
	    
	    // OwningGameInstance 对应的是项目自己的 GameInstance 对象
	    if (WorldContext.OwningGameInstance != nullptr)
	    {
	        // 遍历所有的用户，并销毁
	        for(auto It = WorldContext.OwningGameInstance->GetLocalPlayerIterator(); It; ++It)
			{
				ULocalPlayer *Player = *It;
				if(Player->PlayerController && Player->PlayerController->GetWorld() == WorldContext.World())
				{
					if(Player->PlayerController->GetPawn())
					{
						WorldContext.World()->DestroyActor(Player->PlayerController->GetPawn(), true);
					}
					WorldContext.World()->DestroyActor(Player->PlayerController, true);
					Player->PlayerController = nullptr;
				}
                //.......... 省略此处无关代码 ..............
			}
	    }
	    // TODO：暂时不清楚作用
	    for (FActorIterator ActorIt(WorldContext.World()); ActorIt; ++ActorIt)
		{
			ActorIt->RouteEndPlay(EEndPlayReason::LevelTransition);
		}
		// 清理世界
		WorldContext.World()->CleanupWorld();
        //.......... 省略此处无关代码 ..............
		WorldContext.World()->RemoveFromRoot();
        //.......... 省略此处无关代码 ..............
        // 设置当前 Wolrd 为空
		WorldContext.SetCurrentWorld(nullptr);
	}
	// ---------- 第二部分结束 ---------------
}
```

### 第三部分-加载地图并创建新World {collapsible="true" id="world_2"}

```c++
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{
	// ---------- 第三部分开始 ---------------
	// 调用业务定义的 GameInstance 对象的目标方法
    WorldContext.OwningGameInstance->PreloadContentForURL(URL);
    
    UPackage* WorldPackage = NULL;
	UWorld*	NewWorld = NULL;
	
    const FString URLTrueMapName = URL.Map;
    if (NewWorld == NULL)
    {
		// See if the level is already in memory || 看是否 Level 已经在内存中了
		// 而且所谓的 PackageName 就是地图的名称
		WorldPackage = FindPackage(nullptr, *URL.Map);
		// 首次加载肯定不在内存中，所以 WorldPackage 为空
		bool bPackageAlreadyLoaded = (WorldPackage != nullptr);
		if (WorldPackage == nullptr)
		{
		    // 执行地图加载
		    WorldPackage = LoadPackage(nullptr, *URL.Map, (WorldContext.WorldType == EWorldType::PIE ? LOAD_PackageForPIE : LOAD_None));
		}
		// 从包里找一个新 World 对象
		NewWorld = UWorld::FindWorldInPackage(WorldPackage);
		// TODO：暂时不理解用途
		NewWorld->PersistentLevel->HandleLegacyMapBuildData();
    }
    // 设置 world 的 GameInstance
    NewWorld->SetGameInstance(WorldContext.OwningGameInstance);
    GWorld = NewWorld;
    WorldContext.SetCurrentWorld(NewWorld);
	WorldContext.World()->WorldType = WorldContext.WorldType;
	if (WorldContext.WorldType == EWorldType::PIE)
	{
	}
	else 
	{
	    WorldContext.World()->AddToRoot();
	    WorldContext.World()->InitWorld();
	}
	// ---------- 第三部分结束 ---------------
}
```

### 第四部分-新World初始化 {collapsible="true"}

```c++
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{
    // ---------- 第四部分开始 ---------------
    WorldContext.World()->SetGameMode(URL);
    if (Pending == NULL && (!GIsClient || URL.HasOption(TEXT("Listen"))))
    {
        // 开启网络监听
        WorldContext.World()->Listen(URL);
    }
    //TODO: 暂不清楚
    if (GShaderCompilingManager)
	{
		GShaderCompilingManager->ProcessAsyncResults(false, true);
	}
	// load any per-map packages
	LoadPackagesFully(WorldContext.World(), FULLYLOAD_Map, WorldContext.World()->PersistentLevel->GetOutermost()->GetName());
	// Make sure "always loaded" sub-levels are fully loaded
	WorldContext.World()->FlushLevelStreaming(EFlushLevelStreamingType::Visibility);
    // AI 系统
	WorldContext.World()->CreateAISystem();
    // Initialize network actors and start execution.
    for( int32 LevelIndex=0; LevelIndex<Levels.Num(); LevelIndex++ )
    {
        ULevel*	const Level = Levels[LevelIndex];
        Level->InitializeNetworkActors();
    }
    bStartup = true;
    
    // Spawn server actors
    ENetMode CurNetMode = GEngine != NULL ? GEngine->GetNetMode(this) : NM_Standalone;
    if (CurNetMode == NM_ListenServer || CurNetMode == NM_DedicatedServer)
    {
        GEngine->SpawnServerActors(this);
    }
    
    // Init the game mode.
    if (AuthorityGameMode && !AuthorityGameMode->IsActorInitialized())
    {
        AuthorityGameMode->InitGame( FPaths::GetBaseFilename(InURL.Map), Options, Error );
    }
    
    // 导航系统
    if (NavigationSystem != nullptr)
	{
		NavigationSystem->OnInitializeActors();
	}
	// ai 系统初始化
    if (AISystem != nullptr)
	{
		AISystem->InitializeActorsForPlay(bResetTime);
	}	
	// ---------- 第四部分结束 ---------------
}
```

### 小结

粗略分析 LoadMap 的要点如下：

1. 地图加载之前会执行旧 World 的卸载流程
2. 加载过程会创建一个新的 World 并执行初始化
3. 地图加载有缓存机制，只有新地图会从磁盘加载到内存

另外上面的 `LoadMap` 方法有一点违反直觉。正常是先创建一个 World，然后加载地图资源映射到 World 上。 它的实现是先加载地图，然后从
Map 中获取了一个 World 对象。因此我们无法把加载的地图资源映射到任意 World 上，只能深入再看下 World 是怎样绑定到 Map
上的，也就是 `UWorld::FindWorldInPackage` 方法。

## UWorld::FindWorldInPackage 解析

该方法是 World 类的静态方法

```c++
UWorld* UWorld::FindWorldInPackage(UPackage* Package)
{
    UWorld* RetVal = nullptr;
	TArray<UObject*> PotentialWorlds;
	// 从地图 Package 中获取所有的 Objects，放到 PotentialWorlds 数组中
	GetObjectsWithPackage(Package, PotentialWorlds, false);
	for ( auto ObjIt = PotentialWorlds.CreateConstIterator(); ObjIt; ++ObjIt )
	{
	    // 是否可以强转成 UWorld 类型，是的话说明是 World 对象
		RetVal = Cast<UWorld>(*ObjIt);
		if ( RetVal )
		{
			break;
		}
	}
	return RetVal;
}
```

根据 UE 官方的解释，Package 是一个打包后的资产集合。也就是说在打包的时候，地图和 World 的映射早已建立了。

### 关于 GetObjectsWithPackage 的简要分析 {collapsible="true"}

#### 概念

先给出 GPT 的解释:

---
GetObjectsWithPackage 是一个 UE 的源码方法，它的作用是返回一个给定包中的所有对象的数组¹。你可以使用这个方法来查找一个包中的所有
UObject，或者指定一些过滤条件，比如是否包含嵌套对象，或者排除一些特定的对象标志¹。这个方法的参数和返回值如下：

- 参数：
    - const class UPackage * Outer：要搜索的包的指针
    - TArray < UObject * > & Results：用于存放结果的数组的引用
    - bool bIncludeNestedObjects：是否包含嵌套对象，如果为 true，那么直接或间接以 Outer 为外部的对象都会被包含
    - EObjectFlags ExclusionFlags：用于过滤的对象标志，比如 RF_Transient，RF_Public 等
    - EInternalObjectFlags ExclusionInternalFlags：用于过滤的内部对象标志，比如 EInternalObjectFlags::
      GarbageCollectionKeepFlags 等
- 返回值：
    - void：没有返回值，结果会存放在 Results 数组中

---

UE 中 Package 的概念如下：

---

- UE 中的包（Package）是一种用于存储和分发 Unreal 项目的内容和代码的文件格式。
- 一个包文件通常包含一个或多个资产（Asset），比如模型，材质，纹理，声音，蓝图等2。包文件的扩展名是
  .uasset 或 .umap，它们可以在内容浏览器（Content
  Browser）中查看和管理。
- 打包(Packaging)
  是一个将项目的内容和代码转换为适合目标平台运行的格式的过程3。打包过程涉及到编译（Compile），烘焙（Cook），打包（Package），部署（Deploy）和运行（Run）等步骤3。打包的目的是为了将项目发布给用户，或者进行测试和调试3。你可以在编辑器的菜单栏中选择
  File > Package Project > [PlatformName] 来打包你的项目1。

---

#### 源码解析

从实际的 debug 信息来看，该方法的返回值是如下列表，里面包含一个 World 对象

- [0] = {UMetaData *} 0x00000a99cf39b600 (Name="PackageMetaData")
- [1] = {UBlueprintGeneratedClass *} 0x00000a99cf330700 (Name="CombatTestMap_C")
- [2] = {UWorld *} 0x00000a99ce6dd800 (Name="CombatTestMap")
- [3] = {ALevelScriptActor *} 0x00000a99cf389600 (Name="Default__CombatTestMap_C")
- [4] = {UBlueprintGeneratedClass *} 0x00000a99cf330e00 (Name="SKEL_CombatTestMap_C")
- [5] = {ALevelScriptActor *} 0x00000a99cf387800 (Name="Default__SKEL_CombatTestMap_C")

下面方法的作用就是返回上面的列表

```c++
void GetObjectsWithPackage(const class UPackage* Package, TArray<UObject *>& Results, bool bIncludeNestedObjects, EObjectFlags ExclusionFlags, EInternalObjectFlags ExclusionInternalFlags)
{
	ForEachObjectWithPackage(Package, [&Results](UObject* Object)
	{
		Results.Add(Object);
		return true;
	}, bIncludeNestedObjects, ExclusionFlags, ExclusionInternalFlags);
}
```

以下方法省略了大量代码，保留了和 Operation 相关的部分

```c++
void ForEachObjectWithPackage(const class UPackage* Package, TFunctionRef<bool(UObject*)> Operation, bool bIncludeNestedObjects, EObjectFlags ExclusionFlags, EInternalObjectFlags ExclusionInternalFlags)
{
    // 方法的 Operation 是个 lambda，看一下它被使用的位置
    
    FUObjectHashTables& ThreadHash = FUObjectHashTables::Get();
    // 一个数组，用于存储 Object 的 hash 桶 (也就是链表或者数结构存储的 Object)
    TArray<FHashBucket*, TInlineAllocator<1> > AllInners;
    if (FHashBucket* Inners = ThreadHash.PackageToObjectListMap.Find(Package))
	{
		AllInners.Add(Inners);
	}
	if (FHashBucket* ObjectInners = ThreadHash.ObjectOuterMap.Find(Package))
	{
		AllInners.Add(ObjectInners);
	}
    while (AllInners.Num())
	{
	    FHashBucket* Inners = AllInners.Pop();
	    for (FHashBucketIterator It(*Inners); It; ++It)
	    {
	        UObject *Object = static_cast<UObject*>(*It);
	        // 调用 Lambda
	        Operation(Object)
	    }
	}
}
```

### 小结

从 `UWorld::FindWorldInPackage` 方法可以看出，地图和 World 深层绑定，所以