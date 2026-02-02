```mermaid
graph TD
    %% ================= 定制样式定义 =================
    classDef func fill:#e3f2fd,stroke:#1565c0,stroke-width:2px; %% 蓝色矩形：函数
    classDef variable fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px; %% 绿色椭圆：变量
    classDef param fill:#fff3e0,stroke:#e65100,stroke-width:2px,stroke-dasharray: 5 5,shape:note; %% 橙色便签：参数
    classDef branch fill:#fce4ec,stroke:#c2185b,stroke-width:2px,shape:diamond; %% 粉色菱形：分支判断
    classDef loop fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,stroke-dasharray: 5 5,shape:circle; %% 紫色圆形：循环起点
    classDef process fill:#ffffff,stroke:#000000,stroke-width:1px,stroke-dasharray: 2 2; %% 白色虚线框：主要运算过程集合

    %% ================= 全局参数 =================
    subgraph Global_Params ["全局参数 (Parameter_Generator.m)"]
        Param("security_level (2/3/5)<br>mod_val = 8380417<br>维度: k, l, n=256<br>参数: omega, gamma1/2, tau, beta等"):::param
    end

    %% ================= 签名生成流程 (Sign.m) =================
    subgraph Sign_Process ["签名生成流程 (Sign.m)"]
        direction TB
        
        %% --- 输入 ---
        M_Sign[/"消息 M"/]:::variable
        SK_Sign[/"私钥 (s1_hat, s2_hat, t0_hat, rho', K)"/]:::variable
        
        %% --- 初始哈希 ---
        Hash_M_Func["SPONGE256(M, K, ...)"]:::func
        mu_Sign("μ (消息摘要, 512bit)"):::variable
        M_Sign -->|SHAKE256| Hash_M_Func
        SK_Sign -.-> Hash_M_Func
        Hash_M_Func --> mu_Sign

        %% --- 拒绝采样循环 ---
        subgraph Rejection_Loop ["拒绝采样循环 (While rej_flag == 1)"]
            Loop_Start((循环开始<br>nonce++)):::loop
            
            %% 1. 生成 y 并变换
            Rejsam_y_Func["Rejsam_y(rho', nonce, ...)"]:::func
            y_Sign("y (随机向量, l×n)"):::variable
            NTT_y_Func["ntt(y)"]:::func
            y_hat_Sign("y_hat (NTT域)"):::variable
            
            Loop_Start --> Rejsam_y_Func
            SK_Sign -.->|"rho'"| Rejsam_y_Func
            Rejsam_y_Func -->|"SHAKE256采样 & mod运算"| y_Sign
            y_Sign -->|"NTT变换"| NTT_y_Func --> y_hat_Sign

            %% 2. 计算 w
            MatrixMult_w_Func["A_hat * y_hat"]:::process
            w_hat_Sign("w_hat (NTT域, k×n)"):::variable
            INTT_w_Func["intt(w_hat)"]:::func
            w_Sign("w (时域向量)"):::variable
            
            y_hat_Sign -->|"矩阵乘法 (Pointwise Mont)"| MatrixMult_w_Func
            MatrixMult_w_Func --> w_hat_Sign
            w_hat_Sign -->|"INTT变换"| INTT_w_Func --> w_Sign

            %% 3. 生成挑战 c
            Decompose_w_Func["Decompose(w, 2*gamma2, ...)"]:::func
            w1_Sign("w1 (w的高位部分)"):::variable
            Hash_c_Func["SPONGE256(μ, w1_encode)"]:::func
            c_raw_Sign("c_raw (256bit哈希值)"):::variable
            SampleInBall_Func["SampleInBall(c_raw, tau)"]:::func
            c_poly_Sign("c (挑战多项式, τ个±1)"):::variable
            NTT_c_Func["ntt(c)"]:::func
            c_hat_Sign("c_hat (NTT域)"):::variable

            w_Sign -->|"位分解 & mod运算"| Decompose_w_Func --> w1_Sign
            mu_Sign -.-> Hash_c_Func
            w1_Sign -->|"编码 & SHAKE256"| Hash_c_Func --> c_raw_Sign
            c_raw_Sign -->|"SHAKE256采样"| SampleInBall_Func --> c_poly_Sign
            c_poly_Sign -->|"NTT变换"| NTT_c_Func --> c_hat_Sign

            %% 4. 计算潜在签名 z 和 r0
            Calc_z_proc["z = y + intt(c_hat * s1_hat)"]:::process
            z_Sign("z (候选签名向量, l×n)"):::variable
            Calc_r0_proc["r0 = Lowbits(w - intt(c_hat * s2_hat))"]:::func
            r0_Sign("r0 (低位余数向量, k×n)"):::variable

            y_Sign --> Calc_z_proc
            c_hat_Sign --> Calc_z_proc
            SK_Sign -.->|"s1_hat"| Calc_z_proc
            Calc_z_proc -->|"多项式乘加, INTT & mod"| z_Sign

            w_Sign --> Calc_r0_proc
            c_hat_Sign --> Calc_r0_proc
            SK_Sign -.->|"s2_hat"| Calc_r0_proc
            Calc_r0_proc -->|"多项式运算, Lowbits & mod"| r0_Sign

            %% 5. 安全性检查 (分支)
            Check_Norms{"检查 Norm(z), Norm(r0)<br>及 Makehint 边界"}:::branch
            z_Sign --> Check_Norms
            r0_Sign --> Check_Norms
            c_hat_Sign -.-> Check_Norms
            SK_Sign -.->|"t0_hat, s2_hat"| Check_Norms

            Check_Norms --"不通过 (rej_flag=1)"--> Loop_Start
        end

        %% --- 生成 Hint 并输出 ---
        Check_Norms --"通过 (rej_flag=0)"--> Makehint_Func
        Makehint_Func["Makehint_pre(w, c_hat, s2_hat, t0_hat, ...)"]:::func
        h_Sign("h (Hint向量, k×n, 汉明重还限制)"):::variable
        Signature_Out[/"签名 Sigma = (z, h, c_raw)"/]:::variable

        w_Sign -.-> Makehint_Func
        c_hat_Sign -.-> Makehint_Func
        Makehint_Func -->|"Hint计算 & mod运算"| h_Sign
        
        z_Sign --> Signature_Out
        h_Sign --> Signature_Out
        c_raw_Sign --> Signature_Out
    end

    %% ================= 签名验证流程 (Verify.m) =================
    subgraph Verify_Process ["签名验证流程 (Verify.m)"]
        direction TB
        
        %% --- 输入 ---
        M_Ver[/"消息 M"/]:::variable
        Sig_Ver[/"签名 (z, h, c_raw)"/]:::variable
        PK_Ver[/"公钥 (A_hat, t1_hat, rho)"/]:::variable
        
        %% --- 初始哈希 ---
        Hash_M_Ver_Func["SPONGE256(M, PK_hash, ...)"]:::func
        mu_Ver("μ' (消息摘要)"):::variable
        M_Ver -->|SHAKE256| Hash_M_Ver_Func
        PK_Ver -.->|"Hash(PK)"| Hash_M_Ver_Func
        Hash_M_Ver_Func --> mu_Ver

        %% --- 解析签名与变换 ---
        Sig_Ver --> z_Ver("z"):::variable
        Sig_Ver --> h_Ver("h"):::variable
        Sig_Ver --> c_raw_Ver("c_raw"):::variable
        
        NTT_z_Func["ntt(z)"]:::func
        z_hat_Ver("z_hat (NTT域)"):::variable
        z_Ver -->|"NTT变换"| NTT_z_Func --> z_hat_Ver

        %% --- 重构挑战 c ---
        SampleInBall_Ver_Func["SampleInBall(c_raw, tau)"]:::func
        c_poly_Ver("c' (多项式)"):::variable
        NTT_c_Ver_Func["ntt(c')"]:::func
        c_hat_Ver("c'_hat (NTT域)"):::variable
        c_raw_Ver -->|"SHAKE256采样"| SampleInBall_Ver_Func --> c_poly_Ver
        c_poly_Ver -->|"NTT变换"| NTT_c_Ver_Func --> c_hat_Ver

        %% --- 重构 w 的近似值 w' ---
        Reconstruct_w_Proc["A_hat * z_hat - c'_hat * t1_hat * 2^d"]:::process
        w_approx_hat_Ver("w'_hat (NTT域近似值)"):::variable
        INTT_w_Ver_Func["intt(w'_hat)"]:::func
        w_approx_Ver("w' (时域近似值)"):::variable
        
        PK_Ver -.->|"A_hat, t1_hat"| Reconstruct_w_Proc
        z_hat_Ver --> Reconstruct_w_Proc
        c_hat_Ver --> Reconstruct_w_Proc
        Reconstruct_w_Proc -->|"矩阵运算, 多项式乘sub & mod"| w_approx_hat_Ver
        w_approx_hat_Ver -->|"INTT变换"| INTT_w_Ver_Func --> w_approx_Ver

        %% --- 使用 Hint 恢复 w1' ---
        UseHint_Func["UseHint(w', h, 2*gamma2, ...)"]:::func
        w1_prime_Ver("w1' (恢复的高位部分)"):::variable
        w_approx_Ver --> UseHint_Func
        h_Ver --> UseHint_Func
        UseHint_Func -->|"Hint应用 & mod运算"| w1_prime_Ver

        %% --- 计算验证哈希 c' ---
        Hash_c_prime_Func["SPONGE256(μ', w1'_encode)"]:::func
        c_prime_raw_Ver("c'_raw (计算出的哈希)"):::variable
        mu_Ver -.-> Hash_c_prime_Func
        w1_prime_Ver -->|"编码 & SHAKE256"| Hash_c_prime_Func --> c_prime_raw_Ver

        %% --- 最终比对 (分支) ---
        Final_Check{"c_raw == c'_raw?<br>AND<br>Norm(z) check?"}:::branch
        c_raw_Ver --> Final_Check
        c_prime_raw_Ver --> Final_Check
        z_Ver -.->|"Norm检查"| Final_Check

        Final_Check --"True"--> Valid_Res[/"验证通过 (Accept)"/]:::variable
        Final_Check --"False"--> Invalid_Res[/"验证失败 (Reject)"/]:::variable
    end

    %% ================= 跨流程关联与参数影响 =================
    Param -.-> Sign_Process
    Param -.-> Verify_Process
    Signature_Out === Sig_Ver
