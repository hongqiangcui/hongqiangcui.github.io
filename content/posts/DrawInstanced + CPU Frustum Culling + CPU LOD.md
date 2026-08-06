---
title: "DrawInstanced + CPU Frustum Culling + CPU LOD (Win32 + DirectX12)"

---

<!--more-->

```
#include <windows.h>
#include <wrl.h>
#include <dxgi1_6.h>
#include <d3d12.h>
#include <d3dcompiler.h>
#include <DirectXMath.h>
#include <d3dx12.h>

#include <cstdint>
#include <cstring>
#include <stdexcept>
#include <string>
#include <vector>
#include <cmath>

using Microsoft::WRL::ComPtr;
using namespace DirectX;

static const UINT FrameCount = 2;
static const UINT Width = 1280;
static const UINT Height = 720;

static const UINT GridW = 100;
static const UINT GridH = 100;
static const UINT InstanceCount = GridW * GridH;

static void ThrowIfFailed(HRESULT hr)
{
    if (FAILED(hr))
        throw std::runtime_error("D3D12 call failed.");
}

static UINT Align256(UINT size)
{
    return (size + 255u) & ~255u;
}

struct Vertex
{
    float pos[3];
    float color[4];
};

struct SourceInstance
{
    float offset[2];
};

struct RenderInstance
{
    float offset[2];
    float scale;
    float pad0;
    float color[4];
};

struct alignas(256) FrameCB
{
    XMFLOAT4X4 ViewProj;
    XMFLOAT4 Camera;   // x, y, frustumHalfW, frustumHalfH
    XMFLOAT4 Params;   // x = reserved, y = reserved, z = reserved, w = reserved
};

class App
{
public:
    int Run(HINSTANCE hInst, int nCmdShow)
    {
        InitWindow(hInst, nCmdShow);
        InitD3D();
        MainLoop();
        WaitForGpu();
        if (m_fenceEvent) CloseHandle(m_fenceEvent);
        return 0;
    }

private:
    void GetCamera(float& camX, float& camY) const
    {
        float t = static_cast<float>(GetTickCount64()) * 0.001f;
        camX = std::sin(t * 0.6f) * 3.0f;
        camY = std::cos(t * 0.4f) * 2.0f;
    }

    void InitWindow(HINSTANCE hInst, int nCmdShow)
    {
        WNDCLASSEXW wc = {};
        wc.cbSize = sizeof(wc);
        wc.style = CS_HREDRAW | CS_VREDRAW;
        wc.lpfnWndProc = WindowProc;
        wc.hInstance = hInst;
        wc.hCursor = LoadCursor(nullptr, IDC_ARROW);
        wc.lpszClassName = L"DX12DrawInstancedCpuCullLodSample";

        RegisterClassExW(&wc);

        RECT r = { 0, 0, (LONG)Width, (LONG)Height };
        AdjustWindowRect(&r, WS_OVERLAPPEDWINDOW, FALSE);

        m_hwnd = CreateWindowW(
            wc.lpszClassName,
            L"D3D12 DrawInstanced + CPU Frustum Culling + CPU LOD",
            WS_OVERLAPPEDWINDOW,
            CW_USEDEFAULT, CW_USEDEFAULT,
            r.right - r.left, r.bottom - r.top,
            nullptr, nullptr, hInst, this);

        ShowWindow(m_hwnd, nCmdShow);
        UpdateWindow(m_hwnd);
    }

    void InitD3D()
    {
#if defined(_DEBUG)
        {
            ComPtr<ID3D12Debug> debug;
            if (SUCCEEDED(D3D12GetDebugInterface(IID_PPV_ARGS(&debug))))
            {
                debug->EnableDebugLayer();
            }
        }
#endif

        ThrowIfFailed(CreateDXGIFactory1(IID_PPV_ARGS(&m_factory)));

        ThrowIfFailed(D3D12CreateDevice(
            nullptr, D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&m_device)));

        {
            D3D12_COMMAND_QUEUE_DESC q = {};
            q.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
            ThrowIfFailed(m_device->CreateCommandQueue(&q, IID_PPV_ARGS(&m_queue)));
        }

        {
            DXGI_SWAP_CHAIN_DESC1 sc = {};
            sc.BufferCount = FrameCount;
            sc.Width = Width;
            sc.Height = Height;
            sc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
            sc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
            sc.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
            sc.SampleDesc.Count = 1;

            ComPtr<IDXGISwapChain1> sc1;
            ThrowIfFailed(m_factory->CreateSwapChainForHwnd(
                m_queue.Get(), m_hwnd, &sc, nullptr, nullptr, &sc1));
            ThrowIfFailed(sc1.As(&m_swapChain));
            ThrowIfFailed(m_factory->MakeWindowAssociation(m_hwnd, DXGI_MWA_NO_ALT_ENTER));

            m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();
        }

        {
            D3D12_DESCRIPTOR_HEAP_DESC desc = {};
            desc.NumDescriptors = FrameCount;
            desc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
            ThrowIfFailed(m_device->CreateDescriptorHeap(&desc, IID_PPV_ARGS(&m_rtvHeap)));
            m_rtvInc = m_device->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);

            CD3DX12_CPU_DESCRIPTOR_HANDLE h(m_rtvHeap->GetCPUDescriptorHandleForHeapStart());
            for (UINT i = 0; i < FrameCount; ++i)
            {
                ThrowIfFailed(m_swapChain->GetBuffer(i, IID_PPV_ARGS(&m_backBuffers[i])));
                m_device->CreateRenderTargetView(m_backBuffers[i].Get(), nullptr, h);
                h.Offset(1, m_rtvInc);
            }
        }

        for (UINT i = 0; i < FrameCount; ++i)
        {
            ThrowIfFailed(m_device->CreateCommandAllocator(
                D3D12_COMMAND_LIST_TYPE_DIRECT, IID_PPV_ARGS(&m_allocators[i])));
        }

        CreateRootSignature();
        CreateShadersAndPSO();
        CreateGeometry();
        CreateSourceInstances();
        CreateConstantBuffer();
        CreateInstanceUploadBuffer();

        ThrowIfFailed(m_device->CreateCommandList(
            0,
            D3D12_COMMAND_LIST_TYPE_DIRECT,
            m_allocators[m_frameIndex].Get(),
            m_pso.Get(),
            IID_PPV_ARGS(&m_cmdList)));
        ThrowIfFailed(m_cmdList->Close());

        ThrowIfFailed(m_device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&m_fence)));
        m_fenceValue = 1;
        m_fenceEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
        if (!m_fenceEvent)
        {
            ThrowIfFailed(HRESULT_FROM_WIN32(GetLastError()));
        }

        m_viewport = CD3DX12_VIEWPORT(0.0f, 0.0f, static_cast<float>(Width), static_cast<float>(Height));
        m_scissor = CD3DX12_RECT(0, 0, static_cast<LONG>(Width), static_cast<LONG>(Height));
    }

    void CreateRootSignature()
    {
        CD3DX12_ROOT_PARAMETER params[1];
        params[0].InitAsConstantBufferView(0);

        CD3DX12_ROOT_SIGNATURE_DESC desc;
        desc.Init(1, params, 0, nullptr, D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

        ComPtr<ID3DBlob> sig, err;
        ThrowIfFailed(D3D12SerializeRootSignature(
            &desc, D3D_ROOT_SIGNATURE_VERSION_1, &sig, &err));

        ThrowIfFailed(m_device->CreateRootSignature(
            0, sig->GetBufferPointer(), sig->GetBufferSize(), IID_PPV_ARGS(&m_rootSig)));
    }

    void CreateShadersAndPSO()
    {
        const char* shaderText = R"(
cbuffer FrameCB : register(b0)
{
    row_major float4x4 g_ViewProj;
    float4 g_Camera;
    float4 g_Params;
};

struct VSIn
{
    float3 pos      : POSITION;
    float4 color    : COLOR;
    float2 instance : INSTANCEPOS;
    float  scale    : INSTANCESCALE;
    float4 instCol  : INSTANCECOLOR;
};

struct VSOut
{
    float4 pos   : SV_POSITION;
    float4 color : COLOR;
};

VSOut VSMain(VSIn i)
{
    VSOut o;
    float3 local = i.pos * i.scale;
    float3 world = local + float3(i.instance.xy, 0.0f);
    o.pos = mul(float4(world, 1.0f), g_ViewProj);
    o.color = i.color * i.instCol;
    return o;
}

float4 PSMain(VSOut i) : SV_Target
{
    return i.color;
}
)";

        ComPtr<ID3DBlob> vs, ps, err;
        ThrowIfFailed(D3DCompile(
            shaderText, std::strlen(shaderText), nullptr, nullptr, nullptr,
            "VSMain", "vs_5_0", 0, 0, &vs, &err));
        ThrowIfFailed(D3DCompile(
            shaderText, std::strlen(shaderText), nullptr, nullptr, nullptr,
            "PSMain", "ps_5_0", 0, 0, &ps, &err));

        D3D12_INPUT_ELEMENT_DESC layout[] =
        {
            { "POSITION",    0, DXGI_FORMAT_R32G32B32_FLOAT,    0,  0, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,   0 },
            { "COLOR",       0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,   0 },
            { "INSTANCEPOS", 0, DXGI_FORMAT_R32G32_FLOAT,       1,  0, D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },
            { "INSTANCESCALE",0, DXGI_FORMAT_R32_FLOAT,         1,  8, D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },
            { "INSTANCECOLOR",0, DXGI_FORMAT_R32G32B32A32_FLOAT, 1, 16, D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },
        };

        D3D12_GRAPHICS_PIPELINE_STATE_DESC pso = {};
        pso.InputLayout = { layout, _countof(layout) };
        pso.pRootSignature = m_rootSig.Get();
        pso.VS = { vs->GetBufferPointer(), vs->GetBufferSize() };
        pso.PS = { ps->GetBufferPointer(), ps->GetBufferSize() };
        pso.RasterizerState = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
        pso.BlendState = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
        pso.DepthStencilState.DepthEnable = FALSE;
        pso.DepthStencilState.StencilEnable = FALSE;
        pso.SampleMask = UINT_MAX;
        pso.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
        pso.NumRenderTargets = 1;
        pso.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;
        pso.SampleDesc.Count = 1;

        ThrowIfFailed(m_device->CreateGraphicsPipelineState(&pso, IID_PPV_ARGS(&m_pso)));
    }

    void CreateGeometry()
    {
        const Vertex vertices[] =
        {
            { { -0.25f,  0.25f, 0.0f }, { 0.85f, 0.90f, 0.85f, 1.f } }, // TL
            { {  0.25f,  0.25f, 0.0f }, { 0.85f, 0.90f, 0.85f, 1.f } }, // TR
            { { -0.25f, -0.25f, 0.0f }, { 0.75f, 0.85f, 0.75f, 1.f } }, // BL
            { {  0.25f, -0.25f, 0.0f }, { 0.75f, 0.85f, 0.75f, 1.f } }, // BR
        };

        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(sizeof(vertices)),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&m_vertexBuffer)));

        void* mapped = nullptr;
        CD3DX12_RANGE range(0, 0);
        ThrowIfFailed(m_vertexBuffer->Map(0, &range, &mapped));
        memcpy(mapped, vertices, sizeof(vertices));
        m_vertexBuffer->Unmap(0, nullptr);

        m_vbView.BufferLocation = m_vertexBuffer->GetGPUVirtualAddress();
        m_vbView.StrideInBytes = sizeof(Vertex);
        m_vbView.SizeInBytes = sizeof(vertices);
    }

    void CreateSourceInstances()
    {
        m_sourceInstances.reserve(InstanceCount);

        const float spacing = 0.6f;
        const float centerX = (GridW - 1) * 0.5f;
        const float centerY = (GridH - 1) * 0.5f;

        for (UINT y = 0; y < GridH; ++y)
        {
            for (UINT x = 0; x < GridW; ++x)
            {
                SourceInstance s;
                s.offset[0] = (static_cast<float>(x) - centerX) * spacing;
                s.offset[1] = (static_cast<float>(y) - centerY) * spacing;
                m_sourceInstances.push_back(s);
            }
        }
    }

    void CreateConstantBuffer()
    {
        const UINT cbSize = Align256(sizeof(FrameCB));

        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(cbSize),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&m_cb)));

        void* mapped = nullptr;
        CD3DX12_RANGE range(0, 0);
        ThrowIfFailed(m_cb->Map(0, &range, &mapped));
        m_cbMapped = mapped;
    }

    void CreateInstanceUploadBuffer()
    {
        const UINT bufferSize = InstanceCount * sizeof(RenderInstance);

        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(bufferSize),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&m_instanceUpload)));

        void* mapped = nullptr;
        CD3DX12_RANGE range(0, 0);
        ThrowIfFailed(m_instanceUpload->Map(0, &range, &mapped));
        m_instanceUploadMapped = mapped;
    }

    void UpdateCB()
    {
        float camX = 0.0f;
        float camY = 0.0f;
        GetCamera(camX, camY);

        const float frustumHalfW = 8.0f;
        const float frustumHalfH = 4.5f;

        XMMATRIX view = XMMatrixTranslation(-camX, -camY, 0.0f);
        XMMATRIX proj = XMMatrixOrthographicLH(frustumHalfW * 2.0f, frustumHalfH * 2.0f, 0.0f, 10.0f);
        XMMATRIX vp = view * proj;

        FrameCB cb = {};
        XMStoreFloat4x4(&cb.ViewProj, vp);
        cb.Camera = XMFLOAT4(camX, camY, frustumHalfW, frustumHalfH);
        cb.Params = XMFLOAT4(0, 0, 0, 0);

        memcpy(m_cbMapped, &cb, sizeof(cb));
    }

    void CpuFrustumCullAndBuildVisibleLists()
    {
        m_nearVisible.clear();
        m_midVisible.clear();
        m_farVisible.clear();

        float camX = 0.0f;
        float camY = 0.0f;
        GetCamera(camX, camY);

        const float frustumHalfW = 8.0f;
        const float frustumHalfH = 4.5f;
        const float extra = 0.25f;

        const float minX = camX - frustumHalfW - extra;
        const float maxX = camX + frustumHalfW + extra;
        const float minY = camY - frustumHalfH - extra;
        const float maxY = camY + frustumHalfH + extra;

        const float nearDist = 4.0f;
        const float midDist = 8.0f;

        for (const auto& src : m_sourceInstances)
        {
            const float x = src.offset[0];
            const float y = src.offset[1];

            if (x < minX || x > maxX || y < minY || y > maxY)
                continue;

            const float dx = x - camX;
            const float dy = y - camY;
            const float dist = std::sqrt(dx * dx + dy * dy);

            RenderInstance ri = {};
            ri.offset[0] = x;
            ri.offset[1] = y;

            if (dist < nearDist)
            {
                ri.scale = 1.00f;
                ri.color[0] = 0.85f;
                ri.color[1] = 1.00f;
                ri.color[2] = 0.85f;
                ri.color[3] = 1.00f;
                m_nearVisible.push_back(ri);
            }
            else if (dist < midDist)
            {
                ri.scale = 0.70f;
                ri.color[0] = 0.70f;
                ri.color[1] = 0.85f;
                ri.color[2] = 0.70f;
                ri.color[3] = 1.00f;
                m_midVisible.push_back(ri);
            }
            else
            {
                ri.scale = 0.45f;
                ri.color[0] = 0.55f;
                ri.color[1] = 0.65f;
                ri.color[2] = 0.55f;
                ri.color[3] = 1.00f;
                m_farVisible.push_back(ri);
            }
        }

        m_nearCount = static_cast<UINT>(m_nearVisible.size());
        m_midCount = static_cast<UINT>(m_midVisible.size());
        m_farCount = static_cast<UINT>(m_farVisible.size());

        m_nearStart = 0;
        m_midStart = m_nearCount;
        m_farStart = m_nearCount + m_midCount;

        RenderInstance* dst = reinterpret_cast<RenderInstance*>(m_instanceUploadMapped);
        size_t cursor = 0;

        if (!m_nearVisible.empty())
        {
            memcpy(dst + cursor, m_nearVisible.data(), m_nearVisible.size() * sizeof(RenderInstance));
            cursor += m_nearVisible.size();
        }

        if (!m_midVisible.empty())
        {
            memcpy(dst + cursor, m_midVisible.data(), m_midVisible.size() * sizeof(RenderInstance));
            cursor += m_midVisible.size();
        }

        if (!m_farVisible.empty())
        {
            memcpy(dst + cursor, m_farVisible.data(), m_farVisible.size() * sizeof(RenderInstance));
            cursor += m_farVisible.size();
        }

        m_totalVisible = static_cast<UINT>(cursor);
    }

    void MainLoop()
    {
        MSG msg = {};
        while (msg.message != WM_QUIT)
        {
            if (PeekMessage(&msg, nullptr, 0, 0, PM_REMOVE))
            {
                TranslateMessage(&msg);
                DispatchMessage(&msg);
            }
            else
            {
                Render();
            }
        }
    }

    void PopulateCommandList()
    {
        ThrowIfFailed(m_allocators[m_frameIndex]->Reset());
        ThrowIfFailed(m_cmdList->Reset(m_allocators[m_frameIndex].Get(), m_pso.Get()));

        UpdateCB();
        CpuFrustumCullAndBuildVisibleLists();

        auto toRT = CD3DX12_RESOURCE_BARRIER::Transition(
            m_backBuffers[m_frameIndex].Get(),
            D3D12_RESOURCE_STATE_PRESENT,
            D3D12_RESOURCE_STATE_RENDER_TARGET);
        m_cmdList->ResourceBarrier(1, &toRT);

        CD3DX12_CPU_DESCRIPTOR_HANDLE rtv(
            m_rtvHeap->GetCPUDescriptorHandleForHeapStart(),
            static_cast<INT>(m_frameIndex),
            m_rtvInc);

        FLOAT clear[] = { 0.08f, 0.08f, 0.12f, 1.0f };
        m_cmdList->OMSetRenderTargets(1, &rtv, FALSE, nullptr);
        m_cmdList->ClearRenderTargetView(rtv, clear, 0, nullptr);

        m_cmdList->SetGraphicsRootSignature(m_rootSig.Get());
        m_cmdList->SetGraphicsRootConstantBufferView(0, m_cb->GetGPUVirtualAddress());

        m_cmdList->RSSetViewports(1, &m_viewport);
        m_cmdList->RSSetScissorRects(1, &m_scissor);
        m_cmdList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLESTRIP);

        D3D12_VERTEX_BUFFER_VIEW views[2] = {};
        views[0] = m_vbView;
        views[1].BufferLocation = m_instanceUpload->GetGPUVirtualAddress();
        views[1].StrideInBytes = sizeof(RenderInstance);
        views[1].SizeInBytes = m_totalVisible * sizeof(RenderInstance);
        m_cmdList->IASetVertexBuffers(0, 2, views);

        if (m_nearCount > 0)
        {
            m_cmdList->DrawInstanced(4, m_nearCount, 0, m_nearStart);
        }

        if (m_midCount > 0)
        {
            m_cmdList->DrawInstanced(4, m_midCount, 0, m_midStart);
        }

        if (m_farCount > 0)
        {
            m_cmdList->DrawInstanced(4, m_farCount, 0, m_farStart);
        }

        auto toPresent = CD3DX12_RESOURCE_BARRIER::Transition(
            m_backBuffers[m_frameIndex].Get(),
            D3D12_RESOURCE_STATE_RENDER_TARGET,
            D3D12_RESOURCE_STATE_PRESENT);
        m_cmdList->ResourceBarrier(1, &toPresent);

        ThrowIfFailed(m_cmdList->Close());
    }

    void Render()
    {
        PopulateCommandList();

        ID3D12CommandList* lists[] = { m_cmdList.Get() };
        m_queue->ExecuteCommandLists(1, lists);

        ThrowIfFailed(m_swapChain->Present(1, 0));
        MoveToNextFrame();
    }

    void WaitForGpu()
    {
        ThrowIfFailed(m_queue->Signal(m_fence.Get(), m_fenceValue));
        ThrowIfFailed(m_fence->SetEventOnCompletion(m_fenceValue, m_fenceEvent));
        WaitForSingleObject(m_fenceEvent, INFINITE);
        ++m_fenceValue;
    }

    void MoveToNextFrame()
    {
        const UINT64 currentFence = m_fenceValue;
        ThrowIfFailed(m_queue->Signal(m_fence.Get(), currentFence));
        ++m_fenceValue;

        m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();

        if (m_fence->GetCompletedValue() < currentFence)
        {
            ThrowIfFailed(m_fence->SetEventOnCompletion(currentFence, m_fenceEvent));
            WaitForSingleObject(m_fenceEvent, INFINITE);
        }
    }

    static LRESULT CALLBACK WindowProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
    {
        App* app = nullptr;

        if (msg == WM_NCCREATE)
        {
            CREATESTRUCTW* cs = reinterpret_cast<CREATESTRUCTW*>(lParam);
            app = reinterpret_cast<App*>(cs->lpCreateParams);
            SetWindowLongPtrW(hwnd, GWLP_USERDATA, reinterpret_cast<LONG_PTR>(app));
        }
        else
        {
            app = reinterpret_cast<App*>(GetWindowLongPtrW(hwnd, GWLP_USERDATA));
        }

        if (msg == WM_DESTROY)
        {
            PostQuitMessage(0);
            return 0;
        }

        return DefWindowProcW(hwnd, msg, wParam, lParam);
    }

private:
    HWND m_hwnd = nullptr;

    ComPtr<IDXGIFactory4> m_factory;
    ComPtr<ID3D12Device> m_device;
    ComPtr<ID3D12CommandQueue> m_queue;
    ComPtr<IDXGISwapChain3> m_swapChain;

    ComPtr<ID3D12DescriptorHeap> m_rtvHeap;
    ComPtr<ID3D12Resource> m_backBuffers[FrameCount];
    ComPtr<ID3D12CommandAllocator> m_allocators[FrameCount];
    ComPtr<ID3D12GraphicsCommandList> m_cmdList;
    ComPtr<ID3D12Fence> m_fence;

    ComPtr<ID3D12RootSignature> m_rootSig;
    ComPtr<ID3D12PipelineState> m_pso;

    ComPtr<ID3D12Resource> m_vertexBuffer;
    ComPtr<ID3D12Resource> m_cb;
    ComPtr<ID3D12Resource> m_instanceUpload;

    D3D12_VERTEX_BUFFER_VIEW m_vbView = {};

    void* m_cbMapped = nullptr;
    void* m_instanceUploadMapped = nullptr;

    std::vector<SourceInstance> m_sourceInstances;
    std::vector<RenderInstance> m_nearVisible;
    std::vector<RenderInstance> m_midVisible;
    std::vector<RenderInstance> m_farVisible;

    UINT m_nearCount = 0;
    UINT m_midCount = 0;
    UINT m_farCount = 0;
    UINT m_nearStart = 0;
    UINT m_midStart = 0;
    UINT m_farStart = 0;
    UINT m_totalVisible = 0;

    UINT m_frameIndex = 0;
    UINT m_rtvInc = 0;
    UINT64 m_fenceValue = 0;
    HANDLE m_fenceEvent = nullptr;

    CD3DX12_VIEWPORT m_viewport;
    CD3DX12_RECT m_scissor;
};

int WINAPI wWinMain(HINSTANCE hInst, HINSTANCE, PWSTR, int nCmdShow)
{
    try
    {
        App app;
        return app.Run(hInst, nCmdShow);
    }
    catch (const std::exception& e)
    {
        MessageBoxA(nullptr, e.what(), "Error", MB_OK | MB_ICONERROR);
        return -1;
    }
}

// shader.hlsl
cbuffer FrameCB : register(b0)
{
    row_major float4x4 g_ViewProj;
    float4 g_Camera;
    float4 g_Params;
};

struct VSIn
{
    float3 pos       : POSITION;
    float4 color     : COLOR;
    float2 instance   : INSTANCEPOS;
    float  scale     : INSTANCESCALE;
    float4 instColor : INSTANCECOLOR;
};

struct VSOut
{
    float4 pos   : SV_POSITION;
    float4 color : COLOR;
};

VSOut VSMain(VSIn i)
{
    VSOut o;

    float3 local = i.pos * i.scale;
    float3 world = local + float3(i.instance.xy, 0.0f);

    o.pos = mul(float4(world, 1.0f), g_ViewProj);
    o.color = i.color * i.instColor;
    return o;
}

float4 PSMain(VSOut i) : SV_Target
{
    return i.color;
}
```