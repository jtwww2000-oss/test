```mermaid
graph TD
    %% 样式定义
    classDef func fill:#e1f5fe,stroke:#0277bd,stroke-width:2px;
    classDef variable fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,rx:10,ry:10;
    classDef param fill:#fff3e0,stroke:#e65100,stroke-width:2px,stroke-dasharray: 5 5;
    classDef branch fill:#fce4ec,stroke:#c2185b,stroke-width:2px,shape:diamond;
    classDef loop fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,stroke-dasharray: 5 5;
    classDef process fill:#ffffff,stroke:#000000,stroke-width:1px;

    subgraph "全局参数 (Parameter_Generator.m)"
        Param[/"security_level (2/3/5)\nmod_val = 8380417\nomega, psi, etc."/];
    end
    Param:::param

    %% =================== 签名生成流程 (Sign.m) ===================
    subgraph "签名生成流程 (Sign.m)"
        direction TB
        
        %% 输入与初始化
        Message_Sign(/"Message (M)"/):::variable
        PrivKey_Sign(/"私钥 (s1_hat, s2_hat, t0_hat, rho_apostrophe)"/):::variable
        PubKey_Sign(/"公钥 (A_hat)"/):::variable
        
        Sign_Start[/"Sign() 开始"/]:::func
        
        Sign_Start --> Message_Sign
        Sign_Start --> PrivKey_Sign
        Sign_Start --> PubKey_Sign

        %% 消息哈希
        Hash_M_Sign["SPONGE256(M, ...)"]:::func
        Message_Sign -->|哈希| Hash_M_Sign
        mu_Sign(/"mu (消息摘要)"/):::variable
        Hash_M_Sign --> mu_Sign

        %% 拒绝采样循环
        subgraph "拒绝采样循环 (While rej_flag == 1)"
            direction TB
            Loop_Start((循环开始)):::loop

            nonce_Sign(/"nonce"/):::variable
            Loop_Start --> nonce_Sign

            %% 采样 y
            Rejsam_y_Func["Rejsam_y(rho_apostrophe, nonce, ...)"]:::func
            PrivKey_Sign -.-> Rejsam_y_Func
            nonce_Sign --> Rejsam_y_Func
            y_Sign(/"y (随机向量)"/):::variable
            Rejsam_y_Func -->|"采样 & mod运算"| y_Sign

            %% NTT变换 y
            NTT_y_Func["ntt(y)"]:::func
            y_Sign --> NTT_y_Func
            y_hat_Sign(/"y_hat (NTT域)"/):::variable
            NTT_y_Func --> y_hat_Sign

            %% 矩阵运算 A * y
            MatrixMult_w_Func["A_hat * y_hat"]:::process
            PubKey_Sign -.-> MatrixMult_w_Func
            y_hat_Sign --> MatrixMult_w_Func
            w_hat_Sign(/"w_hat (NTT域)"/):::variable
            MatrixMult_w_Func -->|"矩阵乘法 & mod运算"| w_hat_Sign

            %% INTT变换 w
            INTT_w_Func["intt(w_hat)"]:::func
            w_hat_Sign --> INTT_w_Func
            w_Sign(/"w (时域)"/):::variable
            INTT_w_Func --> w_Sign

            %% 分解 w
            Decompose_w_Func["Decompose(w, ...)"]:::func
            w_Sign --> Decompose_w_Func
            w1_Sign(/"w1 (高位部分)"/):::variable
            Decompose_w_Func -->|"位分解 & mod运算"| w1_Sign

            %% 哈希挑战 c
            Hash_c_Func["SPONGE256(mu, w1_encode)"]:::func
            mu_Sign -.-> Hash_c_Func
            w1_Sign -->|编码| Hash_c_Func
            c_raw_Sign(/"c_raw (哈希值)"/):::variable
            Hash_c_Func --> c_raw_Sign

            %% 采样挑战多项式
            SampleInBall_Func["SampleInBall(c_raw)"]:::func
            c_raw_Sign --> SampleInBall_Func
            c_poly_Sign(/"c_poly (挑战多项式)"/):::variable
            SampleInBall_Func --> SampleInBall_Func
            SampleInBall_Func --> c_poly_Sign

            %% NTT变换 c_poly
            NTT_c_Func["ntt(c_poly)"]:::func
            c_poly_Sign --> NTT_c_Func
            c_hat_Sign(/"c_hat (NTT域)"/):::variable
            NTT_c_Func --> c_hat_Sign

            %% 计算 z 和 r0
            Calc_z_Func["z = y + intt(c_hat * s1_hat)"]:::process
            y_Sign -.-> Calc_z_Func
            c_hat_Sign --> Calc_z_Func
            PrivKey_Sign -.->|"s1_hat"| Calc_z_Func
            z_Sign(/"z (签名向量)"/):::variable
            Calc_z_Func -->|"多项式乘法, INTT & mod运算"| z_Sign

            Calc_r0_Func["r0 = Lowbits(w - intt(c_hat * s2_hat))"]:::func
            w_Sign -.-> Calc_r0_Func
            c_hat_Sign --> Calc_r0_Func
            PrivKey_Sign -.->|"s2_hat"| Calc_r0_Func
            r0_Sign(/"r0 (低位部分)"/):::variable
            Calc_r0_Func -->|"多项式乘法, INTT, 位运算 & mod运算"| r0_Sign

            %% 安全性检查
            Check_Norms{{"Check Norms(z, r0) & Makehint Check"}}:::branch
            z_Sign --> Check_Norms
            r0_Sign --> Check_Norms
            c_hat_Sign -.-> Check_Norms
            PrivKey_Sign -.->|"t0_hat"| Check_Norms

            Check_Norms -- "不通过 (rej_flag=1)" --> Loop_Start
            Check_Norms -- "通过 (rej_flag=0)" --> Loop_End((循环结束)):::loop
        end

        %% 生成 Hint
        Loop_End --> Makehint_Func
        Makehint_Func["Makehint_pre(...)"]:::func
        w_Sign -.-> Makehint_Func
        c_hat_Sign -.-> Makehint_Func
        PrivKey_Sign -.->|"s2_hat, t0_hat"| Makehint_Func
        h_Sign(/"h (Hint向量)"/):::variable
        Makehint_Func -->|"Hint计算 & mod运算"| h_Sign

        %% 输出签名
        Signature_Out(/"Signature (z, h, c_raw)"/):::variable
        z_Sign --> Signature_Out
        h_Sign --> Signature_Out
        c_raw_Sign --> Signature_Out
        Sign_End[/"Sign() 结束"/]:::func
        Signature_Out --> Sign_End

        Param -.-> Rejsam_y_Func
        Param -.-> NTT_y_Func
        Param -.-> INTT_w_Func
        Param -.-> Decompose_w_Func
        Param -.-> NTT_c_Func
        Param -.-> Calc_r0_Func
        Param -.-> Check_Norms
        Param -.-> Makehint_Func
    end

    %% =================== 签名验证流程 (Verify.m) ===================
    subgraph "签名验证流程 (Verify.m)"
        direction TB

        %% 输入
        Message_Verify(/"Message (M)"/):::variable
        Signature_Verify(/"Signature (z, h, c_raw)"/):::variable
        PubKey_Verify(/"公钥 (A_hat, t1_hat, rho)"/):::variable

        Verify_Start[/"Verify() 开始"/]:::func
        Verify_Start --> Message_Verify
        Verify_Start --> Signature_Verify
        Verify_Start --> PubKey_Verify

        %% 消息哈希
        Hash_M_Verify["SPONGE256(M, ...)"]:::func
        Message_Verify -->|哈希| Hash_M_Verify
        mu_Verify(/"mu (消息摘要)"/):::variable
        Hash_M_Verify --> mu_Verify

        %% 解析签名
        z_Verify(/"z"/):::variable
        h_Verify(/"h"/):::variable
        c_raw_Verify(/"c_raw"/):::variable
        Signature_Verify --> z_Verify
        Signature_Verify --> h_Verify
        Signature_Verify --> c_raw_Verify

        %% NTT变换 z
        NTT_z_Verify["ntt(z)"]:::func
        z_Verify --> NTT_z_Verify
        z_hat_Verify(/"z_hat (NTT域)"/):::variable
        NTT_z_Verify --> z_hat_Verify

        %% 采样挑战多项式
        SampleInBall_Verify["SampleInBall(c_raw)"]:::func
        c_raw_Verify --> SampleInBall_Verify
        c_poly_Verify(/"c_poly"/):::variable
        SampleInBall_Verify --> c_poly_Verify

        %% NTT变换 c_poly
        NTT_c_Verify["ntt(c_poly)"]:::func
        c_poly_Verify --> NTT_c_Verify
        c_hat_Verify(/"c_hat (NTT域)"/):::variable
        NTT_c_Verify --> c_hat_Verify

        %% 重构 w_approx
        Reconstruct_w_Func["A_hat * z_hat - c_hat * t1_hat * 2^d"]:::process
        PubKey_Verify -.->|"A_hat, t1_hat"| Reconstruct_w_Func
        z_hat_Verify --> Reconstruct_w_Func
        c_hat_Verify --> Reconstruct_w_Func
        w_approx_hat_Verify(/"w_approx_hat (NTT域)"/):::variable
        Reconstruct_w_Func -->|"矩阵运算, 多项式乘法 & mod运算"| w_approx_hat_Verify

        %% INTT变换 w_approx
        INTT_w_Verify["intt(w_approx_hat)"]:::func
        w_approx_hat_Verify --> INTT_w_Verify
        w_approx_Verify(/"w_approx (时域)"/):::variable
        INTT_w_Verify --> w_approx_Verify

        %% 使用 Hint 恢复 w1
        UseHint_Func["UseHint(w_approx, h, ...)"]:::func
        w_approx_Verify --> UseHint_Func
        h_Verify --> UseHint_Func
        w1_prime_Verify(/"w1_prime (恢复的高位)"/):::variable
        UseHint_Func -->|"Hint应用 & mod运算"| w1_prime_Verify

        %% 哈希计算 c_prime
        Hash_c_prime_Func["SPONGE256(mu, w1_prime_encode)"]:::func
        mu_Verify -.-> Hash_c_prime_Func
        w1_prime_Verify -->|编码| Hash_c_prime_Func
        c_prime_Verify(/"c_prime (计算的哈希值)"/):::variable
        Hash_c_prime_Func --> c_prime_Verify

        %% 验证比对
        Verify_Check{{"c_raw == c_prime AND Check Norm(z)"}}:::branch
        c_raw_Verify --> Verify_Check
        c_prime_Verify --> Verify_Check
        z_Verify -.-> Verify_Check

        Verify_Check -- "通过 (True)" --> Verify_Success[/"验证成功"/]:::variable
        Verify_Check -- "失败 (False)" --> Verify_Fail[/"验证失败"/]:::variable

        Verify_End[/"Verify() 结束"/]:::func
        Verify_Success --> Verify_End
        Verify_Fail --> Verify_End

        Param -.-> NTT_z_Verify
        Param -.-> NTT_c_Verify
        Param -.-> INTT_w_Verify
        Param -.-> UseHint_Func
        Param -.-> Verify_Check
    end

    %% 跨流程关联
    PubKey_Sign -.- PubKey_Verify
    Message_Sign -.- Message_Verify
    Signature_Out -.- Signature_Verify
