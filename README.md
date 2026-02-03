```mermaid
flowchart TB
RS("随机种子")
SHAKE("Shake(扩展种子)")
RS-->SHAKE
SHAKE-->rho("rho")
SHAKE-->K("K")
SHAKE-->rhot("rho'")
ExpandA("矩阵A (NTT 域)")
ExpandS("向量S (拒绝采样/s1∈[-2,2], NTT域)")
rho-->ExpandA
K-->ExpandS
rhot-->ExpandS
ExpandA-->A("A")
ExpandS-->s1s2("s1(NTT域)s2(时域)")
MultAdd("t = A*s1 + s2 ")
A-->MultAdd
s1s2-->MultAdd
Power2Round("Power2Round (t)")
MultAdd-->Power2Round
Power2Round-->t1("t1 Highbits")
Power2Round-->t0("t0 Lowbits")
PK("Public Key (pk) = (rho, t1)")
rho-->PK
t1-->PK
tr("tr = SHAKE(pk)")
PK-->tr
SK("Private Key (sk) = (rho, K, tr, s1, s2, t0)")
rho-->SK
K-->SK
tr-->SK
s1s2-->SK
t0-->SK
