## 资源绑定

为单个物体的shader绑定所需的参数

（如：shader需要物体的world矩阵）

### 创建常量缓冲区视图描述符堆（Descriptor Heap）

**创建堆的描述符（D3D12_DESCRIPTOR_HEAP_DESC）：**

1. 堆中的描述符数量

2. 堆的种类（CBV_SRV_UAV）

**创建堆（ID3D12DescriptorHeap）：**

CreateDescriptorHeap(&cbvHeapDesc 描述符堆的描述, IID_PPV_ARGS(&mCbvHeap) 描述符堆的GUID)



### 创建常量缓冲区视图（CBV）

**创建常量缓冲区视图描述符（CBV Descriptor）**

它描述了要查看的常量：常量在CB中的起始位置和占用的空间（假设CB中有n个常量，那么第i个常量起始地址为i*objCBByteSize）

1. 常量在UploadBuffer（CB）中的起始地址（cbAddress）：从objCB->GetGPUVirtualAddress()开始偏移

2. 常量占用的空间（objCBByteSize）：CalcConstantBufferByteSize(sizeof(ObjectConstants))

**获得描述符在描述符堆中的句柄（CPU Descriptor Handle）**

（从mCbvHeap->GetCPUDescriptorHandleForHeapStart()开始偏移）

**用CBV Descriptor和CPU Descriptor Handle创建常量缓冲区视图（CBV）**

md3dDevice->CreateConstantBufferView(&cbvDesc, handle);



### 创建根签名（Root Signature）

**创建根参数（CD3DX12_ROOT_PARAMETER）**

1. 创建描述符列表（CD3DX12_DESCRIPTOR_RANGE）

2. 用描述符列表初始化根参数：slotRootParameter[0].InitAsDescriptorTable(1, &table)

**创建根签名描述符（CD3DX12_ROOT_SIGNATURE_DESC）**

​	描述了有几个根参数，根参数的引用

**创建根签名**

1. 序列化根签名

​	D3D12SerializeRootSignature(
​        &rootSigDesc,
​        D3D_ROOT_SIGNATURE_VERSION_1,
​        serializedRootSig.GetAddressOf(),
​        errorBlob.GetAddressOf());

2. 创建根签名（ID3D12RootSignature）

​	md3dDevice->CreateRootSignature(
​        0,
​        serializedRootSig->GetBufferPointer(),
​        serializedRootSig->GetBufferSize(),
​        IID_PPV_ARGS(&mRootSignature))



### Update

**更新常量缓冲区**

更新常量缓存区指定索引的变量（currObjCB是UploadBuffer\<ObjectConstants>）

currObjCB->CopyData(ObjCBIndex, objConstants);



### Draw

**绑定资源，设置CBV描述符堆**

ID3D12DescriptorHeap* descriptorHeaps[] = { mCbvHeap.Get() };
    mCommandList->SetDescriptorHeaps(_countof(descriptorHeaps), descriptorHeaps);

**设置根签名**

 mCommandList->SetGraphicsRootSignature(mRootSignature.Get());

**设置根描述符表**

mCommandList->SetGraphicsRootDescriptorTable(0, mCbvHeap->GetGPUDescriptorHandleForHeapStart());

（参数：根参数槽位index，跟根签名声明的槽位对应，描述符堆地址handle）
