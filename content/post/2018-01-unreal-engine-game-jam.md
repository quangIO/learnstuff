---
title: Making multi-player space shooter game in 1 day with Unreal Engine
subtitle: Not really but close (Tutorial to get you start with multi-player, C++, and packaging)
date: 2018-04-04
tags: ["game development", "Unreal Engine", "C++"]
share_img: https://i.imgur.com/1BWFF8u.jpg
---


Last week 2 inexperienced "game dev" and I made a game for a 24 hour game jam at NYU thanks to UE4 (and artists who make their artwork publicly available). This is the demo:

<link rel="stylesheet" href="https://cdn.plyr.io/3.1.0/plyr.css">
<video id="player" playsinline controls>
    <source src="/media/ue4-demo.mp4" type="video/mp4">
</video>

I have seen a lot of people saying using Unity is easier than UE4 for small projects but to me UE4 interface is very understandable and makes me feel that *I am the one in control*. On the contrary, one of my teammate, after looking at UE4 for the first time, said he was confused. This tutorial will try to explain you how to have a "UE4" mindset: be lazy, don't try to know everything, just learn what you need.

# How to make it
(this overall write-up is intended for beginners)

### Think:
If you don't know how to start, let's break down the idea:

* We're going to make a topdown game, so **everything will happen in a same plane X.** We can create the depth for it, but it is just the background.

* Controll the spaceship: **thrust and change the direction.** To thrust, we need to add a force in forward direction. Change direction following the mouse that we have to get the cursor position on the plane X a.k.a raytracing.

* Players can shoot projectiles: first we **spawn the projectiles** *(genius!!!)*, and then the projectile need to fly to mouse position somehow. We can just **add a force** like we thrust the spaceship right?

* Gravity: we basically get all the plannets' positions and masses and **use high school physics** $$F = G \sum \frac {mM}{r^2} |r|$$

* Health and capacity: It would be more fun if players can die, run out of gas... right?

* Collision: When something hits something. **Applying/Receiving damages**.

* Fancy graphic: Let's use free assets, basic shapes with fancy materials. It works.

* Multi-player: This is the hardest part. In single player mode, when you shoot it just shoot. In multi-player, if you are a client, you should request the server first. *Hey server, can you shoot this for me?* the when the server receives this request, it will broadcast to all clients: *a projectile just spawned and have this property*. And after receiving message from server, all clients just need to spawn that projectile to keep things in sync.

Everything sounds easy so far. Let's get to work.

### Basic setup:
Well, trivial task, selecting Twin Stick Shooter template seems like no-brainer. For more flexible, it'd be better to start with C++ template because migrating Blueprint projects to C++ project is a tedious task but you don't really need to migrate from C++ to Blueprint. You can just use Blueprint if you think C++ is hard. However, trust me, writing game logic in C++ is beginner friendly too and it can be cleaner than spahetti Blueprint code in some cases.

### Ray-tracing
A simple trick is creating an invisible plane. This plane will not have any collision with game objects. It blocks `visibility` channel only so when we do the raytracing from the camera, we can have a hit event. Break this hit and we have the position. We can make it better by using a custom channel instead of `visibility`.

```c++
void AMyPlayerController::UpdateRotation()
{
	FHitResult HitResult;
	GetHitResultUnderCursorByChannel(TraceTypeQuery3, false, HitResult);
	if (HitResult.bBlockingHit)
	{
		APawn *ControlledPawn = GetPawn(); // because I implement this logic in player controller
		const FRotator rot = UKismetMathLibrary::FindLookAtRotation(ControlledPawn->GetActorLocation(), HitResult.Location);
		ControlledPawn->SetActorRotation(rot);
	}
}
```

Make it callable by Blueprint by adding `UFUNCTION(BlueprintCallable)`

```
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "MyPlayerController.generated.h"

/**
 *
 */
UCLASS()
class PLANETS_API AMyPlayerController : public APlayerController
{
	GENERATED_BODY()
public:
	UFUNCTION(BlueprintCallable)
	void UpdateRotation();

};

```


![Ray Tracing](/img/ray_tracing.png "Ray Tracing")



### Gravity:
I want some objects get affected by gravity, some won't. So let's create a component, every object having this component will get effected by gravity. We can just use Newton's formula for gravity. However, the world here is *unreal*, I don't trust the constant factor G, and the distant we see is just an illustion, so the formula is something like this where we can edit those parameters p_something: $$F = p_f \sum \frac {mM}{(r * p_r)^2} |r|$$

```
void UOrbitalComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
	FVector Force(.0f, .0f, .0f);
	for (auto &s : Stars)
	{
		FVector r = (s->GetActorLocation() - Parent->GetActorLocation()) * DistanceFactor;
		Force += s->mass * r.GetSafeNormal2D() / r.SizeSquared2D() * MeshComponent->GetMass();
	}

	MeshComponent->AddForce(ForceFactor * Force);
}
```

We don't want to find all the stars every frame. Let's just do it once when the game starts and cache the result in our custom game state.

```
ACustomGameState::ACustomGameState()
{

	TArray<AActor*> Temp;
	UGameplayStatics::GetAllActorsOfClass(this, AStarBase::StaticClass(), Temp);
	for (auto &t: Temp)
	{
		Stars.Add(Cast<AStarBase>(t));
	}
}
```

### Multiplayer
The structure looks like this but you should setup event type acordingly (To Server, Multicast)
<iframe src="https://blueprintue.com/render/60kflqgh" width="100%" height = "600" scrolling="no"></iframe>

This is the end for now. This blog will get update gradually for more detailed explanation when I am free if needed.

<script src="https://cdn.plyr.io/3.1.0/plyr.js"></script>
<script>const player = new Plyr('#player');</script>


