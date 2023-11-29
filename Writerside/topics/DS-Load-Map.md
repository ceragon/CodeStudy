# DS 加载地图流程

## 概述

- DS 启动之后，默认会创建一个 World，该 World 会默认加载一个地图。
- 如果想切换一个地图（非Level）的话，需要新创建一个 World，并指定一个新地图。
- 有没有一种方式是在当前 World 的基础上，直接加载一个新的地图呢？

回答这个问题的前提是先了解一下 DS 加载地图的过程，也就是 `UEngine::LoadMap` 这个方法。

## LoadMap 解析 

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

### 第二部分-卸载 World {collapsible="true"}

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

### 第三部分-加载地图并创建新World {collapsible="true}

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

### 第四部分-新 World 初始化

```c++
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{
    // ---------- 第四部分开始 ---------------
    
	// ---------- 第四部分结束 ---------------
}
```