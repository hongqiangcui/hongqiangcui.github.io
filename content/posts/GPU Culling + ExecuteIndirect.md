---
title: "ExecuteIndirect & GPU culling (Win32 + DirectX12)"

---

<!--more-->

```
// main.cpp
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

using Microsoft::WRL::ComPtr;
using namespace DirectX;

static const UINT FrameCount = 2;
static const UINT Width = 1280;
static const UINT Height = 720;

static const UINT kGridWidth = 128;
static const UINT kInstanceCount = kGridWidth * kGridWidth;
static const float kSpacing = 1.5f;
static const float kCullHalfExtent = 32.0f;
static const float kQuadHalfSize = 0.35f;

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
    float position[3];
    float color[4];
};

struct alignas(256) PerFrameCBData
{
    XMFLOAT4X4 ViewProj;  // row_major in HLSL
    XMFLOAT4 Params0;     // x cameraX, y cameraY, z gridWidth, w instanceCount
    XMFLOAT4 Params1;     // x spacing, y cullHalfExtent, z quadHalfSize, w unused
};

struct IndirectCommandPacked
{
    UINT BaseInstance;
    UINT VertexCountPerInstance;
    UINT InstanceCount;
    UINT StartVertexLocation;
    UINT StartInstanceLocation;
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
    void InitWindow(HINSTANCE hInst, int nCmdShow)
    {
        WNDCLASSEXW wc = {};
        wc.cbSize = sizeof(wc);
        wc.style = CS_HREDRAW | CS_VREDRAW;
        wc.lpfnWndProc = WindowProc;
        wc.hInstance = hInst;
        wc.hCursor = LoadCursor(nullptr, IDC_ARROW);
        wc.lpszClassName = L"GPUCullingExecuteIndirectWindow";

        RegisterClassExW(&wc);

        RECT r = { 0, 0, (LONG)Width, (LONG)Height };
        AdjustWindowRect(&r, WS_OVERLAPPEDWINDOW, FALSE);

        m_hwnd = CreateWindowW(
            wc.lpszClassName,
            L"GPU Culling + ExecuteIndirect",
            WS_OVERLAPPEDWINDOW,
            CW_USEDEFAULT, CW_USEDEFAULT,
            r.right - r.left,
            r.bottom - r.top,
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

        // Queue
        {
            D3D12_COMMAND_QUEUE_DESC q = {};
            q.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
            ThrowIfFailed(m_device->CreateCommandQueue(&q, IID_PPV_ARGS(&m_queue)));
        }

        // Swap chain
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

        // RTV heap
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
        CreateBuffers();
        CreateCommandSignature();

        ThrowIfFailed(m_device->CreateCommandList(
            0,
            D3D12_COMMAND_LIST_TYPE_DIRECT,
            m_allocators[m_frameIndex].Get(),
            m_graphicsPSO.Get(),
            IID_PPV_ARGS(&m_cmdList)));
        ThrowIfFailed(m_cmdList->Close());

        ThrowIfFailed(m_device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&m_fence)));
        m_fenceValue = 1;
        m_fenceEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
        if (!m_fenceEvent)
            ThrowIfFailed(HRESULT_FROM_WIN32(GetLastError()));

        m_viewport = CD3DX12_VIEWPORT(0.0f, 0.0f, (float)Width, (float)Height);
        m_scissor = CD3DX12_RECT(0, 0, (LONG)Width, (LONG)Height);
    }

    void CreateRootSignature()
    {
        // 0: CBV b0
        // 1: UAV u0 (indirect command buffer)
        // 2: UAV u1 (visible count buffer)
        // 3: 1x 32-bit constant at b1 (base instance for ExecuteIndirect)
        CD3DX12_ROOT_PARAMETER params[4];
        params[0].InitAsConstantBufferView(0);                 // b0
        params[1].InitAsUnorderedAccessView(0);                // u0
        params[2].InitAsUnorderedAccessView(1);                // u1
        params[3].InitAsConstants(1, 1);                       // b1, 1 uint

        CD3DX12_ROOT_SIGNATURE_DESC desc;
        desc.Init(
            _countof(params), params,
            0, nullptr,
            D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

        ComPtr<ID3DBlob> sig, err;
        ThrowIfFailed(D3D12SerializeRootSignature(
            &desc, D3D_ROOT_SIGNATURE_VERSION_1, &sig, &err));

        ThrowIfFailed(m_device->CreateRootSignature(
            0, sig->GetBufferPointer(), sig->GetBufferSize(), IID_PPV_ARGS(&m_rootSig)));
    }

    void CreateShadersAndPSO()
    {
        ComPtr<ID3DBlob> vs, ps, cs, err;

        ThrowIfFailed(D3DCompileFromFile(
            L"shader.hlsl", nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE,
            "VSMain", "vs_5_0", 0, 0, &vs, &err));

        ThrowIfFailed(D3DCompileFromFile(
            L"shader.hlsl", nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE,
            "PSMain", "ps_5_0", 0, 0, &ps, &err));

        ThrowIfFailed(D3DCompileFromFile(
            L"shader.hlsl", nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE,
            "CSMain", "cs_5_0", 0, 0, &cs, &err));

        D3D12_INPUT_ELEMENT_DESC inputLayout[] =
        {
            { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0,  D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
            { "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
        };

        // Graphics PSO
        {
            D3D12_GRAPHICS_PIPELINE_STATE_DESC pso = {};
            pso.InputLayout = { inputLayout, _countof(inputLayout) };
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

            ThrowIfFailed(m_device->CreateGraphicsPipelineState(&pso, IID_PPV_ARGS(&m_graphicsPSO)));
        }

        // Compute PSO
        {
            D3D12_COMPUTE_PIPELINE_STATE_DESC pso = {};
            pso.pRootSignature = m_rootSig.Get();
            pso.CS = { cs->GetBufferPointer(), cs->GetBufferSize() };
            ThrowIfFailed(m_device->CreateComputePipelineState(&pso, IID_PPV_ARGS(&m_computePSO)));
        }
    }

    void CreateGeometry()
    {
        // 一个二维小方块（用 triangle strip 画）
        const Vertex vertices[] =
        {
            { { -0.5f,  0.5f, 1.0f }, { 1.f, 0.f, 0.f, 1.f } }, // TL
            { {  0.5f,  0.5f, 1.0f }, { 0.f, 1.f, 0.f, 1.f } }, // TR
            { { -0.5f, -0.5f, 1.0f }, { 0.f, 0.f, 1.f, 1.f } }, // BL
            { {  0.5f, -0.5f, 1.0f }, { 1.f, 1.f, 0.f, 1.f } }, // BR
        };

        const UINT vbSize = sizeof(vertices);

        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(vbSize),
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
        m_vbView.SizeInBytes = vbSize;
    }

    void CreateBuffers()
    {
        const UINT indirectBufferSize = kInstanceCount * sizeof(IndirectCommandPacked);

        // Indirect command buffer (UAV write by compute, then ExecuteIndirect reads it)
        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(
                indirectBufferSize,
                D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS),
            D3D12_RESOURCE_STATE_UNORDERED_ACCESS,
            nullptr,
            IID_PPV_ARGS(&m_indirectCommandBuffer)));

        // Visible count buffer (4 bytes, GPU writes it, ExecuteIndirect reads it as count buffer)
        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(
                sizeof(UINT),
                D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS),
            D3D12_RESOURCE_STATE_COPY_DEST,
            nullptr,
            IID_PPV_ARGS(&m_visibleCountBuffer)));

        // Zero upload buffer used to reset the visible count each frame
        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(Align256(sizeof(UINT))),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&m_zeroUploadBuffer)));

        void* mapped = nullptr;
        CD3DX12_RANGE range(0, 0);
        ThrowIfFailed(m_zeroUploadBuffer->Map(0, &range, &mapped));
        std::memset(mapped, 0, sizeof(UINT));
        m_zeroUploadBuffer->Unmap(0, nullptr);

        // Per-frame CB upload
        const UINT cbSize = Align256(sizeof(PerFrameCBData));
        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(cbSize),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&m_cbUpload)));

        ThrowIfFailed(m_cbUpload->Map(0, &range, &m_cbMapped));
        std::memset(m_cbMapped, 0, cbSize);
    }

    void CreateCommandSignature()
    {
        // Command layout:
        // [0] base instance (1 DWORD root constant for b1)
        // [1..4] D3D12_DRAW_ARGUMENTS
        D3D12_INDIRECT_ARGUMENT_DESC args[2] = {};
        args[0].Type = D3D12_INDIRECT_ARGUMENT_TYPE_CONSTANT;
        args[0].Constant.RootParameterIndex = 3;        // root param 3 = b1
        args[0].Constant.DestOffsetIn32BitValues = 0;
        args[0].Constant.Num32BitValuesToSet = 1;

        args[1].Type = D3D12_INDIRECT_ARGUMENT_TYPE_DRAW;

        D3D12_COMMAND_SIGNATURE_DESC desc = {};
        desc.ByteStride = sizeof(IndirectCommandPacked);
        desc.NumArgumentDescs = _countof(args);
        desc.pArgumentDescs = args;
        desc.NodeMask = 0;

        ThrowIfFailed(m_device->CreateCommandSignature(
            &desc, m_rootSig.Get(), IID_PPV_ARGS(&m_commandSignature)));
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

    void UpdatePerFrameCB()
    {
        float t = (float)GetTickCount64() * 0.001f;

        float camX = std::sinf(t * 0.45f) * 18.0f;
        float camY = std::cosf(t * 0.33f) * 12.0f;

        // 2D orthographic view-projection
        XMMATRIX view = XMMatrixTranslation(-camX, -camY, 0.0f);
        XMMATRIX proj = XMMatrixOrthographicLH(64.0f, 36.0f, 0.0f, 10.0f);
        XMMATRIX viewProj = view * proj;

        PerFrameCBData cb = {};
        XMStoreFloat4x4(&cb.ViewProj, viewProj);
        cb.Params0 = XMFLOAT4(camX, camY, (float)kGridWidth, (float)kInstanceCount);
        cb.Params1 = XMFLOAT4(kSpacing, kCullHalfExtent, kQuadHalfSize, 0.0f);

        std::memcpy(m_cbMapped, &cb, sizeof(cb));
    }

    void PopulateCommandList()
    {
        ThrowIfFailed(m_allocators[m_frameIndex]->Reset());
        ThrowIfFailed(m_cmdList->Reset(m_allocators[m_frameIndex].Get(), nullptr));

        UpdatePerFrameCB();

        // Reset visible count to 0 each frame
        {
            auto toCopyDest = CD3DX12_RESOURCE_BARRIER::Transition(
                m_visibleCountBuffer.Get(),
                D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT,
                D3D12_RESOURCE_STATE_COPY_DEST);
            m_cmdList->ResourceBarrier(1, &toCopyDest);

            m_cmdList->CopyBufferRegion(
                m_visibleCountBuffer.Get(), 0,
                m_zeroUploadBuffer.Get(), 0,
                sizeof(UINT));

            auto toUav = CD3DX12_RESOURCE_BARRIER::Transition(
                m_visibleCountBuffer.Get(),
                D3D12_RESOURCE_STATE_COPY_DEST,
                D3D12_RESOURCE_STATE_UNORDERED_ACCESS);
            m_cmdList->ResourceBarrier(1, &toUav);
        }

        // Ensure indirect command buffer is writable by compute
        {
            auto toUav = CD3DX12_RESOURCE_BARRIER::Transition(
                m_indirectCommandBuffer.Get(),
                D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT,
                D3D12_RESOURCE_STATE_UNORDERED_ACCESS);
            m_cmdList->ResourceBarrier(1, &toUav);
        }

        // Compute culling -> write visible indirect commands
        m_cmdList->SetPipelineState(m_computePSO.Get());
        m_cmdList->SetComputeRootSignature(m_rootSig.Get());
        m_cmdList->SetComputeRootConstantBufferView(0, m_cbUpload->GetGPUVirtualAddress());
        m_cmdList->SetComputeRootUnorderedAccessView(1, m_indirectCommandBuffer->GetGPUVirtualAddress());
        m_cmdList->SetComputeRootUnorderedAccessView(2, m_visibleCountBuffer->GetGPUVirtualAddress());

        UINT dispatchX = (kInstanceCount + 63u) / 64u;
        m_cmdList->Dispatch(dispatchX, 1, 1);

        // Make UAV writes visible before ExecuteIndirect reads them
        m_cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::UAV(m_indirectCommandBuffer.Get()));
        m_cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::UAV(m_visibleCountBuffer.Get()));

        // Transition to indirect-argument state
        {
            auto toIndirect = CD3DX12_RESOURCE_BARRIER::Transition(
                m_indirectCommandBuffer.Get(),
                D3D12_RESOURCE_STATE_UNORDERED_ACCESS,
                D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT);
            m_cmdList->ResourceBarrier(1, &toIndirect);

            auto toIndirectCount = CD3DX12_RESOURCE_BARRIER::Transition(
                m_visibleCountBuffer.Get(),
                D3D12_RESOURCE_STATE_UNORDERED_ACCESS,
                D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT);
            m_cmdList->ResourceBarrier(1, &toIndirectCount);
        }

        // Render
        auto toRT = CD3DX12_RESOURCE_BARRIER::Transition(
            m_backBuffers[m_frameIndex].Get(),
            D3D12_RESOURCE_STATE_PRESENT,
            D3D12_RESOURCE_STATE_RENDER_TARGET);
        m_cmdList->ResourceBarrier(1, &toRT);

        CD3DX12_CPU_DESCRIPTOR_HANDLE rtv(
            m_rtvHeap->GetCPUDescriptorHandleForHeapStart(),
            (INT)m_frameIndex,
            m_rtvInc);

        FLOAT clear[] = { 0.08f, 0.08f, 0.12f, 1.0f };
        m_cmdList->OMSetRenderTargets(1, &rtv, FALSE, nullptr);
        m_cmdList->ClearRenderTargetView(rtv, clear, 0, nullptr);

        m_cmdList->SetGraphicsRootSignature(m_rootSig.Get());
        m_cmdList->SetPipelineState(m_graphicsPSO.Get());
        m_cmdList->SetGraphicsRootConstantBufferView(0, m_cbUpload->GetGPUVirtualAddress());

        m_cmdList->RSSetViewports(1, &m_viewport);
        m_cmdList->RSSetScissorRects(1, &m_scissor);
        m_cmdList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLESTRIP);
        m_cmdList->IASetVertexBuffers(0, 1, &m_vbView);

        // 真正的 GPU-driven draw
        m_cmdList->ExecuteIndirect(
            m_commandSignature.Get(),
            kInstanceCount,
            m_indirectCommandBuffer.Get(),
            0,
            m_visibleCountBuffer.Get(),
            0);

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
            auto* cs = reinterpret_cast<CREATESTRUCTW*>(lParam);
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

    ComPtr<ID3D12RootSignature> m_rootSig;
    ComPtr<ID3D12PipelineState> m_graphicsPSO;
    ComPtr<ID3D12PipelineState> m_computePSO;
    ComPtr<ID3D12CommandSignature> m_commandSignature;

    ComPtr<ID3D12Resource> m_vertexBuffer;
    D3D12_VERTEX_BUFFER_VIEW m_vbView = {};

    ComPtr<ID3D12Resource> m_indirectCommandBuffer;
    ComPtr<ID3D12Resource> m_visibleCountBuffer;
    ComPtr<ID3D12Resource> m_zeroUploadBuffer;

    ComPtr<ID3D12Resource> m_cbUpload;
    void* m_cbMapped = nullptr;

    ComPtr<ID3D12Fence> m_fence;
    UINT64 m_fenceValue = 0;
    HANDLE m_fenceEvent = nullptr;

    UINT m_frameIndex = 0;
    UINT m_rtvInc = 0;

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
    float4 g_Params0;  // x cameraX, y cameraY, z gridWidth, w instanceCount
    float4 g_Params1;  // x spacing, y cullHalfExtent, z quadHalfSize, w unused
};

cbuffer DrawCB : register(b1)
{
    uint g_BaseInstance;
};

struct VSIn
{
    float3 pos   : POSITION;
    float4 color  : COLOR;
};

struct VSOut
{
    float4 pos   : SV_POSITION;
    float4 color : COLOR;
};

float2 GetInstanceWorldXY(uint instanceIndex)
{
    uint gridWidth = (uint)g_Params0.z;
    uint x = instanceIndex % gridWidth;
    uint y = instanceIndex / gridWidth;

    float2 center = float2((gridWidth - 1) * 0.5f, (gridWidth - 1) * 0.5f);
    return (float2(x, y) - center) * g_Params1.x;
}

VSOut VSMain(VSIn input)
{
    VSOut o;

    float2 local = input.pos.xy * g_Params1.z;
    float2 worldXY = GetInstanceWorldXY(g_BaseInstance) + local;

    o.pos = mul(float4(worldXY, 1.0f, 1.0f), g_ViewProj);
    o.color = input.color;
    return o;
}

float4 PSMain(VSOut input) : SV_Target
{
    return input.color;
}

// UAV 0: indirect command buffer
// UAV 1: visible count buffer (4 bytes)
RWByteAddressBuffer g_IndirectCommands : register(u0);
RWByteAddressBuffer g_VisibleCount     : register(u1);

[numthreads(64, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    uint idx = dtid.x;
    uint instanceCount = (uint)g_Params0.w;
    if (idx >= instanceCount)
        return;

    uint gridWidth = (uint)g_Params0.z;
    uint x = idx % gridWidth;
    uint y = idx / gridWidth;

    float2 center = float2((gridWidth - 1) * 0.5f, (gridWidth - 1) * 0.5f);
    float2 worldXY = (float2(x, y) - center) * g_Params1.x;

    float2 rel = worldXY - g_Params0.xy;
    float halfExtent = g_Params1.y;

    // 简化版 2D culling：相当于一个方形视锥
    if (abs(rel.x) > halfExtent || abs(rel.y) > halfExtent)
        return;

    uint dstIndex;
    g_VisibleCount.InterlockedAdd(0, 1, dstIndex);

    // 一个 ExecuteIndirect 命令的布局：
    // [0] baseInstance (root constant b1, 1 DWORD)
    // [1] VertexCountPerInstance
    // [2] InstanceCount
    // [3] StartVertexLocation
    // [4] StartInstanceLocation
    uint baseByte = dstIndex * 20;

    g_IndirectCommands.Store(baseByte + 0,  idx);
    g_IndirectCommands.Store(baseByte + 4,  4);
    g_IndirectCommands.Store(baseByte + 8,  1);
    g_IndirectCommands.Store(baseByte + 12, 0);
    g_IndirectCommands.Store(baseByte + 16, 0);
}
```