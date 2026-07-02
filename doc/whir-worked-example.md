# WHIR 证明:三种读法 + 两轮折叠(手算笔记)

> 一份自包含的 WHIR prove 走查笔记。用最小参数,在 $\mathbb{F}_{17}$ 上每个数都可手算核对。
> 记号约定:数学用 LaTeX;代码标识符用等宽;向量用 $\vec{\cdot}$;下标 0 起始。

---

## 0. 运行实例的参数(让数字可复现)

### 域

| 参数 | 值 | 作用 |
|--|--|--|
| 基域 | $\mathbb{F}_{17}$ | 所有算术模 $17$。选 $17$ 是因为 $17-1=16=2^4$ 高度 2-adic,支持到大小 $16$ 的 FFT / RS 域 |
| 扩域 $EF$ | $=F$ | 仅为手算简化。真实里挑战取自更大的扩域,好让 Schwartz–Zippel 错误率 $\sim 1/\lvert EF\rvert$ 足够小 |

### 结构参数

| 参数                   | 值   | 作用                                                                                                                      |
| -------------------- | --- | ----------------------------------------------------------------------------------------------------------------------- |
| $m$                  | $2$ | hypercube 变量数 $=$ 列高的 $\log_2$。列有 $2^m=4$ 个值                                                                            |
| `l_skip`             | $0$ | 单变量跳过的层数。$0$ $=$ 纯多线性(不跳);$>0$ 则底部 `l_skip` 层用一次 FFT 当单变量处理                                                             |
| `n_stack`            | $2$ | 多线性部分的变量数。约束 $m=$ `l_skip` $+$ `n_stack`                                                                                |
| $k$                  | $1$ | 每个 WHIR 轮的折叠因子:一轮折掉 $2^k=2$(即 1 个变量),含 $k$ 个 sumcheck 子轮                                                                |
| `log_blowup`         | $1$ | RS 冗余的 $\log_2$。blowup $=2$,码字长 $=2\times$ 数据。初始 RS 域大小 $=2^{m+\text{log\_blowup}}=8$                                   |
| `num_whir_rounds`    | $2$ | WHIR 轮数 $=(m-$ `log_final_poly_len` $)/k=2$                                                                             |
| `log_final_poly_len` | $0$ | 折到多项式剩 $2^0=1$ 个系数(常数)就停,明文发 `final_poly`                                                                               |
| 每轮查询数                | $1$ | 每轮抽查次数。真实用几十次保安全,这里 1 次便于手算                                                                                             |
| `*_pow_bits`         | $0$ | 三处 PoW grinding 难度(`mu_pow_bits` / `folding_pow_bits` / `query_phase_pow_bits`)。$0$ $=$ 不 grinding;真实 $>0$ 放大 soundness |

> **`num_whir_rounds` 为什么 $=(m-$`log_final_poly_len`$)/k$**:WHIR 每轮折掉 $k$ 个变量(= $k$ 个 sumcheck 子轮),从 $m$ 个变量一路折到只剩 `log_final_poly_len` 个(此时多项式仅 $2^{\texttt{log\_final\_poly\_len}}$ 个系数,小到可明文发 `final_poly`,不必再折)。**要折掉的变量总数** $=m-$`log_final_poly_len`,**每轮折** $k$ 个 $\Rightarrow$ 轮数 $=(m-$`log_final_poly_len`$)/k$(需整除)。本例 $(2-0)/1=2$:round 0 折 $x_0$($2\to1$)、round 1 折 $x_1$($1\to0$,剩 $2^0=1$ 系数 $=$ `final_poly`)。提前停(`log_final_poly_len`$>0$)是成本权衡:小多项式直接明文发比再折一轮便宜。

> **什么是 PoW / grinding**:同比特币挖矿——抽某个挑战**前**,prover 要找一个 nonce 使 `hash(transcript ‖ nonce)` 有 `pow_bits` 个前导零(平均试 $2^{\texttt{pow\_bits}}$ 次,找到的 nonce 就是 proof 里的 `*_pow_witness`),verifier 一次哈希即可验。作用是 **soundness 放大器**:Fiat–Shamir 下作弊者可反复"磨"挑战直到蒙混,grinding 让每次尝试贵 $2^{\texttt{pow\_bits}}$ 倍,于是 **总安全 $\approx$ 查询贡献 bit $+$ `pow_bits`**——就能用**更少查询**达到同样安全(见 blowup / 码率)。三处分别在抽 μ、抽折叠 α、查询前。本例全 $0$ $=$ 关掉,便于手算。

### 数据与命题

| 参数       | 值           | 作用                             |
| -------- | ----------- | ------------------------------ |
| `col`    | $[2,5,3,6]$ | 承诺的一列 trace(输入数据)              |
| $\vec u$ | $(4,2)$     | 求值点(上游抛的随机挑战);WHIR 证的就是列在此点的取值 |
| claim    | $16$        | 要证的开值,$=P(\vec u)$             |

### 固定挑战(真实由 Fiat–Shamir 哈希出;此处钉死便于手算)

| 挑战                                | 值       | 作用                                 |
| --------------------------------- | ------- | ---------------------------------- |
| $\alpha_0,\ \alpha_1$             | $3,\ 5$ | 折叠挑战(每个 sumcheck 子轮一个),把折叠"绑死"在随机点 |
| $z_0$                             | $5$     | round 0 的 OOD 域外点,钉死刚承诺的码字         |
| $\text{query}_0,\ \text{query}_1$ | $1,\ 1$ | 查询序号(抽查码字的哪个位置)                    |
| $\gamma_0,\ \gamma_1$             | $2,\ 2$ | 累加挑战,把 OOD 与查询用其幂折进下一轮的 claim / 权重 |

角的编号约定:序号 $=x_0+2x_1$(最低位是 $x_0$,LSB-first),所以 4 个角与 `col` 的对应是
$(0,0)\!\to\!2,\ (1,0)\!\to\!5,\ (0,1)\!\to\!3,\ (1,1)\!\to\!6$。

---

## 1. 一份数据,三种读法(A / B / C)

### 1.1 核心:一串数字没有"身份"

`col`$=[2,5,3,6]$ 只是 4 个域元素。"它是求值"还是"它是系数"是**解读方式**,不是数据自带的。同一串数,换个解读 = 另一个多项式。

最小例子(只看头两个数 $[2,5]$):
- 当**求值**:过 $(0,2),(1,5)$ 的线 $\Rightarrow 2+3x$。
- 当**系数**:$\Rightarrow 2+5x$。

不同的多项式($x{=}1$ 处一个 $5$、一个 $7$)。

### 1.2 三种读法各自是什么

| 读法    | 把 `col` 当  | 得到的多项式   | 角上的值             | 系数              |
| ----- | ---------- | -------- | ---------------- | --------------- |
| **A** | 多线性**求值**表 | $P$      | $[2,5,3,6]$(给定)  | $[2,3,1,0]$(反推) |
| **B** | **单变量系数**  | $g_0$    | —                | $[2,5,3,6]$(给定) |
| **C** | 多线性**系数**  | $\hat f$ | $[2,7,5,16]$(算出) | $[2,5,3,6]$(给定) |

**完整公式:**

**读法 A —— $P$**(`col` 是它在 4 个角的值):
- 选择子形(由值直接写出,每个角值 $\times$ 它的选择子):
  $$P(x_0,x_1)=2(1{-}x_0)(1{-}x_1)+5\,x_0(1{-}x_1)+3(1{-}x_0)x_1+6\,x_0x_1$$
- 单项式形(展开合并同类项,得系数 $[2,3,1,0]$):
  $$P(x_0,x_1)=2+3x_0+x_1\qquad(x_0x_1\text{ 系数为 }0)$$
  核对角:$P(0,0){=}2,\ P(1,0){=}2{+}3{=}5,\ P(0,1){=}2{+}1{=}3,\ P(1,1){=}2{+}3{+}1{=}6$ $\checkmark$

**读法 B —— $g_0$**(`col` 是单变量系数):
$$g_0(X)=2+5X+3X^2+6X^3$$

**读法 C —— $\hat f$**(`col` 是多线性系数):
$$\hat f(x_0,x_1)=2+5x_0+3x_1+6x_0x_1$$
它在 4 个角的值(代入):$(0,0){=}2,\ (1,0){=}2{+}5{=}7,\ (0,1){=}2{+}3{=}5,\ (1,1){=}2{+}5{+}3{+}6{=}16$,即 $[2,7,5,16]$。

三者对照:**A 和 C 都是 2 变量多线性多项式,但 A 把 `col` 当值、C 把 `col` 当系数,所以是两个不同的多项式**($P=2+3x_0+x_1$,而 $\hat f=2+5x_0+3x_1+6x_0x_1$);B 是把 `col` 当系数的**单变量**多项式(供 RS 编码用)。

### 1.3 各自的作用与目标

| 读法              | 角色     | 目标 / 用途                     | 用在协议哪一步       |
| --------------- | ------ | --------------------------- | ------------- |
| **B**($g_0$/码字) | **承诺** | 让 prover 赖不掉:RS 编码 + Merkle | 承诺阶段 + 查询时开叶子 |
| **A**($P$)      | **目标** | 要证的事:$P(\vec u)=16$         | 上游交来 + 最后核对   |
| **C**($\hat f$) | **桥**  | sumcheck 实际折叠的对象,把 A 和 B 接上 | WHIR 每轮折叠     |

为什么要三种:每个活在不同形态下才顺。
- 承诺要可抽查、能折叠 $\Rightarrow$ 只有 RS 行,而 RS 要**系数** $\Rightarrow$ 逼出 B。
- 上游命题天然是"MLE 在点的取值" $\Rightarrow$ 逼出 A 的值语义。
- sumcheck 要既折叠又跟码字一致 $\Rightarrow$ 做在 $\hat f$ 上 + mobius 权重 $\Rightarrow$ 逼出 C。

### 1.4 三者怎么关联(C 桥接 A、B)

**两个变换(同一多项式的两张脸):**
- 系数 $\to$ 值(zeta 变换):`col` 当系数 $\Rightarrow$ 值 $[2,7,5,16]$。
- 值 $\to$ 系数(Möbius 变换,zeta 的逆):`col` 当值 $\Rightarrow$ 系数 $[2,3,1,0]$。

**C $\leftrightarrow$ B(共享系数 $\Rightarrow$ 折叠一致):**
$\hat f$(多线性,系数 $=$`col`)与 $g_0$(单变量,系数 $=$`col`)是同一套系数。绑定 $x_0=\alpha$ 折叠 $\hat f$,与对码字做 FRI 偶奇折叠,给出**同一个**多项式:
$$\text{折 }\hat f:\ (2{+}5\alpha)+(3{+}6\alpha)x_1 \qquad=\qquad \text{FRI 偶奇}:\ \underbrace{[2,3]}_{\text{偶}}+\alpha\underbrace{[5,6]}_{\text{奇}}=[\,2{+}5\alpha,\ 3{+}6\alpha\,]$$
$\alpha=3$ 两边都得 $[0,4]$。$\Rightarrow$ sumcheck 折叠时始终和承诺的码字对得上。

**C $\leftrightarrow$ A(mobius 权重 $\Rightarrow$ 求和 = 开值):**
对 $\hat f$ 的值用 **mobius 权重**加权求和,正好等于 $P(\vec u)$:
$$\sum_{\vec x}\hat f(\vec x)\cdot\texttt{mobius\_eq}(\vec u,\vec x)=2\cdot4+7\cdot5+5\cdot3+16\cdot8=186\equiv 16=P(\vec u)$$
(若用普通 eq 权重,得到的是 $\hat f$ 自己的值 $8$,不是我们要的。)

合起来:
$$\underbrace{g_0\ \text{码字}}_{\text{B,系数=col}}\ \xleftrightarrow[\text{共享系数,折叠一致}]{}\ \underbrace{\hat f}_{\text{C,折叠对象}}\ \xleftrightarrow[\texttt{mobius\_eq},\ \text{求和}=16]{}\ \underbrace{P(\vec u)=16}_{\text{A,要证的开值}}$$

### 1.5 权重是怎么算出来的(不是挑的,是解出来的)

每个角的权重 = 逐坐标因子相乘。两套权重只差 $x_i=0$ 的因子:

| | $x_i=0$ 的因子 | $x_i=1$ 的因子 |
|--|--|--|
| eq | $1-u_i$ | $u_i$ |
| mobius | $1-2u_i$ | $u_i$ |

**第 1 步:代 $\vec u=(4,2)$ 算每个坐标的因子**(坐标 0 用 $u_0{=}4$,坐标 1 用 $u_1{=}2$):

| 坐标 | eq $x{=}0$($1{-}u$) | eq $x{=}1$($u$) | mobius $x{=}0$($1{-}2u$) | mobius $x{=}1$($u$) |
|--|--|--|--|--|
| 0 | $1{-}4\equiv14$ | $4$ | $1{-}8\equiv10$ | $4$ |
| 1 | $1{-}2\equiv16$ | $2$ | $1{-}4\equiv14$ | $2$ |

**第 2 步:每个角 = 坐标 0 因子 $\times$ 坐标 1 因子**(角按序号 $x_0{+}2x_1$ 排):

| 角 $(x_0,x_1)$ | eq 乘积 | eq 权重 | mobius 乘积 | mobius 权重 |
|--|--|--|--|--|
| $(0,0)$ | $14\cdot16=224$ | $3$ | $10\cdot14=140$ | $4$ |
| $(1,0)$ | $4\cdot16=64$ | $13$ | $4\cdot14=56$ | $5$ |
| $(0,1)$ | $14\cdot2=28$ | $11$ | $10\cdot2=20$ | $3$ |
| $(1,1)$ | $4\cdot2=8$ | $8$ | $4\cdot2=8$ | $8$ |

$\Rightarrow$ eq 权重 $=[3,13,11,8]$,mobius 权重 $=[4,5,3,8]$。

因子的来历($n=1$ 反解,要求"加权和 $=P(u)$ 对任意 `col` 成立"):

> $c_0,c_1$ 是**任意两个符号值**,代表 1 变量情形下的一列数据(不是具体数字)。用符号而非具体数,是为了让解出的权重对**任何**列都成立,从而能逐项配系数。同一列既当 $P$ 的**值**(eq 那行),又当 $\hat f$ 的**系数**(mobius 那行)——所以 $\hat f$ 的**值**是 $[c_0,\ c_0{+}c_1]$(系数$\to$值的 zeta:值在 $1$ 处 $=$ 系数之和)。两边目标都是 $P(u)=c_0(1{-}u)+c_1 u$。
>
> $m_0,m_1$ 是**待求解的权重(未知数)**:$m_0$ 乘在 $x{=}0$ 的值上、$m_1$ 乘在 $x{=}1$ 的值上。解出来的 $m_0,m_1$ 就是上面因子表的两列($x{=}0$ 因子、$x{=}1$ 因子)——$m$ 只是求解阶段的名字,解完填进表再逐角相乘。

- eq:被加权的是 $P$ 的值 $[c_0,c_1]$ $\Rightarrow$ $c_0 m_0+c_1 m_1=c_0(1{-}u)+c_1 u$ $\Rightarrow$ $m_0=1{-}u,\ m_1=u$。
- mobius:被加权的是 $\hat f$ 的值 $[c_0,\ c_0{+}c_1]$ $\Rightarrow$ $c_0 m_0+(c_0{+}c_1)m_1=c_0(1{-}u)+c_1 u$ $\Rightarrow$ $m_1=u,\ m_0=1{-}2u$。
- 代具体数感受($c_0{=}2,c_1{=}5,u{=}4$,目标 $P(4)=14$):eq 用值 $[2,5]$、权重 $[14,4]$ 得 $48\equiv14$;mobius 用值 $[2,7]$、权重 $[10,4]$ 也得 $48\equiv14$。$m_0$ 从 $14$ 降到 $10$,正好补偿 $\hat f$ 在位置 $1$ 多出的 $c_0$。

多变量:逐坐标因子连乘(多线性的张量结构)。

### 1.6 更深一层:权重 = "求值泛函"的坐标

前面讲了"怎么解";这里讲"为什么是这个形式"——本质是线性代数。

**① 权重是"在 $\vec u$ 求值"这个线性泛函的坐标。** "把多项式映射到它在 $\vec u$ 的取值"是一个**线性泛函**。给定多项式的一种表示(一串数),该泛函就写成一个**权重向量**,求值 $=\langle\text{数据},\ \text{权重}\rangle$(内积)。**同一个泛函,换一种表示,权重就变**——eq 与 mobius 就是"求值"这**同一个泛函**在两种表示(值 / 系数)下的坐标。

**② 为什么"配系数"能唯一钉死权重。** 要求 $\langle\text{数据},\ \text{权重}\rangle=P(\vec u)$ 对**任意**数据成立,等价于"左右是同一个线性泛函";而两个线性泛函相等 $\iff$ 每个坐标的系数都相等。所以逐项配 $c_0,c_1$ 的系数,权重被唯一确定(不是凑的)。

**③ 为什么叫 "mobius":两套权重差一个(转置的)Möbius 变换。** 记 zeta 变换(系数$\to$值)为矩阵 $Z$。`col` 当值就是 $P$,有 $P(\vec u)=\langle\text{col},\ \mathrm{eq}(\vec u)\rangle$;`col` 当系数时 $\hat f$ 的值 $=Z\,\text{col}$。要 $\langle Z\,\text{col},\ m\rangle=\langle\text{col},\ \mathrm{eq}(\vec u)\rangle$ 对任意 col 成立:

$$\langle Z\,\text{col},\ m\rangle=\langle\text{col},\ Z^\top m\rangle\ \Rightarrow\ Z^\top m=\mathrm{eq}(\vec u)\ \Rightarrow\ m=(Z^{-1})^\top\,\mathrm{eq}(\vec u)$$

而 $Z^{-1}$ 正是 **Möbius 变换**(值$\to$系数)。所以 **mobius 权重 $=$ eq 权重经"转置 Möbius 变换"推过来**——名字由此而来。1 变量验证($Z:[c_0,c_1]\mapsto$ 值 $[c_0,\ c_0{+}c_1]$):

$$Z=\begin{bmatrix}1&0\\1&1\end{bmatrix},\quad \mathrm{eq}(u)=\begin{bmatrix}1-u\\u\end{bmatrix},\quad (Z^{-1})^\top=\begin{bmatrix}1&-1\\0&1\end{bmatrix}$$
$$m=(Z^{-1})^\top\,\mathrm{eq}(u)=\begin{bmatrix}(1-u)-u\\ u\end{bmatrix}=\begin{bmatrix}1-2u\\ u\end{bmatrix}\quad\checkmark\ \text{正是 mobius 因子}$$

**④ 多变量 = 张量积**(即上面那句):eq、mobius、MLE 都是多线性,求值泛函随坐标分解,所以 1 变量解出的因子,多变量直接连乘。

### 1.7 折叠详解:两种折叠、一个运算

折叠出现在两处,底层是同一个运算。

**A. sumcheck 折叠(对 MLE 的"值")**:绑一个变量 $x_0=\alpha$。MLE 逐变量线性,$\hat f=(1-x_0)\,\hat f(0,\cdot)+x_0\,\hat f(1,\cdot)$;落到值表,把"只在 $x_0$ 上不同"的一对 $a,b$ 合成 $(1-\alpha)a+\alpha b=a+\alpha(b-a)$,表长减半。**例**:$[2,7,5,16]$ 绑 $x_0=3$ → $[\,2{+}3(7{-}2),\ 5{+}3(16{-}5)\,]=[0,4]$。**原理**:绑一个变量 $=$ 在那坐标求值一次,得少一维的 MLE。

**B. 码字折叠(FRI 偶奇,对 RS 码字)**:把 $g(X)=g_{\text{even}}(X^2)+X\,g_{\text{odd}}(X^2)$ 按次数拆开,折叠 $=g_{\text{even}}(Y)+\alpha\,g_{\text{odd}}(Y)$,次数减半。从陪集 $\{x,-x\}$(因 $x^2=(-x)^2$,平方到同一 $Y$)恢复:$g_{\text{even}}(x^2)=\tfrac{g(x)+g(-x)}{2}$、$g_{\text{odd}}(x^2)=\tfrac{g(x)-g(-x)}{2x}$;`binary_k_fold` 即算 $g_{\text{even}}(x^2)+\alpha\,g_{\text{odd}}(x^2)$(等价 $h(X)+(\alpha-X)\tfrac{h(X)-h(-X)}{2X}$)。**例**:$g_0=2+5X+3X^2+6X^3$,偶 $[2,3]$、奇 $[5,6]$,$\alpha=3$ → $(2{+}3Y)+3(5{+}6Y)\equiv4Y$;查询里陪集 $\{9,8\}$ 值 $[6,8]$ 得 $\tfrac{6+8}{2}+3\cdot\tfrac{6-8}{2\cdot9}\equiv1=g(13)$ ✓。**原理**:偶奇拆分把次数减半,随机 $\alpha$ 把"测低次"归约到折叠后、并绑定 $\alpha$。

**C. 为何两者给出同一多项式(命门)**:$\hat f$(多线性)与 $g$(单变量)共享同一套系数 `col`。sumcheck 绑 $x_0$ 把"$x_0$ 不在 / 在"的项按 $(1,\alpha)$ 组合;FRI 把"偶 / 奇"次项按 $(1,\alpha)$ 组合;而多线性系数 index 的 bit 0($x_0$)$=0/1$ 正是单变量系数的偶 / 奇 index——**同一批系数、同样权重**,故结果相同:两边都得 $[\,2{+}5\alpha,\ 3{+}6\alpha\,]$,$\alpha{=}3$ 都是 $[0,4]$。所以 verifier 从承诺码字折出的值(`binary_k_fold`)必等于 sumcheck 折出的多项式在该点的值——把承诺 B 与折叠 C 焊死(另见 §1.4、§3 ④)。

---

## 2. 输入与预处理(数据流)

```
col=[2,5,3,6]  -- eval_to_coeff_rs_message(0) -->  [2,5,3,6]
               -- coeffs_to_evals (zeta) -->       HatF = [2,7,5,16]
```

- $\hat f=$ HatF $=[2,7,5,16]$(被折叠的对象)。
- $\hat w=\texttt{mobius\_eq}(\vec u)=[4,5,3,8]$。
- 恒等式核对:$\sum\hat f\cdot\hat w=186\equiv 16=\hat f(\vec u)$ 对应的开值。$\checkmark$
- 承诺侧:`col` 当系数 $g_0(X)=2+5X+3X^2+6X^3$,在 $D_8=\langle 9\rangle=[1,9,13,15,16,8,4,2]$ 上 RS 编码得初始码字
  $$C=[16,6,3,7,11,8,12,4]$$
  建 Merkle 树得根 $R_0$(就是传入 `prove_whir_opening` 的 stacking 承诺)。

WHIR 核心 sumcheck 要证:$\displaystyle\sum_{\vec x\in H_2}\hat f(\vec x)\hat w(\vec x)=16$。

---

## 3. round 0(非最后轮)

> **怎么读这两节**:下面是 **prove 过程**——prover 一个人跑,挑战 $\alpha,z_0,\gamma$、query 序号都靠**自己哈希 transcript** 得到(Fiat–Shamir),**全程不做任何核对**。标 **[verify]** 的引用块是 verifier **事后独立**做的核对,**不属于生成证明**(prover 诚实计算,结果自然一致,无需自验)。连 claim 都是 verifier 追踪的量;prover 侧只维护 $\hat f,\hat w$。参见 §6。

进入时(prover 侧):$\hat f=[2,7,5,16]$,$\hat w=[4,5,3,8]$,初始 RS 域大小 $=8$。(verifier 侧目标 claim $=16$。)

### ① sumcheck 折叠(绑定 $x_0$)

**prover 算 $s$**:按 $x_1$ 把 $\hat f,\hat w$ 写成 $x_0$ 的线性式,

| $x_1$ | $\hat f(x_0,x_1)$ | $\hat w(x_0,x_1)$ |
|--|--|--|
| $0$ | $2+5x_0$ | $4+x_0$ |
| $1$ | $5+11x_0$ | $3+5x_0$ |

$$s(x_0)=(2+5x_0)(4+x_0)+(5+11x_0)(3+5x_0)$$
$$s(1)=7\cdot5+16\cdot8\equiv 10,\qquad s(2)=12\cdot6+10\cdot13\equiv 15$$

**产出** → `whir_sumcheck_polys[0]`$=[10,15]$。

> **$s$ 是什么、为什么只发两个值**:$s(x_0)$ 是这一子轮的一元多项式(留 $x_0$ 自由、对其余变量求和),$\hat f\cdot\hat w$ 两个一次相乘 $\Rightarrow$ $s$ 是 **2 次**,需 3 个点才能定死。但 verifier 靠 sumcheck 一致性 $s(0)=\text{claim}-s(1)$ **免费补第 3 个点**,所以 prover 只交 $s(1),s(2)$ 两个域元素;verifier 凑齐 $(0,s(0)),(1,s(1)),(2,s(2))$ 插值出 $s$,再在折叠挑战 $\alpha$ 处求值得新 claim。

**抽挑战(Fiat–Shamir)**:把 $[10,15]$ observe 进 transcript,哈希出折叠挑战 $\alpha_0=3$(抽前先 grinding,本例 $0$ bit 略过)。

**prover 折叠**(代 $x_0=3$,表长减半):
$$\hat f\to[\,2{+}15,\ 5{+}33\,]=[0,4],\qquad \hat w\to[\,4{+}3,\ 3{+}15\,]=[7,1]$$

> **[verify]**:补 $s(0)=\text{claim}-s(1)=16-10=6$(直接验 $2\cdot4+5\cdot3\equiv6$),核对 $s(0)+s(1)=16$ ✓;同样哈希出 $\alpha_0$;把 claim 更新为 $s(3)=4$(对 $[6,10,15]$ 二次插值)。

### ② 重新编码 + 承诺(prover)

折出的 $\hat f=[0,4]$ 当系数 $\Rightarrow g(Y)=4Y$。RS 域减半到 $4$,域 $D_4=\langle 13\rangle=[1,13,16,4]$,求值得码字:
$$g_{rs}=[\,4\cdot1,\ 4\cdot13,\ 4\cdot16,\ 4\cdot4\,]=[4,1,13,16]$$
哈希成 Merkle 根 $R_1$、observe 进 transcript。**产出** → `codeword_commits[0]`$=R_1$。

### ③ OOD(域外抽点)

**抽挑战**:哈希出域外点 $z_0=5$(不在 $D_4$)。
**prover 算**:$y_0=g(5)=4\cdot5=20\equiv 3$,observe 进 transcript。**产出** → `ood_values[0]`$=3$。作用:钉死刚承诺的码字。

### ④ 域内查询(prover 只负责开值)

**抽挑战**:哈希出 query 序号 $=1$。
**prover 算**:折叠域点 $z_i=9^2=13$,它在初始域 $D_8$ 的平方根陪集 $\{9,8\}$;从初始码字 $C=[16,6,3,7,11,8,12,4]$ 取 $C(9)=6,\ C(8)=8$,并取这两点到根 $R_0$ 的 Merkle 认证路径。**产出** → `initial_round_opened_rows`$=[[[6,8]]]$、`initial_round_merkle_proofs`。

> **[verify]**:用开值算 $\texttt{binary\_k\_fold}([6,8],[3],9)=6+(3-9)\tfrac{6-8}{2\cdot9}\equiv 1$,核对 $=g(13)=4\cdot13\equiv 1$ ✓(折叠与承诺一致);并验 Merkle 路径对得上 $R_0$。**注意 $\texttt{binary\_k\_fold}$ 是 verifier 算的——prover 只开值,不折。**

### ⑤ $\gamma$ 累加

**抽挑战**:哈希出 $\gamma_0=2$。
**prover 更新权重**(供下一轮 sumcheck):把 OOD 与查询的 eq 约束按 $\gamma$ 的幂累加进 $\hat w$($\text{eq}(\cdot,z)=[1{-}z,\ z]$):
$$\hat w=[7,1]\xrightarrow{+\gamma_0[13,5]}[16,11]\xrightarrow{+\gamma_0^2[5,13]}[2,12]$$

> **[verify]**:相应地把 claim 累加 $4+y_0\gamma_0+y_i\gamma_0^2=4+3\cdot2+1\cdot4=14$。(claim 由 verifier 追踪;prover 只更新权重 $\hat w$。)

**round 0 出口**(prover 侧):$\hat f=[0,4]$,$\hat w=[2,12]$,域减半($m=1$,大小 $4$)。(verifier 侧 claim $=14$。)

---

## 4. round 1(最后轮)

进入时(prover 侧):$\hat f=[0,4]$,$\hat w=[2,12]$,手里有 round 0 的码字 $g_{rs}=[4,1,13,16]$。最后一轮**跳过 ②③、不更新权重**。(verifier 侧 claim $=14$。)

### ① sumcheck 折叠(绑定 $x_1$)

**prover 算 $s$**:只剩 1 变量,$s(x_1)=\hat f(x_1)\hat w(x_1)$,其中 $\hat f(x_1)=4x_1,\ \hat w(x_1)=2+10x_1$:
$$s(1)=4\cdot12\equiv 14,\qquad s(2)=8\cdot5\equiv 6$$
**产出** → `whir_sumcheck_polys[1]`$=[14,6]$。
**抽挑战**:哈希出 $\alpha_1=5$。
**prover 折叠**:$\hat f\to[\,4\cdot5\,]=[20]\equiv[3]$(折成常数)。

> **[verify]**:补 $s(0)=14-14=0$(验 $0\cdot2=0$),核对 $s(0)+s(1)=14$ ✓;哈希出 $\alpha_1$;claim $\to s(5)=3$。

### ② 明文发 `final_poly`(最后轮:不编码、不 OOD)

折出的常数直接作 **产出** → `final_poly`$=[3]$,observe 进 transcript。

### ④ 域内查询(prover 从 round 0 码字开)

**抽挑战**:哈希出 query 序号 $=1$。
**prover 算**:$z_i=13^2\equiv 16$,平方根陪集 $\{13,4\}$;从 $g_{rs}=[4,1,13,16]$ 取 $g_{rs}(13)=1,\ g_{rs}(4)=16$,并取到根 $R_1$ 的 Merkle 路径。**产出** → `codeword_opened_values`$=[[[1,16]]]$、`codeword_merkle_proofs`。

> **[verify]**:$\texttt{binary\_k\_fold}([1,16],[5],13)=1+(5-13)\tfrac{1-16}{2\cdot13}\equiv 3$,核对 $=\texttt{final\_poly}=3$ ✓;验 Merkle 路径对 $R_1$。

### ⑤ $\gamma_1=2$(最后轮不更新权重)

**抽挑战**:哈希出 $\gamma_1=2$(仅为统一 verifier 逻辑;prover 不再更新 $\hat w$,也无更多产出)。

**到此 prove 结束** —— prover 已产出完整 `WhirProof`(见 §5)。下面全是 verifier 的事。

### 收尾:最终等式 $\text{acc}=\text{claim}$(纯 [verify])

verifier 用唯一明文的 `final_poly` 重算 $\text{acc}$,4 块相加(其中 claim 经 ⑤ 累加为 $s(5)+y_i\gamma_1^2=3+3\cdot4=15$):

| 贡献 | 算式 | 值 |
|--|--|--|
| 主项 | $\texttt{mobius\_eq}((4,2),(3,5))\cdot 3=11\cdot3$ | $16$ |
| round 0 OOD | $\gamma_0\cdot\text{eq}([5],[5])\cdot3=2\cdot7\cdot3$ | $8$ |
| round 0 查询 | $\gamma_0^2\cdot\text{eq}([5],[13])\cdot3=4\cdot11\cdot3$ | $13$ |
| round 1 查询 | $\gamma_1^2\cdot1\cdot3=4\cdot1\cdot3$ | $12$ |

$$\text{acc}=16+8+13+12=49\equiv \mathbf{15}=\text{claim}\ \checkmark\quad\Rightarrow\ \text{verifier 接受}$$

---

## 5. 输出:`WhirProof`

| 字段 | 值(本例) | 意义 |
|--|--|--|
| `mu_pow_witness` | $0$ | 抽 μ 批量挑战前的 PoW grinding 见证(本例单列无批量、`*_pow_bits`$=0$,故为 $0$) |
| `whir_sumcheck_polys` | $[[10,15],[14,6]]$ | 每个 sumcheck 子轮的多项式,以在 $\{1,2\}$ 的取值 $[s(1),s(2)]$ 给出、按 WHIR 轮扁平化;verifier 用运行 claim 补 $s(0)$ |
| `codeword_commits` | $[R_1]$ | 每轮折叠后码字的 Merkle 根(**最后轮除外**);本例仅 round 0 的 $[4,1,13,16]$ 之根 |
| `ood_values` | $[3]$ | 每轮(最后轮除外)的域外值 $y_0=g(z_0)$;本例 round 0 的 $g(5)=3$ |
| `folding_pow_witnesses` | $[0,0]$ | 每个 sumcheck 子轮、抽折叠挑战 $\alpha$ 前的 PoW 见证;长度 $=$ `num_whir_rounds`$\times k=2$ |
| `query_phase_pow_witnesses` | $[0,0]$ | 每个 WHIR 轮、查询阶段前的 PoW 见证;长度 $=$ `num_whir_rounds`$=2$ |
| `initial_round_opened_rows` | $[[[6,8]]]$ | **round 0** 从 stacking 承诺(初始码字 $C$)开出的陪集值;嵌套 $=$ 每承诺 $\times$ 每查询 $\times$ $2^k$ 陪集值($\times$ 每列) |
| `initial_round_merkle_proofs` | 到 $R_0$ 的路径 | 上面开值对初始承诺根 $R_0$ 的 Merkle 认证路径 |
| `codeword_opened_values` | $[[[1,16]]]$ | **非初始轮**从该轮码字(上一轮承诺的 `g_rs`)开出的陪集值;本例 round 1 从 round 0 码字开的 $[1,16]$ |
| `codeword_merkle_proofs` | 到 $R_1$ 的路径 | 上面开值对对应码字根(如 $R_1$)的 Merkle 路径 |
| `final_poly` | $[3]$ | 最后一轮折完的多项式系数,明文发送;长度 $=2^0=1$(`log_final_poly_len`$=0$,常数 $3$) |

> **为什么查询开值分两组**:round 0 的码字不是单独承诺的,而是从 stacking 承诺(原始列矩阵,可能多列)推出来,所以 `initial_round_*` 嵌套多一层"每列";round $\ge 1$ 直接开上一轮承诺的单列码字,用 `codeword_*`。
>
> **哪些进 transcript、哪些是 hint**:sumcheck 多项式、码字根、OOD 值、PoW 见证都 observe 进 transcript(参与 Fiat–Shamir);Merkle 路径是 hint(由 index $+$ 根确定,verifier 自己能重算,不 observe)。

### 5.1 每个值是怎么算出来的

按"哪一步产出它"分组(完整推导见 §3 / §4):

**来自 sumcheck(步 ①)**
- `whir_sumcheck_polys` $=[[10,15],[14,6]]$:每轮的 $[s(1),s(2)]$。round 0 用 $\hat f{=}[2,7,5,16],\ \hat w{=}[4,5,3,8]$ 算得 $[10,15]$;round 1 用 $\hat f{=}[0,4],\ \hat w{=}[2,12]$ 算得 $[14,6]$。
- `final_poly` $=[3]$:最后一轮把 $\hat f{=}[0,4]$ 于 $\alpha_1{=}5$ 折成常数 $[0+5\cdot4]=[20]\equiv3$。

**来自承诺 + OOD(步 ②③,仅非最后轮)**
- `codeword_commits` $=[R_1]$:round 0 折出 $g(Y){=}4Y$,编码成码字 $[4,1,13,16]$,哈希成 Merkle 根 $R_1$(round 1 最后轮不承诺)。
- `ood_values` $=[3]$:round 0 的 $z_0{=}5$,$y_0=g(5)=4\cdot5\equiv3$。

**来自查询(步 ④)**
- `initial_round_opened_rows` $=[[[6,8]]]$:round 0 query$=1$ → 折叠域点 $z_i{=}13$ → 初始域 $D_8$ 里的平方根陪集 $\{9,8\}$;从初始码字 $C=[16,6,3,7,11,8,12,4]$ 取 $C(9){=}6,\ C(8){=}8$。
- `codeword_opened_values` $=[[[1,16]]]$:round 1 query$=1$ → $z_i{=}16$ → 陪集 $\{13,4\}$;从 round 0 码字 $g_{rs}=[4,1,13,16]$ 取 $g_{rs}(13){=}1,\ g_{rs}(4){=}16$。
- `initial_round_merkle_proofs` / `codeword_merkle_proofs`:上面开值到对应根($R_0$ / $R_1$)的认证路径——沿 Merkle 树从叶子到根、每层的兄弟哈希,由 index $+$ 树唯一确定。

**PoW grinding(本例全 $0$)**
- `mu_pow_witness`、`folding_pow_witnesses`、`query_phase_pow_witnesses`:抽相应挑战前 grinding 找到的 nonce(要求 hash 有若干前导零)。本例 `*_pow_bits`$=0$,不需做功,全为 $0$;`folding` 每 sumcheck 轮一个 → $[0,0]$,`query_phase` 每 WHIR 轮一个 → $[0,0]$。

### 5.2 verifier 怎么用这些值验证

verifier 拿到 `WhirProof` 后**重演一遍**:用同样方式哈希出所有挑战,把每个字段"喂"回去,最后归到一个等式。关键认知——**真正逐步就拒的只有 PoW 和 Merkle;其余一致性(sumcheck、折叠)全部由最后一个 acc == claim 一并兜住。**

**A. 重建 claim(从 stacking 开值出发,逐轮更新)**
- 起点:过 `mu_pow_witness` 的 PoW、抽 μ 后,claim $=\sum_j\mu^j\cdot$(stacking 开值)(单列即 $16$)。
- 每个 sumcheck 子轮:读 `whir_sumcheck_polys` 的 $[s(1),s(2)]$,observe → 过 `folding_pow_witnesses` → 抽 α;令 $s(0):=\text{claim}-s(1)$(sumcheck 一致性由此**构造性成立**,不单独判),claim ← 二次插值 $s(\alpha)$。
- 每轮末:读 `ood_values` 的 $y_0$、由查询算出的 $y_i$,过 `query_phase_pow_witnesses`、抽 γ,累加 claim ← claim $+\,y_0\gamma+\sum y_i\gamma^{\cdots}$。
- 本例 claim 轨迹:$16\xrightarrow{\text{r0 sumcheck}}4\xrightarrow{\text{r0 累加}}14\xrightarrow{\text{r1 sumcheck}}3\xrightarrow{\text{r1 累加}}15$。

**B. 每个查询:Merkle 验证 + `binary_k_fold`(算出 $y_i$)**
- 初始轮:用 `initial_round_opened_rows`(+ μ)重构码字值,哈希成叶子,`merkle_verify` 对初始承诺根 $R_0$ ——**不过就拒**;再 `binary_k_fold` 得 $y_i$。
- 后续轮:用 `codeword_opened_values`,`merkle_verify` 对 `codeword_commits`(如 $R_1$),`binary_k_fold` 得 $y_i$。
- $y_i$ 不当场跟谁比,而是进 A 累加进 claim——正确性由 C 的最终等式兜。

**C. 最终等式:acc == claim(唯一的"大判")**
- 用**明文的 `final_poly`** 重算 acc:主项 $\texttt{mobius\_eq}(u_{[:t]},\alpha_{[:t]})\cdot\text{MLE}(\texttt{final\_poly})(u_{[t:]})$,再加每轮每个 $z_0/z_i$ 的 γ 加权项 $\text{eq}(\cdots)\cdot\text{horner}(\texttt{final\_poly},\cdots)$(即 §4 收尾那 4 块)。
- 本例 acc $=16+8+13+12=15$。`ensure(acc == claim)`:$15=15$ ✓ → 接受。

**硬性拒绝点**(其余靠 C 兜):(1) 三处 PoW witness 任一不过;(2) 任一查询的 Merkle 路径不符;(3) `final_poly` 长度不对,或 **acc ≠ claim**。

**每个字段在 verify 里的用途:**

| 字段 | verifier 拿它做什么 |
|--|--|
| `mu_pow_witness` | 过 PoW 后抽 μ |
| `whir_sumcheck_polys` | 逐轮 $s(0){:=}\text{claim}{-}s(1)$、插值更新 claim |
| `folding_pow_witnesses` / `query_phase_pow_witnesses` | 抽 α / 抽 query 前过 PoW |
| `codeword_commits` | observe;作后续轮查询的 Merkle 根 |
| `ood_values` | observe;累加进 claim;进最终 acc |
| `initial_round_opened_rows` + `..._merkle_proofs` | 重构初始码字值、验 Merkle(对 $R_0$)、`binary_k_fold` |
| `codeword_opened_values` + `..._merkle_proofs` | 后续轮同上(对 $R_1$…) |
| `final_poly` | observe;算 acc,做 acc == claim |

> **和 §3/§4 的呼应**:那里每步标 ✓ 的一致性,实现里并非逐步独立拒绝——sumcheck 的 $s(0)+s(1)=\text{claim}$ 由 $s(0){:=}\text{claim}-s(1)$ 构造性成立,查询的 `binary_k_fold` 值 $y_i$ 累加进 claim,**全部由最后 acc == claim 一并验证**;逐步就拒的只有 PoW 与 Merkle。

---

## 6. 谁做什么:prover vs verifier + Fiat–Shamir

证明系统有两个独立阶段:

| 阶段 | 谁 | 干什么 | 产物 |
|--|--|--|--|
| **Prove** | prover 一人 | 算 $s$、折叠、承诺、OOD 值、query 开值 | `WhirProof` |
| **Verify** | verifier 一人(之后) | 重做核对 | 接受 / 拒绝 |

- 上面把两边数字并排展示,是为了看到处处自洽;但"prover 发……"才进 proof,"verifier 核对……"是事后验证。
- **挑战** $\alpha,z_0,\gamma,$ query 概念上是 verifier 的随机数(soundness 必需:必须在 prover 承诺之后、且不可预测,否则 prover 能作弊)。
- **Fiat–Shamir**:把"verifier 抛随机"换成"挑战 $=$ 当前 transcript 的哈希"。于是 prover 自己哈希出每个挑战,**一个人**跑完产出 proof;真正的 verifier 之后用同样哈希重算挑战、独立核对。

### 6.1 transcript 与各挑战

**transcript** $=$ 协议里"已发生的所有消息"(prover 每条:sumcheck 多项式、码字根、OOD 值、`final_poly`……)按顺序攒成的一份哈希记录。两个操作:**observe**(吸收一条消息、更新哈希态)、**sample**(从当前哈希态挤出一个挑战)。Fiat–Shamir 令 **挑战 $=$ 当前 transcript 的哈希**;prover 与 verifier 按同样顺序吸收同样消息 $\Rightarrow$ 算出相同挑战。安全性:挑战依赖此前**所有**消息 $\Rightarrow$ prover 不能"先看挑战再改消息",改任何早先消息都会让后续挑战全变 $\Rightarrow$ 钉死 prover。

各挑战的作用与来处(都在吸收了它要"测"的消息**之后**才从 transcript 哈希出):

| 挑战           | 干什么                                                   | 何时从 transcript 哈希出          |
| ------------ | ----------------------------------------------------- | --------------------------- |
| $\mu$(批量)    | 用 $\mu$ 的幂把多列 RLC 成一条 $\hat f$                        | 开头(过 μ PoW 后);单列则平凡         |
| $\alpha$(折叠) | sumcheck 每子轮绑定一个变量的随机点;prover 已先交 $[s(1),s(2)]$,故折叠可信 | 每子轮 observe $[s(1),s(2)]$ 后 |
| $z_0$(OOD)   | 域外随机点,钉死刚承诺的码字(prover 报 $g(z_0)$),防"近似码字"作弊           | observe 码字根后(仅非最后轮)         |
| query 序号     | 抽查码字哪些位置(FRI 式随机查询 + Merkle 开),越多越安全                  | OOD/承诺后、开值前(`sample_bits`)  |
| $\gamma$(累加) | 用 $\gamma$ 的幂把 OOD 值 + 查询值 RLC 进下一轮 claim/权重          | 每轮末(查询之后)                   |

> **RLC(random linear combination,随机线性组合)**:用一个随机挑战 $r$ 的幂把若干东西合成一个 $\sum_i r^i a_i$。关键性质——"组合 $=$ 目标" $\Rightarrow$ "每个分量各自 $=$ 目标"(高概率,Schwartz–Zippel:某分量错则组合是 $r$ 的非零多项式,随机 $r$ 处几乎必非零),于是**把多个检查批成一个**。随机是必需的:固定系数下作弊者能让分量相消蒙混。WHIR 里 $\mu$ 用它把多列批成一条 $\hat f$、$\gamma$ 用它把 OOD/查询批进 claim(zerocheck 里也用它把多条约束批成一条)。

**顺序由 soundness 决定**(挑战必须在它要测的消息之后才抽):

$$\mu\ \to\ \Big[\ \underbrace{k\times(\text{交}[s(1),s(2)]\to\alpha)}_{\text{sumcheck}}\ \to\ \text{交码字根}\to z_0\to\text{交}y_0\ \to\ \text{query}\to\text{开值}\ \to\ \gamma\ \Big]\ \text{每轮}$$

---

## 7. 速查

- **A/B/C**:同一份 `col`,A 当值($P$,目标)、B 当单变量系数($g_0$,承诺)、C 当多线性系数($\hat f$,折叠桥)。
- **桥接**:C 与 B 共享系数 $\Rightarrow$ 折叠一致;C 用 mobius 权重 $\Rightarrow$ 求和 $=$ A 的开值。
- **每轮 5 步**:① sumcheck 折叠 ② 重编码+承诺 ③ OOD ④ 查询 ⑤ $\gamma$ 累加(最后轮:② 改发 `final_poly`、跳过 ③、不更新权重)。
- **三类核对**:$s(0)+s(1)=$claim;`binary_k_fold` $=$ 折叠多项式值;$\text{acc}=$claim。
- **命门**:同一个 $\hat f$ 既被折叠(对齐 B),其 mobius 加权和又等于开值(对齐 A)——"绑最低位变量"与"FRI 偶奇折叠"是同一个运算。

---

*配套源码:`openvm-org/stark-backend` 的 `develop-v2` 分支 —— `prover/whir.rs`、`verifier/whir.rs`、`prover/poly.rs`、`poly_common.rs`、`config.rs`、`proof.rs`。*
