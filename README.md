## About

A hands-on tutorial on elliptic curves

### Lesson 0 - defining elliptic curves

Continuous case uses geogebra


```python
import plotly
```

```python
import numpy as np
```

```python
def elliptic(p, a, b):
    x = y = np.arange(p)
    xx, yy = np.meshgrid(x,y)
    return (np.mod(yy**2, p) - np.mod(xx**3+a*xx+b, p) == 0)*1.0
```

```python

```

## Finite elliptic curves

```python
import plotly.graph_objects as go

fig = go.Figure(data=go.Heatmap(z=elliptic(37, a=0, b=7), colorscale='gray'), layout=dict(width=700, height=700))
fig
```

Note symmetry about $p/2$


## Algebraic sum

Given
$$P=(x_p, y_p)$$
$$Q=(x_q, y_q)$$
What is $P + Q = -R$?

We want the (modulus) line passing through $P, Q$. The slope $m$ of that line is given by

$$m = (y_p-y_q)(x_p-x_q)^{-1} mod(p)  \quad P!=Q$$
$$ (3x_P^2+a) (2y_P)^{-1} mod(p) \quad P = Q, $$
$$a = 7 \quad \text{for bitcoin}$$

With $m$ we can obtain the $R$:
$$ x_R = (m^2 - x_P - x_Q) mod(p) $$
$$ y_r = (y_p + m(x_R-x_P)) mod(p) $$

```python
# taken from https://stackoverflow.com/questions/4798654/modular-multiplicative-inverse-function-in-python
# and http://anh.cs.luc.edu/331/notes/xgcd.pdf
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def xgcd(a, b):
    prevx, x = 1, 0
    prevy, y = 0, 1
    while b:
        q = a//b
        x, prevx = prevx - q*x, x
        y, prevy = prevy - q*y, y
        a, b = b, a % b
    return a, prevx, prevy
    
def modinv_2(a, b):
#         a = abs(a)%b # seems to be a bug because the next line allows negative assertion to pass
        if a < 0:
            a += b
            
        for m in range(b):
            if (a*m)%b == 1:
                return m
        return None
    
assert (modinv_2(13, 17)*13)%17 == 1
assert (modinv_2(-13, 17)*(-13))%17 == 1


# def modinv(a, m):
#     """multiplicative mod inverse of a mod m"""
# #         if( n < 0 ) {
# #             n = n + this.k;
# #         }
#     if a < 0:
#         a = a + m
# #     g, x, y = egcd(a, m)
#     g, x, y = xgcd(a, m)
#     if g != 1:
#         raise Exception(f'modular inverse does not exist for {a} mod {m}')
#     else:
#         return x % m


```

```python
def modinv(a, b):
    """from Jimmy Song"""
    return pow(a, b - 2, b) 

assert (modinv(13, 17)*13)%17 == 1
assert (modinv(-13, 17)*-13)%17 == 1
```

```python
def add_mod(P, Q, p, a, b=None):
    """add points P and Q through their intserection with curve (p,a,b) b is unused
    
    based on https://github.com/andreacorbellini/ecc/blob/865b8f9e79ed94a9d004a9b553f4b3add55878be/interactive/ec.js#L1059
    """
    if P is None:
        return Q
    if Q is None:
        return P
    
    x_p, y_p = P
    x_q, y_q = Q
    
    if x_p != x_q: # distinct points
        print('distinct')
        m = (((y_p-y_q)%p)*modinv((x_p - x_q)%p, p))%p;
        x_r = ((m*m)%p - (x_p - x_q)%p)%p
        y_r = ((m*((x_p-x_r)%p)%p)%p - y_p)%p
        return x_r, y_r
    else:
        if (y_p == 0)&(y_q == 0 ):
        # This may only happen if P=Q is a root of the elliptic
        # curve, hence the line is vertical.
            print('found root of elliptic')
            return None
        elif y_p == y_q:
        # The points are the same, but the line is not vertical.
#             print('points are the same')
            m = ((((3*(x_p*x_p)%p)%p+a)%p)*modinv((2*y_p)%p, p))%p
            x_r = ((m**2)%p - (2*x_p)%p)%p
            y_r = ((m*((x_p - x_r)%p))%p - y_p)%p
            return x_r, y_r
    
        else:
        # The points are not the same and the line is vertical.
            print('found vertical')
            return None


add_mod((22, 6), (22, 6), 37, 7)
```

```python
P_ = (3,6)
for i in range(10):
    print(i, P_)
    P_ = add_mod(P_, P_, p=97, a=2)
```

0 (3, 6)
1 (80, 10)
2 (3, 91)
3 (80, 87)
4 (3, 6)
5 (80, 10)
6 (3, 91)
7 (80, 87)
8 (3, 6)
9 (80, 10)

```python
def multiply_mod(P, n, p, a, b=None):
    """multiply P n times for curve (p,a,b)"""
    P_ = P + ()
    if n == 0:
        return 0
    elif n == 1:
        return P_
    for _ in range(1, n):
        P_ = add_mod(P_, P_, p, a) + ()
    return P_

multiply_mod((3, 6), 7, p=97, a=2)
```

```python
e2 = dict(p=97, a=2, b=3)

for _ in range(10):
    print(f"{_}: {multiply_mod((3,6), _, **e2)}")
```

How many points are in the subgroup generated by P?

1. count all points on the curve N (see [Schoof's Algorithm](https://en.wikipedia.org/wiki/Schoof%27s_algorithm))
1. find all divisors of N
1. for every divisor of N, see if $nP=0$

```python
order_dict = {} # cache results

def order(p, a, b):
    """calculate the order of the field including the point at infinity"""
    if (p,a,b) not in order_dict:
        order_ = int(elliptic(p, a, b).sum()+1) # brute force
        order_dict[(p,a,b)] = order_
    else:
        order_ = order_dict[(p,a,b)]
    return order_

order(37, a=-1, b=3)
```

```python
def divisors(n):
    for i in range(1, int(n / 2) + 1):
        if n % i == 0:
            yield i
    yield n
```

```python
def subgroup_order(P, p, a, b):
    N = order(p, a, b)
    for _ in divisors(N):
        P_ = P
        for i in range(_):
            P_ = add_mod(P_, P_, p=p,a=a)
        if P_ == P:
            break

    if _ == N:
        return N # already contains the point at infinity
    else:
        return _+1

subgroup_order((2,3), p=37, a=-1, b=3)
```

```python
order(p=29, a=-1, b=1) # is prime!
```

```python
subgroup_order((9,24), p=29, a=-1, b=1) # should be 37 instead of 38
```

get the subgroup order to do the multiplication modulus

```python
def multiply_mod(P, n, p, a, b=None):
    """multiply P n times for curve (p,a,b)"""
    if P is None:
        return None
    sgo = subgroup_order(P, p=p, a=a, b=b)
    
    # n(P) = (n mod sgo) P
    n_ = n%sgo

    P_ = P + ()
    if n_ == 0:
        return None
    elif n == 1:
        return P_
    
    for _ in range(1, n_):
        P_ = add_mod(P_, P_, p, a) + ()
    return P_
```

```python
e2 = dict(p=97, a=2, b=3)

for _ in range(10):
    print(f"{_}: {multiply_mod((3,6), _, **e2)}")
```

```python
fig = go.Figure(data=go.Heatmap(z=elliptic(p=29, a=-1, b=1), colorscale='gray'),
                layout=dict(width=700, height=700))
fig
```

## My implementation doesn't match corbellini's

Checking with Jimmy Song's version:

```python
import sys

sys.path.append('../programmingbitcoin/code-ch03/')

from ecc import Point, FieldElement
```

```python
x_, y_ = FieldElement(3, 97), FieldElement(6, 97)
a_, b_ = FieldElement(2, 97), FieldElement(3, 97)

e2 = dict(p=97, a=2, b=3)
```

```python
p_ = Point(x_, y_, a_, b_)
for _ in range(10):
    print(_, p_)
    p_ = p_+p_
```

```python
(0*Point(x_, y_, a_, b_))
```

0 (3, 6)
1 (80, 10)
2 (3, 91)
3 (80, 87)
4 (3, 6)
5 (80, 10)
6 (3, 91)
7 (80, 87)
8 (3, 6)
9 (80, 10)


I'm getting the same result as Jimmy's code!

<!-- #region -->
```python
    def __add__(self, other):
        if self.a != other.a or self.b != other.b:
            raise TypeError('Points {}, {} are not on the same curve'.format(self, other))
        # Case 0.0: self is the point at infinity, return other
        if self.x is None:
            return other
        # Case 0.1: other is the point at infinity, return self
        if other.x is None:
            return self

        # Case 1: self.x == other.x, self.y != other.y
        # Result is point at infinity
        if self.x == other.x and self.y != other.y:
            return self.__class__(None, None, self.a, self.b)

        # Case 2: self.x ≠ other.x
        # Formula (x3,y3)==(x1,y1)+(x2,y2)
        # s=(y2-y1)/(x2-x1)
        # x3=s**2-x1-x2
        # y3=s*(x1-x3)-y1
        if self.x != other.x:
            s = (other.y - self.y) / (other.x - self.x)
            x = s**2 - self.x - other.x
            y = s * (self.x - x) - self.y
            return self.__class__(x, y, self.a, self.b)

        # Case 4: if we are tangent to the vertical line,
        # we return the point at infinity
        # note instead of figuring out what 0 is for each type
        # we just use 0 * self.x
        if self == other and self.y == 0 * self.x:
            return self.__class__(None, None, self.a, self.b)

        # Case 3: self == other
        # Formula (x3,y3)=(x1,y1)+(x1,y1)
        # s=(3*x1**2+a)/(2*y1)
        # x3=s**2-2*x1
        # y3=s*(x1-x3)-y1
        if self == other:
            s = (3 * self.x**2 + self.a) / (2 * self.y)
            x = s**2 - 2 * self.x
            y = s * (self.x - x) - self.y
            return self.__class__(x, y, self.a, self.b)

    # tag::source3[]
    def __rmul__(self, coefficient):
        coef = coefficient
        current = self  # <1>
        result = self.__class__(None, None, self.a, self.b)  # <2>
        while coef:
            if coef & 1:  # <3>
                result += current
            current += current  # <4>
            coef >>= 1  # <5>
        return result
    # end::source3[]

```
<!-- #endregion -->

# choosing generator point P

Here's a paper on how secp256 generator was randomly selected

https://www.secg.org/sec2-v2.pdf

Trick is 
1. given N choose the subgroup order n to be a prime divisor of N
1. compute cofactor h = N/n
1. choose random point P -- this could be fun!
1. compute G=hP
1. if G != 0, then n(hP)=0 and G is a generator of the whole curve
1. if G = 0, go back to (3)

```python
def is_prime(n):
    prime_flag = 0
      
    if(n > 1):
        for i in range(2, int(np.sqrt(n)) + 1):
            if (n % i == 0):
                prime_flag = 1
                break
        if (prime_flag == 0):
            return True
    return False
```

```python
def prime_divisor(n):
    """find the largest prime divisor of n"""
    for _ in list(divisors(n))[::-1]:
        if is_prime(_):
            return _

def cofactor(order, n):
    return int(order/n)

prime_divisor(42)
```

```python
e1 = dict(p=29, a=-1, b=1)
```

```python
order(**e1)
```

```python
prime_divisor(37)
```

```python
cofactor(37, 37)
```

```python

N_ = order(**e1)
N_
```

```python
H__ = (12, 8)

G__ = multiply_mod(H__, 37, **e1)
G__
```

## Secret sharing

Since $H=dG$ is uninvertable, we can define a public/private key pairs as ($d_{priv}, H_{pub}$)

Suppose alice has public $H_a$ and private key $d_a$ and similarly for Bob $H_b$ and $d_b$. Alice and Bob may compute a shared secret $S = d_aH_b = d_bH_a = d_ad_bG$. This shared secret can be used by either party to encrypt blobs of data without fear of eavesdropping.

```python
d_a = 5
H_a = multiply_mod(H__, d_a, **e1)

d_b = 9
H_b = multiply_mod(H__, d_b, **e1)

S_b = multiply_mod(H_a, d_b, **e1) # Bob generates secret key using Alice's pub key
S_a = multiply_mod(H_b, d_a, **e1) # Alice generates secret key using Bob's pub key
assert S_a == S_b # check that Alice and Bob get the same result
S_a
```

## ECDSA

1. choose $k \in [1, n]$ for subgroup order $n$
1. calculate $P=kG$
1. r = P_x mod n # get x coordinate of P (unless $r=0$)
1. $s = k^{-1}(z+rd_A) modn$

```python
from random import randrange

G_ = (9,24) # generator
n_ = subgroup_order(G_, **e1) # should be 37 for {'p': 29, 'a': -1, 'b': 1}
G_, n_
```

```python
e1
```

```python
d_a = 5 #alice's private key
H_a = multiply_mod(G_, d_a, **e1) #alice's pub key
H_a
```

```python
import random

def get_rk(G_, n_, seed=0, k=None, **ec):
    """find point for subgroup order n, random integer k in [1,n] and elliptic curve ec"""
    random.seed(seed)
    r = 0
    if k is None:
        while r == 0:
            k = randrange(n_)%n_ # random in [1,n]
            P_ = multiply_mod(G_, k, **ec)
            r, _ = P_
            r = r%n_
    else:
        P_ = multiply_mod(G_, k, **ec)
        r, _ = P_
        r = r%n_
    return P_, k
```

```python
P_, k = get_rk(G_, n_, k=3, **e1)
r, _ = P_
print(r, k)
```

```python
multiply_mod(G_, 3, **e1)
```

```python
assert P_ == multiply_mod(G_, k, **e1)
```

```python
k_inv = modinv(k, n_) # only works for n_ prime
k_inv
```

```python
def get_s(k, z, r, d_a, n):
    k_inv = modinv(k, n)
    return (k_inv*((z%n_ + (r*d_a)%n_)%n_))%n_
```

```python
z = randrange(n_)%n_ # represents hash with same bitlength as n_
```

```python
s = get_s(k, z, r, d_a, n_) # s = k^-1(z+r*d_a) mod n_
```

```python
assert k == get_s(s, z, r, d_a, n_) # k = s^-1(z+r*d_a) mod_n
```

```python
sig_az = (r, s) # alice's signature on z message
sig_az
```

To verify signature:

1. $u_1 = s^{-1}z \text{modn}$
1. $u_2 = s^{-1}r \text{modn}$
1. $P = u_1 G + u_2 H_a$

```python
u_1 = (modinv(s, n_)*z)%n_
u_1
```

```python
u_2 = (modinv(s, n_)*r)%n_
u_2
```

```python
multiply_mod(G_, u_1, **e1), multiply_mod(H_a, u_2, **e1)
```

```python
a_ = multiply_mod(G_, u_1, **e1)
a_
```

```python
b_ = multiply_mod(H_a, u_2, **e1)
b_
```

```python
add_mod(a_, b_, **e1)
```

```python
P_result = add_mod(multiply_mod(G_, u_1, **e1), multiply_mod(H_a, u_2, **e1), **e1)
P_result
```

```python
if r != P_result[0]%n_:
    raise AssertionError(f"signature does not match! {r} != {P_result[0]}")
```

Compare with Jimmy Song's version

```python
import sys

sys.path.append('../programmingbitcoin/code-ch03/')

from ecc import Point, FieldElement
```

Choose the generator point

```python
n_
```

```python
G_ = Point(FieldElement(9, e1['p']),
           FieldElement(24, e1['p']),
           FieldElement(e1['a']%e1['p'], e1['p']),
           FieldElement(e1['b']%e1['p'], e1['p'])
          )
G_
```

```python
add_mod((12, 8), (12, 8), **e1)
```

```python
P_test = Point(FieldElement(12, e1['p']),
           FieldElement(8, e1['p']),
           FieldElement(e1['a']%e1['p'], e1['p']),
           FieldElement(e1['b']%e1['p'], e1['p'])
          )
```

```python
P_test + P_test
```

```python
P_ = (9,24)
G_test = G_
for i in range(1,15):
    print(G_test, P_)
    P_ = add_mod(P_, P_, **e1)
    G_test = G_test + G_test
```

```python
m_ = 3
```

```python
m_ >>= 1
```

```python
m_
```

Choose k

```python
k = 3
```

```python
P_ = k*G_
P_
```

```python
r = P_.x.num%n_
```

Set Alice's priv key

```python
d_a = 5
d_a
```

Compute Alice's pub key

```python
H_a = d_a*G_
H_a
```

Get Alice's signature

```python
s = get_s(k, z, r, d_a, n_)
assert k == get_s(s, z, r, d_a, n_) # check math
s
```

Check signature

```python
u_1 = (modinv(s, n_)*z)%n_
u_1
```

```python
u_2 = (modinv(s, n_)*r)%n_
u_2
```

```python
assert u_1*G_ + u_2*H_a == P_
```

# Dashboard

```python
import dash_bootstrap_components as dbc
```

```python
from psidash.psidash import load_app

app = load_app(__name__, 'elliptic.yaml')

if __name__ == '__main__':
    app.run_server(host='0.0.0.0',
                   port=8050,
                   mode='external',
                   extra_files=["elliptic.yaml", "elliptic/dashboard.py"],
                   debug=True)
```

```python

```

```python

from elliptic.dashboard import elliptic, point_in_curve

a = elliptic(37, 0, 7)
```

```python

```

```python
import numpy as np
```

```python
%load_ext autoreload

%autoreload 2
```

$ (3, 4) $

```python

```
