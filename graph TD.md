```mermaid
graph TD
    %% --- 输入 ---
    Input_rho[输入: rho (256bit)] --> P_GenA
    Input_key[输入: key (256bit)] --> P_GenS
    %% Input_rho_prime[输入: rho_prime] -- 未在代码中使用 --> P_Unused

    %% --- 过程 1: 生成初始多项式 (扩展) ---
    %% 注：代码中通过预加载RAM模拟了SHAKE扩展
    subgraph "扩展阶段 (SHAKE Expansion)"
        P_GenA("P1: 生成矩阵 A<br>(SHAKE-128 扩展)")
        P_GenS("P2: 生成向量 s1, s2<br>(SHAKE-256 扩展)")
    end

    %% --- 数据存储 (RAMs - 初始状态) ---
    P_GenA -- 多项式系数 --> DS_A[("DS1: 矩阵 A 存储<br>(16x RAMs)")]
    P_GenS -- s1 系数 --> DS_s1[("DS2: 向量 s1 存储<br>(4x RAMs)")]
    P_GenS -- s2 系数 --> DS_s2[("DS3: 向量 s2 存储<br>(4x RAMs)")]

    %% --- 过程 2: NTT 变换 ---
    subgraph "域变换 (NTT Domain)"
        P_NTT_A("P3a: NTT 变换 (A)")
        P_NTT_s1("P3b: NTT 变换 (s1)")
        P_NTT_s2("P3c: NTT 变换 (s2)")
    end

    DS_A -- A 多项式 --> P_NTT_A -- NTT(A) --> DS_A
    DS_s1 -- s1 多项式 --> P_NTT_s1 -- NTT(s1) --> DS_s1
    DS_s2 -- s2 多项式 --> P_NTT_s2 -- NTT(s2) --> DS_s2

    %% --- 过程 3: 矩阵向量运算 ---
    subgraph "核心计算 ( t = A*s1 + s2 )"
        DS_A -- NTT(A) --> P_PWM
        DS_s1 -- NTT(s1) --> P_PWM
        P_PWM("P4: 点乘 (PWM)<br>NTT(A[i][j]) * NTT(s1[j])") --> P_Accum["P5: 累加器<br>(Sum = Σ PWM)"]
        DS_s2 -- NTT(s2) --> P_Add_s2
        P_Accum -- Sum(NTT(A)*NTT(s1)) --> P_Add_s2
        P_Add_s2("P6: 向量加法<br>Sum + NTT(s2)") -- NTT(t) --> P_INTT
    end

    %% --- 过程 4: 逆变换与舍入 ---
    subgraph "后处理 (Post-processing)"
        P_INTT("P7: 逆 NTT 变换 (INTT)") -- 多项式 t --> P_Round
        P_Round("P8: 模约减与舍入<br>(Power2Round)")
    end

    %% --- 输出 ---
    P_Round -- "t1 (高位部分)" --> Output_t1[输出: t1_out]
    P_Round -- "t0 (低位部分)" --> Output_t0[输出: t0_out]

    subgraph "私钥打包 (Packing)"
       DS_s1 -- s1 (最终状态) --> P_Pack_s1("P9a: 打包 s1")
       DS_s2 -- s2 (最终状态) --> P_Pack_s2("P9b: 打包 s2")
    end
    
    P_Pack_s1 --> Output_s1[输出: s1_out]
    P_Pack_s2 --> Output_s2[输出: s2_out]

    %% --- 样式定义 ---
    classDef process fill:#E3F2FD,stroke:#1565C0,stroke-width:2px;
    classDef store fill:#FFF9C4,stroke:#FBC02D,stroke-width:2px,stroke-dasharray: 5 5;
    classDef io fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px;

    class P_GenA,P_GenS,P_NTT_A,P_NTT_s1,P_NTT_s2,P_PWM,P_Accum,P_Add_s2,P_INTT,P_Round,P_Pack_s1,P_Pack_s2 process;
    class DS_A,DS_s1,DS_s2 store;
    class Input_rho,Input_key,Output_t1,Output_t0,Output_s1,Output_s2 io;
    ```