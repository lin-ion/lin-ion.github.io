---
layout: single
title: "삼각형 레스터라이제이션"
date: 2025-02-27
categories: graphics
mathjax: true
read_time: false
---

<!-- {: .notice--info} -->

[Tinyrenderer](https://github.com/ssloy/tinyrenderer/wiki) Lesson 2 에서는 세 정점으로 이루어진 삼각형의 내부를 색칠하는 방법을 다루었다.

## Line sweeping

첫 번째로 제시된 Line sweeping 방법은 <cb>Bresenham's Algorithm</cb>을 응용한 것으로, 삼각형의 한 정점에서 다른 두 정점으로 가는 선의 기울기를 계산하고 스캐닝 라인을 한 칸씩 옮겨가듯이 순차적으로 삼각형의 내부 픽셀을 채워낸다.

이 방법은 최소한의 연산으로 삼각형을 칠할 수 있는 매우 효율적인 알고리즘이지만 CPU집약적이며 병렬화가 힘들다.

> Moreover, it is really an old-school approach designed for mono-thread CPU programming.

## 주어진 좌표가 삼각형 내부에 포함되는지 검사

```cpp
triangle(vec2 points[3]) {
  vec2 bbox[2] = find_bounding_box(points);
  for (each pixel in the bounding box) {
    if (inside(points, pixel)) {
      put_pixel(pixel);
    }
  }
}
```

이 방법에서는 정점 좌표와 픽셀 좌표가 주어졌을 때 해당 픽셀 좌표가 삼각형의 내부인지 외부인지를 판단한다. 픽셀이 삼각형에 포함되는지의 여부는 픽셀의 <cb>barycentric coordinates</cb>를 구하여 알 수 있다.

- 첫 번째 방법은 <cb>삼각형이 어떤 픽셀들을 포함하는지</cb>를 계산하기 위해서 3개의 정점 좌표를 입력받고 삼각형 내부의 픽셀을 한꺼번에 계산하므로 작업의 분할이 어려운 반면에
- 두 번째 방법은 <cb>어떤 픽셀이 삼각형에 포함되는지</cb>를 계산하기 위해서 3개의 정점 좌표와 픽셀의 좌표를 받고 픽셀이 삼각형에 포함되는지만 계산하므로 작업을 픽셀을 기준으로 분할할 수 있다. 계산한 좌표는 셰이더에서 유용한 정보로 활용될 수 있다.

### Barycentric coordinates(무게중심좌표) [wikipedia](https://en.wikipedia.org/wiki/Barycentric_coordinate_system)

<!-- https://erkaman.github.io/posts/fast_triangle_rasterization.html -->

삼각형 $ABC$ 에 대한 점 $P$ 의 좌표는 다음과 같다.

$$
  \begin{array} \
    P=(1-u-v)A+uB+vC
  \end{array}
$$

$(1-u-v, u, v)$ 는 점 $P$ 가 삼각형의 정점 $(A, B, C)$ 와 얼마나 가까운지에 대한 parameter 또는 weight로 생각할 수 있으며,
어떠한 값도 음수가 아니라면 해당 좌표는 삼각형의 내부에 위치한 점으로 볼 수 있다.

$$
  \begin{array} \
  P = (1-u-v)A+uB+vC \\
  (A-P)+u(B-A)+v(C-A) = \overrightarrow{0} \\
  u\overrightarrow{AB} + v\overrightarrow{AC} + \overrightarrow{PA} = \overrightarrow{0}
  \end{array}
$$

식을 $x,y$ 각각에 대해 분리하고 내적 형태로 표현해보자.

$$
  \left\{
    \begin{array}{l}
      \left[
        \begin{array}\
            u & v & 1
        \end{array}
      \right]
      \left[
        \begin{array}\
            AB_x\\
            AC_x\\
            PA_x
        \end{array}
      \right] = 0 \\
      \left[
        \begin{array}\
            u & v & 1
        \end{array}
      \right]
      \left[
        \begin{array}\
            AB_y\\
            AC_y\\
            PA_y
        \end{array}
      \right] = 0
    \end{array}
  \right.
$$

벡터의 내적이 $0$ 이면 두 벡터는 직교(Orthogonal)이다.

즉, 벡터 $(u,v,1)$ 은 위 식의 두 벡터에 대해서 동시에 직교하는 벡터이므로, 두 벡터의 외적을 구하고 세 번째 요소의 값이 $1$ 이 되도록 정규화해서 점 $P$ 의 Barycentric coordinate: $u, v$ 를 알아낼 수 있다.
우선 두 벡터의 외적을 먼저 구해보자.

$$
  \begin{array}\
    a = AB, &
    b = AC, &
    c = PA
  \end{array}
$$

$$
  \begin{array}{l}\
    (a_x,b_x,c_x) \times (a_y,b_y,c_y)
    = \hat{i}\left|
      \begin{array}\
        b_x & b_y \\
        c_x & c_y
      \end{array}
    \right|
    - \hat{j}\left|
      \begin{array}\
        a_x & a_y \\
        c_x & c_y
      \end{array}
    \right|
    + \hat{k}\left|
      \begin{array}\
        a_x & a_y \\
        b_x & b_y
      \end{array}
    \right|
    \\ \\
    = \begin{array}\
      \left[
        \begin{array}\
          b_x c_y - c_x b_y \\
          c_x a_y - a_x c_y \\
          a_x b_y - b_x a_y
        \end{array}
      \right]
      & \left[
        \begin{array}\
          u \\
          v \\
          1 \\
        \end{array}
      \right]
      = \left[
        \begin{array}\
          (b_x c_y - c_xb_y) / (a_xb_y-b_xa_y) \\
          (-c_xa_y+a_xc_y) / (a_xb_y-b_xa_y) \\
          (a_xb_y-b_xa_y) / (a_xb_y-b_xa_y)
        \end{array}
      \right]
    \end{array}
  \end{array}
$$

행렬을 전치하더라도 determinant는 같으므로, 결과로 나온 벡터의 각 스칼라 값은 $(AC \times PA, PA \times AB, AB \times AC)$ 를 나타내는 것을 알 수 있다.

- $(AC \times PA)$ 는 점 $P$ 가 벡터 $AC$ 의 우측에 위치하는지를 판단하고,
- $(PA \times AB)$ 는 점 $P$ 가 벡터 $AB$ 의 좌측에 위치하는지를 판단한다.
- $(AB \times AC)$ 는 두 벡터가 이루는 평행사변형의 면적과 같으므로 삼각형 $ABC$ 의 넓이의 $2$ 배와 같다.

앞의 두 값을 $(AB \times AC)$ 로 나누어서 우리가 원하는 $u, v$ 를 구할 수 있다.
