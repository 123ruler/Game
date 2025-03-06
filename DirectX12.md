## 资源绑定

为单个物体的shader绑定所需的参数

（如：shader需要物体的world矩阵）

### 创建常量缓冲区视图（CBV）

**1. 创建常量缓冲区视图描述符（CBV Descriptor）**

它描述了要查看的常量：常量在CB中的起始位置和占用的空间（假设CB中有n个常量，那么第i个常量起始地址为i*objCBByteSize）

1. 常量在UploadBuffer（CB）中的起始地址（cbAddress）：从objCB->GetGPUVirtualAddress()开始偏移

2. 常量占用的空间（objCBByteSize）：CalcConstantBufferByteSize(sizeof(ObjectConstants))

获得描述符在描述符堆中的句柄（CPU Descriptor Handle）

（从mCbvHeap->GetCPUDescriptorHandleForHeapStart()开始偏移）

**2. 用CBV Descriptor和CPU Descriptor Handle创建常量缓冲区视图（CBV）**

md3dDevice->CreateConstantBufferView(&cbvDesc, handle);



### 创建常量缓冲区视图描述符堆（Descriptor Heap）

**1. 创建堆的描述符（D3D12_DESCRIPTOR_HEAP_DESC）：**

1. 堆中的描述符数量

2. 堆的种类（CBV_SRV_UAV）

**2. 创建堆（ID3D12DescriptorHeap）：**

md3dDevice->CreateDescriptorHeap(&cbvHeapDesc 描述符堆的描述, IID_PPV_ARGS(&mCbvHeap) 描述符堆的GUID)



### 创建根签名（Root Signature）

**1. 创建根参数（CD3DX12_ROOT_PARAMETER）**

1. 创建描述符列表（CD3DX12_DESCRIPTOR_RANGE）

2. 用描述符列表初始化根参数：slotRootParameter[0].InitAsDescriptorTable(1, &table)

**2. 创建根签名描述符（CD3DX12_ROOT_SIGNATURE_DESC）**

​	描述了有几个根参数，根参数的引用

**3. 创建根签名**

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

XMMATRIX world = MathHelper::Identity4x4();

ObjectConstants objConstants;
XMStoreFloat4x4(&objConstants.World, XMMatrixTranspose(world));

currObjCB->CopyData(ObjCBIndex, objConstants);



### Draw

**1. 绑定资源，设置CBV描述符堆**

ID3D12DescriptorHeap* descriptorHeaps[] = { mCbvHeap.Get() };
    mCommandList->SetDescriptorHeaps(_countof(descriptorHeaps), descriptorHeaps);

**2. 设置根签名**

 mCommandList->SetGraphicsRootSignature(mRootSignature.Get());

**3. 设置根描述符表**

mCommandList->SetGraphicsRootDescriptorTable(0, mCbvHeap->GetGPUDescriptorHandleForHeapStart());

（参数：根参数槽位index，跟根签名声明的槽位对应，描述符堆地址handle）

### Shader

```hlsl
cbuffer cbPerObject : register(b0)
{
    float4x4 gWorld;
};

VertexOut VS(VertexIn vin)
{
    VertexOut vout;
	
    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosH = mul(posW, gViewProj);
    
    return vout;
}
```



## 绘制网格

### 构建网格

```c++
GeometryGenerator::MeshData box = geoGen.CreateBox(1.5f, 0.5f, 1.5f, 3);

UINT boxVertexOffset = 0;
UINT boxIndexOffset = 0;

std::vector<Vertex> vertices;
for (size_t i = 0; i < box.Vertices.size(); i++)
{
    Vertex v;
    v.Color = XMFLOAT4(DirectX::Colors::Red);
    v.Pos = box.Vertices[i].Position;
    vertices.push_back(v);
}
std::vector<std::uint16_t> indices;
indices.insert(indices.end(), std::begin(box.GetIndices16()), std::end(box.GetIndices16()));

const UINT vbByteSize = (UINT)vertices.size() * sizeof(Vertex);
const UINT ibByteSize = (UINT)indices.size() * sizeof(std::uint16_t);

auto geo = std::make_unique<MeshGeometry>();
geo->Name = "shapeGeo";

ThrowIfFailed(D3DCreateBlob(vbByteSize, &geo->VertexBufferCPU));
CopyMemory(geo->VertexBufferCPU->GetBufferPointer(), vertices.data(), vbByteSize);

ThrowIfFailed(D3DCreateBlob(ibByteSize, &geo->IndexBufferCPU));
CopyMemory(geo->IndexBufferCPU->GetBufferPointer(), indices.data(), ibByteSize);

geo->VertexBufferGPU = d3dUtil::CreateDefaultBuffer(md3dDevice.Get(), mCommandList.Get(), vertices.data(), vbByteSize, geo->VertexBufferUploader);

geo->IndexBufferGPU = d3dUtil::CreateDefaultBuffer(md3dDevice.Get(), mCommandList.Get(), indices.data(), ibByteSize, geo->IndexBufferUploader);

geo->VertexByteStride = sizeof(Vertex);
geo->VertexBufferByteSize = vbByteSize;
geo->IndexFormat = DXGI_FORMAT_R16_UINT;
geo->IndexBufferByteSize = ibByteSize;

SubmeshGeometry submeshBox;
submeshBox.IndexCount = (UINT)box.Indices32.size();
submeshBox.StartIndexLocation = boxIndexOffset;
submeshBox.BaseVertexLocation = boxVertexOffset;

geo->DrawArgs["box"] = submeshBox;
```



### Draw

**1. 顶点**

设置顶点，索引，拓扑类型

commandList->IASetVertexBuffers(0, 1, mGeo->VertexBufferView());
commandList->IASetIndexBuffer(mGeo->IndexBufferView());
commandList->IASetPrimitiveTopology(mPrimitiveType);

**2. 设置根描述符表**

指定需要绑定到流水线的资源，cbv中存有该物体的World矩阵

 auto cbvHandle = CD3DX12_GPU_DESCRIPTOR_HANDLE(mCbvHeap->GetGPUDescriptorHandleForHeapStart());
 cbvHandle.Offset(ObjCBIndex, mCbvSrvUavDescriptorSize);
 commandList->SetGraphicsRootDescriptorTable(0, cbvHandle);

**3. 绘制网格**

commandList->DrawIndexedInstanced(IndexCount, 1, StartIndexLocation, BaseVertexLocation, 0);



### 动态顶点

使用动态缓冲区，在CPU更新顶点数据，再通过上传缓冲区更新顶点数据

（用顶点上传缓冲区UploadBuffer\<Vertex>设置到geo的VertexBufferGPU）

**Initialize**

geo的VertexBufferCPU为空，geo的IndexBufferCPU和IndexBufferGPU还是保持原样

```c++
geo->VertexBufferCPU = nullptr;
geo->VertexBufferGPU = nullptr;

ThrowIfFailed(D3DCreateBlob(ibByteSize, &geo->IndexBufferCPU));
CopyMemory(geo->IndexBufferCPU->GetBufferPointer(), indices.data(), ibByteSize);

geo->IndexBufferGPU = d3dUtil::CreateDefaultBuffer(md3dDevice.Get(),
	mCommandList.Get(), indices.data(), ibByteSize, geo->IndexBufferUploader);
```

**Update**

geo的VertexBufferGPU设置为UploadBuffer\<Vertex>.Resource()

```c++
UploadBuffer<Vertex>* currWavesVB = mCurrFrameResource->WavesVB.get();
for(int i = 0; i < mWaves->VertexCount(); ++i)
{
	Vertex v;
	v.Pos = mWaves->Position(i);
	currWavesVB->CopyData(i, v);
}

mWavesRitem->Geo->VertexBufferGPU = currWavesVB->Resource();
```



## 渲染项

当绘制多个物体时，每个物体都是1个渲染项RenderItem

每个物体和其CBV的绑定是依赖commandList设置的顺序的，先设置cbv和其他数据，才可以用这些数据绘制网格及在物体shader中获取cbv数据

```c++
// 在Draw()中调用
void ShapesApp::DrawRenderItems(
    ID3D12GraphicsCommandList* cmdList, 
    const std::vector<RenderItem*>& ritems)
{
    UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));
 
	auto objectCB = mCurrFrameResource->ObjectCB->Resource();

    for(size_t i = 0; i < ritems.size(); ++i)
    {
        auto ri = ritems[i];

        // 设置第i个物体的网格数据
        cmdList->IASetVertexBuffers(0, 1, &ri->Geo->VertexBufferView());
        cmdList->IASetIndexBuffer(&ri->Geo->IndexBufferView());
        cmdList->IASetPrimitiveTopology(ri->PrimitiveType);

        // 设置第i个物体的cbv
        UINT cbvIndex = mCurrFrameResourceIndex * (UINT)mOpaqueRitems.size() + ri->ObjCBIndex;
        auto cbvHandle = CD3DX12_GPU_DESCRIPTOR_HANDLE(mCbvHeap->GetGPUDescriptorHandleForHeapStart());
        cbvHandle.Offset(cbvIndex, mCbvSrvUavDescriptorSize);
        cmdList->SetGraphicsRootDescriptorTable(0, cbvHandle);

        // 根据以上设置的数据，绘制1个物体
        cmdList->DrawIndexedInstanced(ri->IndexCount, 1, ri->StartIndexLocation, ri->BaseVertexLocation, 0);
    }
}
```



## XMFLOAT4X4 和 XMMATRIX

将矩阵传给Shader之前 要进行转置的原因：

有一个矩阵（线代）
$$
\left[
\matrix{
  a & b & c & d\\
  e & f & g & h\\
  i & j & k & l\\
  m & n & o & p
}
\right]
$$
行主，内存位置索引为：a b c d e f h i j k l m n o p

列主，内存位置索引为：a e i m b f j n c g k o d h l p

HLSL中使用右乘即V*M，按道理说行主矩阵应右乘，但HLSL又期望矩阵为列主（为了速度），所以HLSL中的矩阵乘法为：
$$
V*M = [V * M[1,2,3,4], …] (数字为内存位置索引)
	= [V * M[a,e,i,m], …]
$$
DX是行主序存储矩阵的，但将矩阵看作列主来读取（1,2,3,4），所以在传给shader之前先将矩阵转置，那么HLSL中可以以读取列主矩阵的方式读取到行主矩阵，获得行主矩阵后，就应该进行右乘计算

**总结：在平常shader计算时，可以将矩阵看成行主来右乘计算，但传给shader之前要将原先的行主矩阵进行转置**

opengl	右手坐标系 列向量 左乘 列主序存储矩阵
osg		 右手坐标系 行向量 右乘 行主序存储矩阵
d3d		 左手坐标系 行向量 右乘 行主序存储矩阵
ogre		右手坐标系 列向量 左乘 行主序存储矩阵
