
## 📍 ЧАСТЬ 1. СИНХРОННОЕ ВЗАИМОДЕЙСТВИЕ (gRPC)

```mermaid
flowchart LR
    subgraph CLIENT [КЛИЕНТ]
        A[Consumer / Producer]
    end

    subgraph SERVER [gRPC СЕРВЕР :50051]
        B[ProcessTransaction]
        C[CheckPalindrome]
        D[ComputeSHA256]
    end

    A -- "➡ gRPC запрос" --> B
    A -- "➡ gRPC запрос" --> C
    A -- "➡ gRPC запрос" --> D
    
    B -- "⬅ gRPC ответ (APPROVED/DECLINED)" --> A
    C -- "⬅ gRPC ответ (True/False)" --> A
    D -- "⬅ gRPC ответ (SHA256 hex)" --> A

    style CLIENT fill:#4CAF50,stroke:#2E7D32,stroke-width:2px,color:#fff
    style SERVER fill:#2196F3,stroke:#0D47A1,stroke-width:2px,color:#fff
    style B fill:#64B5F6,stroke:#1976D2,stroke-width:1px,color:#000
    style C fill:#64B5F6,stroke:#1976D2,stroke-width:1px,color:#000
    style D fill:#64B5F6,stroke:#1976D2,stroke-width:1px,color:#000
