```mermaid
flowchart TB
    %% 定义节点样式（兼容所有解析器）
    classDef ellipse fill:#f0f8ff,stroke:#333,stroke-width:1,rx:10,ry:10
    classDef rect fill:#e6f7ff,stroke:#333,stroke-width:1
    classDef roundedRect fill:#fff2e6,stroke:#333,stroke-width:1,rx:5,ry:5
    classDef hidden fill:none,stroke:none

    %% 1. 顶层输入与种子扩展
    RS[Random Seed]:::ellipse
    SHAKE[SHAKE\n(Seed Expansion)]:::roundedRect
    RS --> SHAKE

    %% 2. SHAKE 输出分支（隐藏节点传递中间数据）
    SHAKE --> rho[rho]:::hidden & K[K]:::hidden & rhoPrime[rho']:::hidden

    %% 3. ExpandA 和 ExpandS 模块
    ExpandA[ExpandA\n(NTT form)]:::rect
    ExpandS[ExpandS\n(Rejsam_s/s1∈[-2,2], NTT form)]:::roundedRect
    rho --> ExpandA
    K & rhoPrime --> ExpandS

    %% 4. 中间计算（NTT 乘法/加法）
    ExpandA --> A[A]:::hidden
    ExpandS --> s1s2[s1, s2]:::hidden
    MultAdd[t = A*s1 + s2\n(NTT mult/add)]:::roundedRect
    A & s1s2 --> MultAdd

    %% 5. Power2Round 分解
    Power2Round[Power2Round (d)]:::roundedRect
    MultAdd --> Power2Round
    Power2Round --> t1[t1\n(Highbits)]:::hidden & t0[t0\n(Lowbits)]:::hidden

    %% 6. 公钥与私钥生成（修复核心笔误）
    PK[Public Key (pk) = (rho, t1)]:::ellipse
    rho & t1 --> PK
    tr[tr = SHAKE(pk)]:::hidden
    PK --> tr
    SK[Private Key (sk)\n(rho, K, tr, s1, s2, t0)]:::ellipse
    rho & K & tr & s1s2 & t0 --> SK
