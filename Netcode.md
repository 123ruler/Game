# 网络NGO

- 服务端权威是指游戏的逻辑和状态都由服务端来控制和同步，客户端只负责接收服务端的数据并展示给用户，或者向服务端发送用户的操作请求。这种模式的优点是可以保证游戏的数据一致性和安全性，防止客户端作弊或篡改数据，适用于对精确性和反作弊要求较高的游戏，比如射击类、竞技类、策略类等。缺点是会增加服务端的压力和成本，以及网络延迟和丢包的影响。
- 客户端权威是指游戏的逻辑和状态都由客户端来控制和同步，服务端只负责转发客户端的数据或提供一些辅助功能。这种模式的优点是可以减少服务端的压力和成本，以及网络延迟和丢包的影响，适用于对网络延迟、运维成本和复杂性要求较低的游戏，比如合作类、休闲类、冒险类等。缺点是会降低游戏的数据一致性和安全性，容易出现客户端作弊或篡改数据的情况。



### NetworkObject

NetworkObjectId：两个客户端各自生成1个玩家，两个端都是玩家一id=1，玩家二id=2

可以在所有端中找到同一个副本（玩家一在机器1生成控制的，但在其他机器上也有玩家一的副本，通过id可以在所有机找到玩家一本身及其副本）



网络通信就是 同NetworkObjectId的网络组件通信 或 同NetworkObjectId的网络组件同属性字段同步



RPC找到对应NetworkObjectId的对应NtworkBehaviour ID对象，执行RPC方法





### NetworkBehaviour

继承这个类，可以使用各种关于网络的属性和方法



电脑1的P1和P2：

<img src="Assets\Netcode\image-20230610230221936.png" alt="image-20230610230221936" style="zoom:90%;" /> <img src="D:\我的配置\笔记图片\Unity\image-20230610230255315.png" alt="image-20230610230255315" style="zoom:90%;" /> 

电脑2的P1和P2：

<img src="D:\我的配置\笔记图片\Unity\image-20230610230347322.png" alt="image-20230610230347322" style="zoom:90%;" /> <img src="D:\我的配置\游戏\Game\Assets\Netcode\image-20230610230414329.png" alt="image-20230610230414329" style="zoom:90%;" /> 



- **isOwner**：当前实例是否属于当前客户端

  如电脑1的P1的IsOwner=true，电脑2的P2的IsOwner=true

  每个 `NetworkObject` 都有一个所有者。所有者通常是创建该对象的客户端。例如，当客户端创建一个新的玩家对象并将其同步到服务器和其他客户端时，该客户端就成为该玩家对象的所有者。

  

  玩家类移动，当前播放器控制当前玩家的移动（玩家1播放器控制玩家1移动，玩家2播放器执行玩家1的Update方法时会返回）

  ```csharp
  private void Update() {
      // 只在本地播放器中执行Update
      if (!IsOwner) return;
  
      HandleMovement();
      HandleInteractions();
  }
  ```

  

- NetworkBehaviour.IsServer：当前实例是否运行在Server端（与NetworkManager.Singleton.IsServer的值相同）

  如Host端的P1和P2的IsServer=true，Client端的的P1和P2的IsServer=false

  （IsClient同理：是否运行在Client端）

  

- NetworkObject.Despawn() ：从所有客户端中销毁当前游戏对象，但在服务器中该对象还存在在内存中（但从场景中移除，要彻底删除传入参数true）



### NetworkTransform

让两端同步某个游戏对象的Transform，这样两边运动对方才能看到

为Server权限，由服务器检测Transform更改，并推送到客户端。

Server端控制移动，游戏中对象才会移动，Client端无法使其对象移动

![image-20230529220846012](D:\我的配置\游戏\Game\Assets\Netcode\image-20230529220846012.png) 

### ClientNetworkTransform

让客户端将Transform的变化推送出去，所有客户端控制自身的Transform

```csharp
using Unity.Netcode.Components;
using UnityEngine;

namespace Unity.Multiplayer.Samples.Utilities.ClientAuthority
{
    [DisallowMultipleComponent]
    public class ClientNetworkTransform : NetworkTransform
    {
        protected override bool OnIsServerAuthoritative()
        {
            return false;
        }
    }
}
```



### NetworkAnimator

将当前网络对象的动画同步到其他端，加上这个玩家能看到其他玩家的动画

默认为服务器权威

![image-20230530142937692](D:\我的配置\游戏\Game\Assets\Netcode\image-20230530142937692.png) 

改成所有者权威：

### OwnerNetworkAnimator

```csharp
public class OwnerNetworkAnimator : NetworkAnimator
{
    protected override bool OnIsServerAuthoritative()
    {
        return false;
    }
}
```

并且在设置动画参数的时候，仅由所有者设置本身的参数（不然可能当B触发动画时，A的播放器中的B没触发动画，所以A中会设置B是没触发动画的状态，会影响到B）客户端对不属于自己的对象进行操作或更新，会造成数据不一致或者冲突的问题

```csharp
private void Update() {
    // 只在所有者的播放器执行
    if (!IsOwner) return;

    animator.SetBool(IS_WALKING, player.IsWalking());
}
```



NetworkObject

父对象里面有了，子对象就可以不用加



ClientRpc

在**服务端调用**ClientRpc方法（非客户端调用会报错），所有**客户端执行**该方法

生成菜单的行为希望在服务端执行（保证数据精确同步），然后客户端获取服务端生成的数据

（注意该方法，当host先进入，这个菜单就会开始生成，如果生成了一些菜单后，其他客户端才加入，只能获取到加入后的菜单，获取不到加入前的菜单，所以要设计成所有客户端准备就绪才能开始游戏）

```csharp
private void Update() {
    // 由服务端控制菜单生成
    if (!IsServer) return;

    spawnRecipeTimer -= Time.deltaTime;
    if (spawnRecipeTimer <= 0f) {
        spawnRecipeTimer = spawnRecipeTimerMax;

        if (KitchenGameManager.Instance.IsGamePlaying() && waitingRecipeSOList.Count < waitingRecipesMax) {
            // 服务端生成新菜单
            int waitingRecipeSOIndex = UnityEngine.Random.Range(0, recipeListSO.recipeSOList.Count);

            // 服务端调用，客户端执行
            SpawnNewWaitingRecipeClientRPC(waitingRecipeSOIndex);
        }
    }
}

[ClientRpc]
private void SpawnNewWaitingRecipeClientRPC(int waitingRecipeSOIndex)
{
    // 所有客户端执行此处代码
    // 客户端获取到服务端生成的waitingRecipeSOIndex，进行显示菜单等操作
    // （每个客户端都有个waitingRecipeSOList，这一步将waitingRecipeSO加到所有客户端的list中，所有客户端保持同步，这个像是一种”手动同步“，而不是依靠一些networkTransform那种组件帮助同步所有客户端的变换）
    RecipeSO waitingRecipeSO = recipeListSO.recipeSOList[waitingRecipeSOIndex];
    waitingRecipeSOList.Add(waitingRecipeSO);

    OnRecipeSpawned?.Invoke(this, EventArgs.Empty);
}
```



经过server转发到client

ClientRpc和ServerRPC会在**本机器本对象**执行，和在**其他机器的本对象**执行（如P1在机器1射击，会在机器1的P1对象上执行，和在其他机器上的P1对象上执行），ClientRpc是以调用对象为“作用域”的

这样可以在ServerRPC中检测子弹是否合法（服务端校验），可防外挂，缺点是可能客户端没能立刻看到子弹生成（因为还要经过server转发）

ClientRpc和ServerRPC会根据NetworkObject的IsServer属性，判断当前游戏对象是为服务端对象还是客户端对象，决定能否执行ClientRpc和ServerRPC

以下挂载在Player上，机器1、2有P1、P2两个对象，现在在机器1上P1射击

- 机器1上只有P1调用ServerRpc，P1也是Server所以在机器1的P1上执行ServerRPC，所以在机器1的P1上调用和执行ClientRPC（机器1只输出1个客户端收到）
- 机器2上只有P1执行ServerRPC（P1对象是服务端对象），所以在机器2的P1上调用和执行ClientRPC（机器1只输出1个客户端收到）

```csharp
public class PTest : NetworkBehaviour
{
    private void Update()
    {
        // owner射击
        if (!IsOwner) return;

        // 射击
        if (Input.GetMouseButtonDown(0))
        {
            // 告诉服务端生成子弹
            Debug.Log("客户端发送：" + Time.time);
            ShootServerRPC(Time.time);
        }
    }

    [ServerRpc]
    private void ShootServerRPC(float value)
    {
        // 告诉客户端生成子弹
        ShootClientRPC(value);
    }

    [ClientRpc]
    private void ShootClientRPC(float value)
    {
        Debug.Log("客户端收到：" + value);
    }
}
```

