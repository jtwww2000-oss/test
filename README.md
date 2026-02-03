```mermaid
flowchart TB
    %% 最基础原生语法，无任何自定义样式，全解析器兼容
    RS(Random Seed)
    SHAKE[SHAKE (Seed Expansion)]
    RS --> SHAKE

    %% SHAKE输出分支
    SHAKE --> rho(rho)
    SHAKE --> K(K)
    SHAKE --> rho_prime(rho')

    %% Expand模块
    ExpandA{ExpandA (NTT form)}
    ExpandS[ExpandS (Rejsam_s/s1∈[-2,2], NTT form)]
    rho --> ExpandA
    K --> ExpandS
    rho_prime --> ExpandS

    %% NTT乘法加法计算
    ExpandA --> A(A)
    ExpandS --> s1s2(s1, s2)
    MultAdd[t = A*s1 + s2 (NTT mult/add)]
    A --> MultAdd
    s1s2 --> MultAdd

    %% Power2Round分解
    Power2Round[Power2Round (d)]
    MultAdd --> Power2Round
    Power2Round --> t1(t1 Highbits)
    Power2Round --> t0(t0 Lowbits)

    %% 公私钥生成
    PK(Public Key (pk) = (rho, t1))
    rho --> PK
    t1 --> PK
    tr[tr = SHAKE(pk)]
    PK --> tr
    SK(Private Key (sk) = (rho, K, tr, s1, s2, t0))
    rho --> SK
    K --> SK
    tr --> SK
    s1s2 --> SK
    t0 --> SK
