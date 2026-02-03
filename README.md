```mermaid
graph TD
    %% --- 全局标题与参数设置 ---
    title[Isomorphic Data Flow Diagram of CRYSTALS-Dilithium Signature Scheme]
    subgraph GlobalParams [全局参数配置 (Global Parameters)]
        style GlobalParams fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
        NOTE_Q[**所有模运算 (mod q) 均基于 q = 8380417**]
        NOTE_SL[安全等级分支 (Security Levels):<br/>Level 2: k=4, l=4<br/>Level 3: k=6, l=5<br/>Level 5: k=8, l=7]
    end

    %% ==================================================
    %% 阶段一：密钥生成 (Key Generation)
    %% ==================================================
    subgraph KeyGen [Phase 1: 密钥生成 (KeyGen)]
        style KeyGen fill:#fbe9e7,stroke:#d84315,stroke-width:2px
        direction TB
        
        KG_In((随机种子 Seed)) --> KG_SHAKE[**SHAKE256**<br/>种子扩展]
        KG_SHAKE --ρ (256bit)--> KG_GenA[**Rejsam_a**<br/>生成矩阵 A ∈ R_q^(k×l)<br/>(均匀采样)]
        KG_SHAKE --ρ', K--> KG_GenS[**Rejsam_s**<br/>生成私钥向量 s1, s2<br/>(短系数采样 s_i ∈ [-η, η])]
        
        KG_GenS --> KG_NTT_S[**NTT 变换**<br/>多项式环模运算]
        KG_GenA & KG_NTT_S --> KG_MatMul[矩阵运算: t = A*s1 + s2]
        KG_MatMul --> KG_NTT_t[**NTT 变换**]
        KG_NTT_t --> KG_Decomp[**Highbits / Lowbits**<br/>位分解: t -> t1, t0]
        
        KG_Decomp --t1--> KG_PackPK[打包公钥 pk = (ρ, t1)]
        KG_Decomp --t0--> KG_PackSK[打包私钥 sk = (ρ, K, tr, s1, s2, t0)]
        KG_PackPK --> KG_HashPK[**CRH (SHAKE256)**<br/>计算 tr = H(pk)]
        KG_HashPK --> KG_PackSK
        
        KG_PackPK --> PK_Out((输出 pk))
        KG_PackSK --> SK_Out((输出 sk))
    end

    %% ==================================================
    %% 阶段二：签名生成 (Signature Generation)
    %% ==================================================
    subgraph Sign [Phase 2: 签名生成 (Sign)]
        style Sign fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
        direction TB
        
        S_InM((待签消息 M)) & S_InSK((输入 sk)) --> S_Prep[预处理: 解包 sk, 计算 μ = H(tr||M)]
        S_Prep --> S_LoopStart{拒绝采样循环开始}
        
        subgraph RejectionLoop [**Rejection Sampling Loop**]
            style RejectionLoop fill:#fff,stroke:#aaa,stroke-dasharray: 5 5
            S_LoopStart --κ (计数器)--> S_RejY[**Rejsam_y**<br/>采样掩码向量 y<br/>(非均匀, 范围较大)]
            S_RejY --> S_NTT_Y[**NTT 变换**] --> S_MatMul_Ay[矩阵运算: w = A*y]
            S_MatMul_Ay --> S_INTT_w[**INTT 变换**] --> S_Decomp_w[**Highbits**<br/>w -> w1]
            
            S_Prep --μ--> S_HashC
            S_Decomp_w --w1--> S_HashC[**SampleInBall (SHAKE)**<br/>生成挑战多项式 c<br/>(稀疏三元多项式)]
            S_HashC --c--> S_NTT_C[**NTT 变换**]
            
            S_NTT_C & S_InSK --> S_CalcZ[计算 z = y + c*s1]
            S_NTT_C & S_InSK & S_INTT_w --> S_CalcR0[计算 r0 = Lowbits(w - c*s2)]
            
            S_CalcZ & S_CalcR0 --> S_Check1{**Check 1**<br/>检查 z, r0 范数<br/>是否越界?}
            S_Check1 --Yes (Reject)--> S_LoopStart
            
            S_Check1 --No (Pass)--> S_MakeHint[**MakeHint**<br/>生成提示 h<br/>基于 -ct0 和 w-cs2+ct0]
            S_MakeHint --> S_Check2{**Check 2**<br/>检查 ct0 范数 &<br/>h 中 1 的个数?}
            S_Check2 --Yes (Reject)--> S_LoopStart
        end
        
        S_Check2 --No (Pass)--> S_PackSig[打包签名 σ = (z, h)]
        S_PackSig --> Sig_Out((输出签名 σ))
    end

    %% ==================================================
    %% 阶段三：签名验证 (Verification)
    %% ==================================================
    subgraph Verify [Phase 3: 签名验证 (Verify)]
        style Verify fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
        direction TB
        
        V_InM((消息 M)) & V_InPK((公钥 pk)) & V_InSig((签名 σ)) --> V_Unpack[解包: pk(ρ, t1), σ(z, h)]
        
        V_Unpack --> V_CheckZ{**Check z**<br/>检查 z 的无穷范数<br/>是否 < γ1 - β?}
        V_CheckZ --Yes (Fail)--> V_ResInvalid((验证失败 Invalid))
        
        V_CheckZ --No (Pass)--> V_Recon[重建阶段]
        V_Unpack --ρ--> V_GenA[**Rejsam_a**<br/>重建矩阵 A]
        V_Unpack --pk--> V_HashPK[**CRH**<br/>重建 tr = H(pk)]
        V_HashPK & V_InM --> V_HashMu[**CRH**<br/>重建 μ = H(tr||M)]
        
        V_Unpack --z--> V_NTT_Z[**NTT 变换**]
        V_GenA & V_NTT_Z & V_Unpack --t1--> V_ApproxW[计算近似 w'<br/>A*z - t1*2^d]
        V_ApproxW --> V_INTT_W[**INTT 变换**]
        
        V_INTT_W & V_Unpack --h--> V_UseHint[**UseHint**<br/>利用 hint h 重建 w1']
        V_HashMu & V_UseHint --> V_ReconC[**SampleInBall**<br/>重建挑战 c']
        
        V_ReconC & V_Unpack --h--> V_FinalCheck{**Final Check**<br/>验证 hint h 是否匹配<br/>生成的 c' 和 w'?}
        
        %% 注：标准流程通常比较 c'与c，但Dilithium实际实现通常比较 hint 是否一致
        V_FinalCheck --Yes (Match)--> V_ResValid((验证成功 Valid))
        V_FinalCheck --No (Mismatch)--> V_ResInvalid
    end

    %% --- 跨阶段依赖关系 (虚线) ---
    PK_Out -.-> V_InPK
    SK_Out -.-> S_InSK
    Sig_Out -.-> V_InSig

    %% --- 样式调整 ---
    classDef process fill:#fff,stroke:#333,stroke-width:1px;
    classDef decision fill:#fff,stroke:#333,stroke-width:1px,stroke-dasharray: 5 5;
    classDef boldOp font-weight:bold;
    
    class KG_SHAKE,KG_GenA,KG_GenS,KG_NTT_S,KG_NTT_t,KG_Decomp,KG_HashPK,S_RejY,S_NTT_Y,S_INTT_w,S_Decomp_w,S_HashC,S_NTT_C,S_MakeHint,V_GenA,V_HashPK,V_HashMu,V_NTT_Z,V_INTT_W,V_UseHint,V_ReconC boldOp;
