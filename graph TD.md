```mermaid
%% 设置图表方向为从左到右，使数据流向更清晰
graph LR
    %% --- 样式定义 (采用鲜艳、专业的配色) ---
    %% 过程节点：深蓝色背景，白色文字
    classDef process fill:#1565C0,stroke:#0D47A1,stroke-width:2px,color:white,rx:5,ry:5;
    %% 存储节点：金黄色背景，深色文字，虚线边框
    classDef store fill:#FFC107,stroke:#FF6F00,stroke-width:2px,stroke-dasharray: 5 5,color:black,rx:5,ry:5;
    %% 输入输出节点：鲜绿色背景，白色文字
    classDef io fill:#2E7D32,stroke:#1B5E20,stroke-width:2px,color:white,rx:5,ry:5;
    %% 默认线条样式：加粗，深灰色
    linkStyle default stroke-width:2px,stroke:#37474F,fill:none;

    %% ========= 左侧：输入与初始生成 =========
    subgraph "阶段 1: 输入与扩展"
        Input_rho["输入: rho (256bit)"]:::io --> P_GenA
        Input_key["输入: key (256bit)"]:::io --> P_GenS

        P_GenA("P1: 生成矩阵 A<br>(SHAKE-128)"):::process
        P_GenS("P2: 生成向量 s1, s2<br>(SHAKE-256)"):::process
    end

    %% ========= 中间：存储与核心计算 =========
    subgraph "阶段 2: 存储与域变换"
        %% 数据存储 (圆柱体)
        DS_A[("DS1: 矩阵 A<br>(16x RAMs)")]:::store
        DS_s1[("DS2: 向量 s1<br>(4x RAMs)")]:::store
        DS_s2[("DS3: 向量 s2<br>(4x RAMs)")]:::store

        %% 连接生成器到存储
        P_GenA --> DS_A
        P_GenS --> DS_s1
        P_GenS --> DS_s2
        
        %% NTT 变换过程
        P_NTT_A("P3a: NTT(A)"):::process
        P_NTT_s1("P3b: NTT(s1)"):::process
        P_NTT_s2("P3c: NTT(s2)"):::process

        %% NTT 数据流 (读 -> 变换 -> 写回)
        DS_A -- Let A --> P_NTT_A -- NTT(A) --> DS_A
        DS_s1 -- Let s1 --> P_NTT_s1 -- NTT(s1) --> DS_s1
        DS_s2 -- Let s2 --> P_NTT_s2 -- NTT(s2) --> DS_s2
    end

    subgraph "阶段 3: 核心计算 (NTT域)"
        %% 点乘与累加
        P_PWM("P4: 点乘 (PWM)<br>A[i][j]*s1[j]"):::process
        P_Accum["P5: 累加器 (Sum)"]:::process
        P_Add_s2("P6: 加法 (+s2)"):::process

        %% 计算数据流
        DS_A -- NTT(A) --> P_PWM
        DS_s1 -- NTT(s1) --> P_PWM
        P_PWM --> P_Accum --> P_Add_s2
        DS_s2 -- "NTT(s2)" --> P_Add_s2
    end

    %% ========= 右侧：后处理与输出 =========
    subgraph "阶段 4: 后处理与输出"
        P_INTT("P7: 逆变换 (INTT)"):::process
        P_Round("P8: 舍入 (Power2Round)"):::process
        
        P_Add_s2 -- NTT(t) --> P_INTT -- Poly t --> P_Round
        
        %% 公钥输出
        Output_t1["输出: t1 (高位)"]:::io
        Output_t0["输出: t0 (低位)"]:::io
        P_Round --> Output_t1
        P_Round --> Output_t0

        %% 私钥打包与输出
        P_Pack_s1("P9a: 打包 s1"):::process
        P_Pack_s2("P9b: 打包 s2"):::process
        Output_s1["输出: s1"]:::io
        Output_s2["输出: s2"]:::io

        DS_s1 -. 最终状态 .-> P_Pack_s1 --> Output_s1
        DS_s2 -. 最终状态 .-> P_Pack_s2 --> Output_s2
    end
