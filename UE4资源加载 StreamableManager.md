# 从FStreamableManager::RequestAsyncLoad开始讲解UE4资源加载流程

## 前言
在UE4中有许许多多加载资源的接口，同步的有LoadObject，LoadClass，LoadPackage，StaticLoad系列。异步的有LoadPackageAsync，FStreamableManager::RequestAsyncLoad。有些接口是偏底层，有些接口是上层封装的。其中官方案例讲解异步加载就是用FStreamableManager::RequestAsyncLoad作为案例，本文也一样。
其实同步加载跟异步加载其实是殊途同归，同步加载也是发起异步加载请求，只不过在后面会马上进行flush，阻塞线程以马上完成这个异步加载的请求。

## 1.从FStreamableManager::RequestAsyncLoad开始
FStreamableManager::RequestAsyncLoad是比较上层的封装加载接口
```C++
TSharedPtr<FStreamableHandle> FStreamableManager::RequestAsyncLoad(const FSoftObjectPath& TargetToStream, FStreamableDelegate DelegateToCall, TAsyncLoadPriority Priority, bool bManageActiveHandle, bool bStartStalled, FString DebugName)
```
除了第一个资源加载路径，其他参数均可选，以下是调用案例
```C++
void AMyActor::BeginPlay()
{
	Super::BeginPlay();
	FStreamableDelegate Delegate;
	//AssetPath FString /Game/TestActor.TestActor_C
	FSoftObjectPath path(AssetPath);
	Delegate.BindLambda([this](FSoftObjectPath Objpath)
	{
		TSoftClassPtr<AActor> ObjectPtr = TSoftClassPtr<AActor>(Objpath);
		UClass* Cls = ObjectPtr.Get();
		GetWorld()->SpawnActor<AActor>(Cls,FVector(0,0,100),FRotator::ZeroRotator);
		UE_LOG(LogTemp,Log,TEXT("load finish %s"),*Objpath.ToString());
	},path);
	//sbm FStreamableManager 全局单例
	sbm.RequestAsyncLoad(path,Delegate);
}
```
![](https://raw.githubusercontent.com/txkxk/ImageBed/master/content.png?token=ADEBULES6ZQSJG32TNVIT7TBYFGVG)
关于 FSoftObjectPath,TSoftClassPtr(TSoftObjectPtr),StreamableManager 可参考文档
https://docs.unrealengine.com/4.27/zh-CN/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/

##2.创建异步加载请求
全部RequestAsyncLoad重载都会进入TArray\<FSoftObjectPath\>的版本。
在这个函数里，首先创建一个异步请求对象FStreamableHandle，存放请求的基本信息，如完成回调，资源路径，优先级等。
然后会经过一系列的路径合法性检查，最终调用StartHandleRequests开始请求。
在StartHandleRequests中会对每个请求资源调用StreamInternal。在StreamInternal中，会在内部创建Streamable对象并构建FSoftObjectPath到Streamable的映射关系。在创建/获取Streamable对象后，会把当前请求对象记录到Streamable对象里(Existing->AddLoadingRequest(Handle))。
由于当前请求的资源有可能已经被请求过，所以会检查所有的Streamable是否都已经成功加载完毕
```C++
void FStreamableManager::StartHandleRequests(TSharedRef<FStreamableHandle> Handle)
{
	TRACE_LOADTIME_REQUEST_GROUP_SCOPE(TEXT("StreamableManager - %s"), *Handle->GetDebugName());

	TArray<FStreamable *> ExistingStreamables;
	ExistingStreamables.Reserve(Handle->RequestedAssets.Num());

	for (int32 i = 0; i < Handle->RequestedAssets.Num(); i++)
	{
		FStreamable* Existing = StreamInternal(Handle->RequestedAssets[i], Handle->Priority, Handle);
		check(Existing);

		ExistingStreamables.Add(Existing);
		Existing->AddLoadingRequest(Handle);
	}

	// Go through and complete loading anything that's already in memory, this may call the callback right away
	for (int32 i = 0; i < Handle->RequestedAssets.Num(); i++)
	{
		FStreamable* Existing = ExistingStreamables[i];

		if (Existing && (Existing->Target || Existing->bLoadFailed))
		{
			Existing->bAsyncLoadRequestOutstanding = false;

			CheckCompletedRequests(Handle->RequestedAssets[i], Existing);
		}
	}
}
```
来到StreamInternal。先检查该资源是否已经被请求过，如果有并且已经加载完成就返回，没有就创建FStreamable对象
```C++
FStreamable* FStreamableManager::StreamInternal(const FSoftObjectPath& InTargetName, TAsyncLoadPriority Priority, TSharedRef<FStreamableHandle> Handle)
{
	check(IsInGameThread());
	UE_LOG(LogStreamableManager, Verbose, TEXT("Asynchronous load %s"), *InTargetName.ToString());

	FSoftObjectPath TargetName = ResolveRedirects(InTargetName);
	FStreamable* Existing = StreamableItems.FindRef(TargetName);
	if (Existing)
	{
		if (Existing->bAsyncLoadRequestOutstanding)
		{
			UE_LOG(LogStreamableManager, Verbose, TEXT("     Already in progress %s"), *TargetName.ToString());
			check(!Existing->Target); // should not be a load request unless the target is invalid
			ensure(IsAsyncLoading()); // Nothing should be pending if there is no async loading happening

			// Don't return as we potentially want to sync load it
		}
		if (Existing->Target)
		{
			UE_LOG(LogStreamableManager, Verbose, TEXT("     Already Loaded %s"), *TargetName.ToString());
			return Existing;
		}
	}
	else
	{
		Existing = StreamableItems.Add(TargetName, new FStreamable());
	}
```
检查资源是否已经在内存中，有则获取它。FindInMemory内部调用StaticFindObject
```C++
    if (!Existing->bAsyncLoadRequestOutstanding)
	{
		FindInMemory(TargetName, Existing);
	}
```
如果找不到，就进行加载。分两种情况，如果当前无法进行异步加载，就直接强制使用同步加载，调用StaticLoadObject。否则调用LoadPackageAsync请求资源异步加载。
```C++
    if (!Existing->Target)
	{
		// Disable failed flag as it may have been added at a later point
		Existing->bLoadFailed = false;

		FUObjectThreadContext& ThreadContext = FUObjectThreadContext::Get();

		// If async loading isn't safe or it's forced on, we have to do a sync load which will flush all async loading
		if (GIsInitialLoad || ThreadContext.IsInConstructor > 0 || bForceSynchronousLoads)
		{
                    //...
                    //同步加载
			Existing->Target = StaticLoadObject(UObject::StaticClass(), nullptr, *TargetName.ToString());
                    //...
                }
    else
    {
        //...
        //异步加载
        int32 RequestId = LoadPackageAsync(Package, FLoadPackageAsyncDelegate::CreateSP(Handle, &FStreamableHandle::AsyncLoadCallbackWrapper, TargetName), Priority);
    }
    return Existing;
```
```C++
int32 LoadPackageAsync(const FString& InName, const FGuid* InGuid /*= nullptr*/, const TCHAR* InPackageToLoadFrom /*= nullptr*/, FLoadPackageAsyncDelegate InCompletionDelegate /*= FLoadPackageAsyncDelegate()*/, EPackageFlags InPackageFlags /*= PKG_None*/, int32 InPIEInstanceID /*= INDEX_NONE*/, int32 InPackagePriority /*= 0*/, const FLinkerInstancingContext* InstancingContext /*=nullptr*/)
{
	LLM_SCOPE(ELLMTag::AsyncLoading);
	return GetAsyncPackageLoader().LoadPackage(InName, InGuid, InPackageToLoadFrom, InCompletionDelegate, InPackageFlags, InPIEInstanceID, InPackagePriority, InstancingContext);
}

int32 LoadPackageAsync(const FString& PackageName, FLoadPackageAsyncDelegate CompletionDelegate, int32 InPackagePriority /*= 0*/, EPackageFlags InPackageFlags /*= PKG_None*/, int32 InPIEInstanceID /*= INDEX_NONE*/)
{
	const FGuid* Guid = nullptr;
	const TCHAR* PackageToLoadFrom = nullptr;
	return LoadPackageAsync(PackageName, Guid, PackageToLoadFrom, CompletionDelegate, InPackageFlags, InPIEInstanceID, InPackagePriority );
}
```
GetAsyncPackageLoader会按照当前不同的平台返回不同的AsyncPackageLoader,然后调用AsyncPackageLoader::LoadPackage方法。
来到AsyncPackageLoader::LoadPackage，首先会经历一连串关于传入路径的处理和路径合法性的判断。最终在确认该路径合法之后，会广播OnAsyncLoadPackage事件，表示该资源正式开始异步加载。并且生成RequestID，创建FAsyncPackageDesc对象用于描述一个异步加载所需的全部内容。
```C++
struct FAsyncPackageDesc
{
	/** Handle for the caller */
	int32 RequestID;
	/** Name of the UPackage to create. */
	FName Name;
	/** Name of the package to load. */
	FName NameToLoad;
	/** GUID of the package to load, or the zeroed invalid GUID for "don't care" */
	FGuid Guid;
	/** Delegate called on completion of loading. This delegate can only be created and consumed on the game thread */
	TUniquePtr<FLoadPackageAsyncDelegate> PackageLoadedDelegate;
	/** The flags that should be applied to the package */
	EPackageFlags PackageFlags;
	/** Package loading priority. Higher number is higher priority. */
	TAsyncLoadPriority Priority;
	/** PIE instance ID this package belongs to, INDEX_NONE otherwise */
	int32 PIEInstanceID;

    //...
}
```
```C++
int32 FAsyncLoadingThread::LoadPackage(const FString& InName, const FGuid* InGuid, const TCHAR* InPackageToLoadFrom, FLoadPackageAsyncDelegate InCompletionDelegate, EPackageFlags InPackageFlags, int32 InPIEInstanceID, int32 InPackagePriority, const FLinkerInstancingContext* InstancingContext)
{
    FString PackageName;
    bool bValidPackageName = true;
    //...InName合法性检查以及各种路径转换，按照InName生成PackageName和PackageNameToLoad
    if (bValidPackageName)
	{
		if ( FCoreDelegates::OnAsyncLoadPackage.IsBound() )
		{
			FCoreDelegates::OnAsyncLoadPackage.Broadcast(InName);
		}

		// Generate new request ID and add it immediately to the global request list (it needs to be there before we exit
		// this function, otherwise it would be added when the packages are being processed on the async thread).
		RequestID = GPackageRequestID.Increment();
		TRACE_LOADTIME_BEGIN_REQUEST(RequestID);
		AddPendingRequest(RequestID);

		// Allocate delegate on Game Thread, it is not safe to copy delegates by value on other threads
		TUniquePtr<FLoadPackageAsyncDelegate> CompletionDelegatePtr;
		if (InCompletionDelegate.IsBound())
		{
			CompletionDelegatePtr = MakeUnique<FLoadPackageAsyncDelegate>(MoveTemp(InCompletionDelegate));
		}

		// Add new package request
		FAsyncPackageDesc PackageDesc(RequestID, *PackageName, *PackageNameToLoad, InGuid ? *InGuid : FGuid(), MoveTemp(CompletionDelegatePtr), InPackageFlags, InPIEInstanceID, InPackagePriority);
		if (InstancingContext)
		{
			PackageDesc.SetInstancingContext(*InstancingContext);
		}
		QueuePackage(PackageDesc);
	}
	else
	{
		InCompletionDelegate.ExecuteIfBound(FName(*InName), nullptr, EAsyncLoadingResult::Failed);
	}

	return RequestID;
}
```
在创建完成FAsyncPackageDesc对象后，调用QueuePackage把该对象插入加载队列里。之后在加载线程的Tick函数(FAsyncLoadingThread::TickAsyncThread)里会调用CreateAsyncPackagesFromQueue，对所有请求调用ProcessAsyncPackageRequest，把该请求转化成FAsyncPackage对象，并且调用InsertPackage把该Package插入到加载队列(AsyncPackages)里。
```C++
EAsyncPackageState::Type FAsyncLoadingThread::TickAsyncThread(bool bUseTimeLimit, bool bUseFullTimeLimit, float TimeLimit, bool& bDidSomething, FFlushTree* FlushTree)
{
	check(!IsInGameThread() || !IsMultithreaded());
	EAsyncPackageState::Type Result = EAsyncPackageState::Complete;
	if (!bShouldCancelLoading)
	{
		int32 ProcessedRequests = 0;
		double TickStartTime = FPlatformTime::Seconds();
		if (AsyncThreadReady.GetValue())
		{
			if (GIsInitialLoad && GEventDrivenLoaderEnabled)
			{
				EDLBootNotificationManager.FireCompletedCompiledInImports();
			}
			{
				FGCScopeGuard GCGuard;
				//把FAsyncPackageDesc对象转成FAsyncPackage并插入到队列里，并做去重等处理
				CreateAsyncPackagesFromQueue(bUseTimeLimit, bUseFullTimeLimit, TimeLimit, FlushTree);
			}
			float TimeUsed = (float)(FPlatformTime::Seconds() - TickStartTime);
			const float RemainingTimeLimit = FMath::Max(0.0f, TimeLimit - TimeUsed);
			//如果这一帧要GC，则不进行任何加载
			if (IsGarbageCollectionWaiting() || (RemainingTimeLimit <= 0.0f && bUseTimeLimit && !IsMultithreaded()))
			{
				Result = EAsyncPackageState::TimeOut;
			}
			else
			{
				FGCScopeGuard GCGuard;
				//真正执行加载，处理在CreateAsyncPackagesFromQueue函数里插入到队列里的对象
				Result = ProcessAsyncLoading(ProcessedRequests, bUseTimeLimit, bUseFullTimeLimit, RemainingTimeLimit, FlushTree);
				bDidSomething = bDidSomething || ProcessedRequests > 0;
			}
		}

		if (ProcessedRequests == 0 && IsMultithreaded() && Result == EAsyncPackageState::Complete)
		{
			uint32 WaitTime = 30;
			if (IsEventDrivenLoaderEnabled())
			{
				if (!EDLBootNotificationManager.IsWaitingForSomething() && !(IsGarbageCollectionWaiting() || IsGarbageCollecting()))
				{
					CheckForCycles();
					IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("CheckForCycles (non-shipping)"));
				}
				else
				{
					WaitTime = 1; // we are waiting for the game thread to deal with the boot constructors, so lets spin tighter
				}
			}
			const bool bIgnoreThreadIdleStats = true;
			SCOPED_LOADTIMER(Package_Temp3);
			QueuedRequestsEvent->Wait(WaitTime, bIgnoreThreadIdleStats);
		}

	}
	else
	{
		// Blocks main thread
		double TickStartTime = FPlatformTime::Seconds();
		CancelAsyncLoadingInternal();
		IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("CancelAsyncLoadingInternal"));
		bShouldCancelLoading = false;
	}

#if LOOKING_FOR_PERF_ISSUES
	// Update stats
	SET_FLOAT_STAT( STAT_AsyncIO_AsyncLoadingBlockingTime, FPlatformTime::ToSeconds( BlockingCycles.GetValue() ) );
	BlockingCycles.Set( 0 );
#endif

	return Result;
}
```
在ProcessAsyncLoading函数中会处理由CreateAsyncPackagesFromQueue插入到加载队列里的FAsyncPackage对象，对队列里的每个FAsyncPackage对象调用TickAsyncPackage开始加载并且查询其加载进度，走到哪个stage。全部stage完成后会调用完成回调，并且移出加载队列，至此算加载完成了一个资源。关于加载资源都有哪些stage，为什么要进行这些stage，就涉及到UE4的资源序列化问题，可参考文章。
https://zhuanlan.zhihu.com/p/357904199
https://www.dazhuanlan.com/mazijun/topics/1069642

## 小结
UE4的异步加载可以看作是一个经典的“生产者-消费者”模式，由游戏主线程生产加载请求，异步加载线程消费请求，QueuedPackages为工作队列。
![](https://raw.githubusercontent.com/txkxk/ImageBed/master/a.jpg?token=ADEBULAUWKCUGJP3KQXMP3TBYFGVE)

