# 客户端加载地图流程

## 概述

客户端进入游戏或是切换地图，都需要调用 `APlayerController::ClientTravel` 方法。需要调研一下该方法的执行流程：

## 发起 travel

```c++
void APlayerController::ClientTravel(const FString& URL, ETravelType TravelType, bool bSeamless, FGuid MapPackageGuid)
{
    //...... 省略代码 
	ClientTravelInternal(URL, TravelType, bSeamless, MapPackageGuid);
}
```

下面的方法被上面的方法调用

```c++
void APlayerController::ClientTravelInternal_Implementation(const FString& URL, ETravelType TravelType, bool bSeamless, FGuid MapPackageGuid)
{
    if (bSeamless && TravelType == TRAVEL_Relative)
    {
        // 无缝切换
        World->SeamlessTravel(URL);
    }
    else 
    {
        // 普通切换
        GEngine->SetClientTravel(World, *URL, (ETravelType)TravelType);
    }
}
```

## 普通切换

在发起 travel 的流程中，普通切换会调用下面的方法

```c++
void UEngine::SetClientTravel( UWorld *InWorld, const TCHAR* NextURL, ETravelType InTravelType )
{
    FWorldContext &Context = GetWorldContextFromWorldChecked(InWorld);
    Context.TravelURL    = NextURL;
    Context.TravelType   = InTravelType;
}
```

由于没有任何切换的行为，所以后续流程应该是异步执行。因此反查一下 TravelURL 变量的使用。

```c++
void UEngine::TickWorldTravel(FWorldContext& Context, float DeltaSeconds)
{
    if( !Context.TravelURL.IsEmpty() )
    {
        AGameModeBase* const GameMode = Context.World()->GetAuthGameMode();
		if (GameMode)
		{
		    // 通知 gameMode 准备离开当前地图
			GameMode->StartToLeaveMap();
			FString Error, TravelURLCopy = Context.TravelURL;
			// 重点是这个 Browse 方法，在里面执行了切换
			if (Browse( Context, FURL(&Context.LastURL,*TravelURLCopy,(ETravelType)Context.TravelType), Error ) == EBrowseReturnVal::Failure)
			{
			    if (Context.World() == NULL)
			    {
			        // 切换失败，回到默认场景
				    BrowseToDefaultMap(Context);
			    }
			    // 发出切换失败的通知
			    BroadcastTravelFailure(Context.World(), ETravelFailure::ClientTravelFailure, Error);
			}
		}
    }
}
```

再查看一下 Browse 方法的调用，由于是客户端跳转，所以会执行 `else if` 的逻辑

```c++
EBrowseReturnVal::Type UEngine::Browse( FWorldContext& WorldContext, FURL URL, FString& Error )
{
    if( URL.IsLocalInternal() )
    {
        // 执行服务器的地图跳转
        // ....... 代码省略
    }
    else if( URL.IsInternal() && GIsClient )
    {
        // 执行客户端的地图跳转
        if( WorldContext.PendingNetGame )
        {
            CancelPending(WorldContext);
        }
        if (WorldContext.World() && ShouldShutdownWorldNetDriver())
        {
            // 关闭 ds 连接
            ShutdownWorldNetDriver(WorldContext.World());
        }
        
        WorldContext.PendingNetGame = NewObject<UPendingNetGame>();
        WorldContext.PendingNetGame->Initialize(URL);
        // 在 InitNetDriver 会执行与目标 ds 的连接
        WorldContext.PendingNetGame->InitNetDriver();
        
        return EBrowseReturnVal::Pending;
    }
}
```

由于 browse 中，只是设置了 PendingNetGame，也没有真正执行跳转，所以还需要进一步查看 PendingNetGame 的使用。

```c++
void UEngine::TickWorldTravel(FWorldContext& Context, float DeltaSeconds)
{
    if( Context.PendingNetGame )
    {
        Context.PendingNetGame->Tick( DeltaSeconds );
        if ( Context.PendingNetGame && Context.PendingNetGame->ConnectionError.Len() > 0 )
        {
            // 连接出现错误
            //....... 代码省略
        }
        else if (Context.PendingNetGame && Context.PendingNetGame->bSuccessfullyConnected && !Context.PendingNetGame->bSentJoinRequest && !Context.PendingNetGame->bLoadedMapSuccessfully && (Context.OwningGameInstance == NULL || !Context.OwningGameInstance->DelayPendingNetGameTravel()))
        {
            // 连接成功
            //........ 省略执行不成功的部分
            if (!Context.PendingNetGame->bLoadedMapSuccessfully)
            {
                // 执行加载地图，此时的 Context.PendingNetGame->URL 已经时本地地图路径 
                const bool bLoadedMapSuccessfully = LoadMap(Context, Context.PendingNetGame->URL, Context.PendingNetGame, Error);
				if (Context.PendingNetGame != nullptr)
				{
				    // 加载完成,修改 bLoadedMapSuccessfully 为 true
		            Context.PendingNetGame->LoadMapCompleted(this, Context, bLoadedMapSuccessfully, Error)		    
				}
				else
				{
				    // 回到默认地图
				}
            }
        }
        
        if (Context.PendingNetGame && Context.PendingNetGame->bLoadedMapSuccessfully && (Context.OwningGameInstance == NULL || !Context.OwningGameInstance->DelayCompletionOfPendingNetGameTravel()))
        {
            if (!Context.PendingNetGame->HasFailedTravel() )
            {
                // 所有工作都完成后,调用 travel 完成
                Context.PendingNetGame->TravelCompleted(this, Context);
                Context.PendingNetGame = nullptr;
            }
        }
    }
}
```

由于 Travel 传入的 URL 是个 ds 的地址，所以需要连上服务器后才能拿到真正的地图名称或路径。而拿到真正地图名称的流程如下：

1. Client 在执行 `WorldContext.PendingNetGame->InitNetDriver()` 时，发送 `NMT_Hello` 消息
2. DS 收到后发送 `NMT_Challenge` 消息
3. Client 收到后发送 `NMT_Login` 消息
4. DS 收到后发送 `NMT_Welcome` 消息，该消息包含 LevelName 和 GameName 信息，其中 LevelName 对应的就是地图名称
5. Client 收到后读取 LevelName 并存入 PendingNetGame->URL 中。相关源码如下：

```c++
void UPendingNetGame::NotifyControlMessage(UNetConnection* Connection, uint8 MessageType, class FInBunch& Bunch)
{
    switch (MessageType)
    {
        case NMT_Welcome:
        {
            FString GameName;
            FString RedirectURL;
            // 将 LevelName 的值赋值到 URL.Map
            if (FNetControlMessage<NMT_Welcome>::Receive(Bunch, URL.Map, GameName, RedirectURL))
            {
                //......... 省略代码 ............
                // 修改变量为 true，使得 LoadMap 操作可以正常执行.
                bSuccessfullyConnected = true;
            }
        }
    }
}
```

## travel 完成

以下是 client 流程

```c++
void UPendingNetGame::TravelCompleted(UEngine* Engine, FWorldContext& Context)
{
    Engine->TransitionType = ETransitionType::Connecting;
    // 发送加入
    Context.PendingNetGame->SendJoin();
    //........ 省略代码 ..........
}
```

```c++
void UPendingNetGame::SendJoin()
{
    // 发送 NMT_Join 消息
    FNetControlMessage<NMT_Join>::Send(NetDriver->ServerConnection);
}
```

以下是 server 流程

```c++
void UWorld::NotifyControlMessage(UNetConnection* Connection, uint8 MessageType, class FInBunch& Bunch)
{
    if( NetDriver->ServerConnection )
    {
        switch (MessageType)
        {
            case NMT_Join:
            {
                if (Connection->PlayerController == NULL)
                {
                    // ..........省略代码
                    // 创建 PlayerController
                    Connection->PlayerController = SpawnPlayActor( Connection, ROLE_AutonomousProxy, InURL, Connection->PlayerId, ErrorMsg );
                    Connection->SetClientLoginState(EClientLoginState::ReceivedJoin);
                    // ..........省略代码                  
                }
            }
        }
    }
}
```

由此可知,join 成功的标志就是 PlayerController 被成功创建. 至此client 登录 ds 完成.