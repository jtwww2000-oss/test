```mermaid
flowchart TB
RS(Random Seed)
SHAKE[SHAKE (Seed Expansion)]
RS-->SHAKE
SHAKE-->rho(rho)
SHAKE-->K(K)
SHAKE-->rhot(rho t)
ExpandA[ExpandA (NTT form)]
ExpandS[ExpandS (Rejsam_s/s1∈[-2,2], NTT form)]
rho-->ExpandA
K-->ExpandS
rhot-->ExpandS
ExpandA-->A(A)
ExpandS-->s1s2(s1, s2)
MultAdd[t = A*s1 + s2 (NTT mult/add)]
A-->MultAdd
s1s2-->MultAdd
Power2Round[Power2Round (d)]
MultAdd-->Power2Round
Power2Round-->t1(t1 Highbits)
Power2Round-->t0(t0 Lowbits)
PK(Public Key (pk) = (rho, t1))
rho-->PK
t1-->PK
tr[tr = SHAKE(pk)]
PK-->tr
SK(Private Key (sk) = (rho, K, tr, s1, s2, t0))
rho-->SK
K-->SK
tr-->SK
s1s2-->SK
t0-->SK
s1s2-->SK
t0-->SK
