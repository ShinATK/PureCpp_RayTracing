#gamedev #raytracing
- [# Ray Tracing in One Weekend](https://raytracing.github.io/v3/books/RayTracingInOneWeekend.html#overview)
---
# 1 总览

相关链接：
- 项目地址， [# RayTracing (Online Books)](https://github.com/RayTracing/raytracing.github.io/)
- 作者相关 blog， [In One Weekend](https://in1weekend.blogspot.com/)

---
# 2 输出图片

PPM，可移植像素图格式（使用 RGB 颜色）
- 一种跨平台的图像格式，统称 PNM 格式
- 其他：可移植灰度图格式 PGM（灰度图），可移植位图格式 PBM（单色）
参考：[PBM 格式 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/PBM%E6%A0%BC%E5%BC%8F)

下面是一个 PPM 图片的 example
```ppm
P3 # 文件格式
4 4 # 宽度和高度
15 # 表示最大颜色
# 每个像素的值（三个为一组RGB）
0 0 0 0 0 0 0 0 0 15 0 15
0 0 0 0 15 7 0 0 0 0 0 0
0 0 0 0 0 0 0 15 7 0 0 0
15 0 15 0 0 0 0 0 0 0 0 0
```

利用 C++ 输出一个 PPM 图片文件：
- 使用重定向运算符 `>` 将程序输出到一个 `image.ppm` 文件中：`xxx.exe > image.ppm`
- 使用 `std::cerr` 将进度提示输出到 error output stream 中
- 查看 PPM 文件的网站：[PPM Viewer](https://www.cs.rhodes.edu/welshc/COMP141_F16/ppmReader.html)

![](img/Pasted%20image%2020240119171528.png)

---
# 3 vec3 Class

建立三维向量的类：vec3
- 为了区分点的三维坐标和颜色的 RGB，另建立一个 color 子类


---
# 4 光线、简单的相机和背景

## 4.1 光线 Rays

定义光线形式：

$$
\vec{P}(t) = \vec{A} + t\cdot\vec{b}
$$

- $\vec{P}$：三维空间的一个位置
- $\vec{A}$：光线起始位置
- $\vec{b}$：光线的方向
- $t$：控制光线的传播位置

- **核心**：根据起点坐标和方向向量就可以确定一条光线


## 4.2 向场景投射光线

建立的相机空间如图，

![](img/Pasted%20image%2020240120180604.png)

投射光线的具体步骤可以分解为（相机空间中）：
1. 从相机发射光线到像素（相机在原点，像素位置向量即光线方向）
2. 计算光线与场景的交点（交点在什么物体上）
3. 获取交点处的颜色（物体颜色）

首先设置相机以及屏幕的一些参数：[# focal length 和 focus distance 的区别](https://borislavkostov.wordpress.com/articles/macro-photography-eng/focal-length-focusing-distance-working-distance/)

```cpp
// Camera
auto viewport_height = 2.0;
auto viewport_width = aspect_ratio * viewport_height;
auto focal_length = 1.0;

auto origin = point3(0, 0, 0);
auto horizontal = vec3(viewport_width, 0, 0);
auto vertical = vec3(0, viewport_height, 0);
auto lower_left_corner = origin - horizontal/2 - vertical/2 - vec3(0, 0, focal_length);
```

- 注意宽高比和要渲染的图片相同

for 循环中遍历屏幕像素生成从原点（相机）发出的光线：

```cpp
for(...)
{
	for(...)
	{
		auto u = double(i) / (image_width - 1);
		auto v = double(j) / (image_height - 1);
		ray r(origin, lower_left_corner + u*horizontal + v*vertical - origin);
		
		getIntersection(scene, ray);
		getColor(IntersectionPoint);
	}
}
```

之后需要计算交点位置并获取交点处颜色，这里将设置一个随着 y 轴坐标变化，而出现蓝白渐变的图片，所以单独设置一个函数来产生这种渐变颜色

```cpp
color ray_color(const ray& r)
{
	vec3 unit_direction = unit_vector(r.direction());
	auto t = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}
```

所以每个交点处的像素颜色为：

```cpp
for(...)
{
	for(...)
	{
		generate_ray(); // represent for the code ahead
		
		color pixel_color = ray_color(r)
		write_color(std::cout, pixel_color);
	}
}
```

最后可以得到的图片如下：

![](img/Pasted%20image%2020240120170929.png)

---
# 5 添加球面

在章节 4 中只是从 camera 发出射线打到像素，并根据像素的 y 值设置颜色，并没有真正进行交点的计算，这里将在场景中设置一个球体，并计算光线与球体的交点

## 5.1 光线与球面求交

三维空间球面公式：

$$
(x-C_x)^2 + (y-C_y)^2 + (z-C_z)^2 = r^2
$$

写成三维向量表示：

$$
(\vec{P}-\vec{C})\cdot(\vec{P}-\vec{C}) = r^2
$$

结合光线公式：

$$
\vec{P}(t) = \vec{A} + t\vec{b}
$$

联立公式求解：

$$
t^2\vec{b}\cdot\vec{b} + 2t\vec{b}\cdot(\vec{A}-\vec{C}) + (\vec{A}-\vec{C})\cdot(\vec{A}-\vec{C}) - r^2 = 0
$$

求解存在三种情况：
- 没有交点
- 一个交点
- 两个交点

![](img/Pasted%20image%2020240120180551.png)

## 5.2 第一个光线追踪图片

根据 5.1 中的分析，是否有交点可以直接利用一元二次方程的求根公式

```cpp
bool hit_sphere(const point3& center, double radius, const ray& r)
{
	vec3 oc = r.origin() - center;
	auto a = dot(r.direction(), r.direction());
	auto b = 2.0 * dot(oc, r.direction());
	auto c = dot(oc, oc) - radius * radius;
	auto discriminant = b*b - 4*a*c;
	return (discriminant > 0);
}
```

这里设置球体为红色，即 `color(1,0,0)`，其他位置仍然保持蓝白渐变
- 即存在交点，颜色设置为红色
- 否，则保持原样

```cpp
color ray_color(const ray& r)
{
	if(hit_sphere(point3(0,0,-1),0.5,r))
		return color(1,0,0);
	vec3 unit_direction = unit_vector(r.direction());
	auto t = 0.5 * (unit_direction.y() + 1.0);
	
	return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```

得到的图片如下：

![](img/Pasted%20image%2020240120180537.png)

此时还缺少很多：
- 阴影
- 反射光线
- 以及其他物体

而此时最大的问题是：**如果将球体位置设置在 z=+1，仍然会得到同样的图片**，这说明看向 z 的负方向但却同时能看到 z 的正方向上的物体
- 因为此时只是判断有没有交点，而没有判断交点对应的 t 的正负
- 显然，上面描述的这种问题，是当 t<0 时发生的

---
# 6 表面法线和多个物体

首先，增加对交点是否满足 t>0 的条件判断

```cpp
double hit_sphere(const point3& center, double radius, const ray& r)
{
	vec3 oc = r.origin() - center;
	auto a = dot(r.direction(), r.direction());
	auto b = 2.0 * dot(oc, r.direction());
	auto c = dot(oc, oc) - radius * radius;
	auto discriminant = b*b - 4*a*c;

	// 增加判断部分
	if(discriminant < 0)
		return -1.0;
	else
		return (-b - sqrt(discriminant) ) / (2.0*a);
}
```

## 6.1 利用表面法线着色

之前的计算中，只进行了是否与球体碰撞的判断，并直接设置了固定的颜色

这里将引入球面法线，给球面赋予颜色变化

```cpp
color ray_color(const ray& r)
{
	// 获取交点的 t
	auto t = hit_sphere(point3(0,0,-1), 0.5, r);
	if(t > 0.0)
	{
		// 计算交点法线
		vec3 N = unit_vector(r.at(t) - vec3(0,0,-1));
		// 返回交点处颜色
		return 0.5*color(N.x()+1, N.y()+1, N.z()+1);
	}

	vec3 unit_direction = unit_vector(r.direction());
	auto t = 0.5 * (unit_direction.y() + 1.0);
	return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```

![Alt text](../RayTracing_in_OneWeekend/img/shading_witNormal.png)

## 6.2 简化交点计算

简化的核心推导：$b=2h$


$$\begin{align*}
	&\frac{-b\pm\sqrt{b^2-4ac}}{2a} \\
	&= \frac{-2h\pm\sqrt{(2h)^2-4ac}}{2a} \\
	&= \frac{-h\pm\sqrt{h^2-ac}}{a}
\end{align*}$$

## 6.3 可碰撞物体的抽象类

将球面扩大为一切可发生碰撞的抽象类，其应当用于计算其碰撞点信息
- 碰撞点的坐标
- 碰撞点的法线
- 碰撞点对应光线的 t 值

```cpp
#ifndef HITTABLE_H
#define HITTABLE_H

#include "ray.h"

class hit_record {
  public:
    point3 p;
    vec3 normal;
    double t;
};

class hittable {
  public:
    virtual ~hittable() = default;

    virtual bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const = 0;
};

#endif
```

进一步，建立球面的类，继承自可碰撞类，注意析构函数和成员函数需要在继承类中完成

```cpp
#ifndef SPHERE_H
#define SPHERE_H

#include "hittable.h"
#include "vec3.h"

class sphere : public hittable {
  public:
    sphere(point3 _center, double _radius) : center(_center), radius(_radius) {}

    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override {
        vec3 oc = r.origin() - center;
        auto a = r.direction().length_squared();
        auto half_b = dot(oc, r.direction());
        auto c = oc.length_squared() - radius*radius;

        auto discriminant = half_b*half_b - a*c;
        if (discriminant < 0) return false;
        auto sqrtd = sqrt(discriminant);

        // Find the nearest root that lies in the acceptable range.
        auto root = (-half_b - sqrtd) / a;
        if (root <= ray_tmin || ray_tmax <= root) {
            root = (-half_b + sqrtd) / a;
            if (root <= ray_tmin || ray_tmax <= root)
                return false;
        }

        rec.t = root;
        rec.p = r.at(rec.t);
        rec.normal = (rec.p - center) / radius;

        return true;
    }

  private:
    point3 center;
    double radius;
};

#endif
```

## 6.4 定义正向面和反向面

一个面需要有正反之分，同样需要在碰撞点记录，光线是从外部还是从内部与面相交的，故增加`bool front_face`以区分交点位于正面还是反面，具体计算则是根据光线方向和面的朝外向量点积

```cpp
class hit_record {
  public:
    point3 p;
    vec3 normal;
    double t;
    bool front_face;

    void set_face_normal(const ray& r, const vec3& outward_normal) {
        // Sets the hit record normal vector.
        // NOTE: the parameter `outward_normal` is assumed to have unit length.

        front_face = dot(r.direction(), outward_normal) < 0;
        normal = front_face ? outward_normal : -outward_normal;
    }
};
```

同时记得给`sphere`类中，增加交点的正反记录

```cpp
class sphere : public hittable {
  public:
    ...
    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const {
        ...

        rec.t = root;
        rec.p = r.at(rec.t);
        vec3 outward_normal = (rec.p - center) / radius;
        rec.set_face_normal(r, outward_normal);

        return true;
    }
    ...
};
```

## 6.5 可碰撞对象的列表

除了球类，场景中还可能有其他会与光线发生交点的物体，所以这里使用一个列表类用以存储场景中可碰撞的对象

```cpp
#ifndef HITTABLE_LIST_H
#define HITTABLE_LIST_H

#include "hittable.h"

#include <memory>
#include <vector>

using std::shared_ptr;
using std::make_shared;

class hittable_list : public hittable {
  public:
    std::vector<shared_ptr<hittable>> objects;

    hittable_list() {}
    hittable_list(shared_ptr<hittable> object) { add(object); }

    void clear() { objects.clear(); }

    void add(shared_ptr<hittable> object) {
        objects.push_back(object);
    }

    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override {
        hit_record temp_rec;
        bool hit_anything = false;
        auto closest_so_far = ray_tmax;

        for (const auto& object : objects) {
            if (object->hit(r, ray_tmin, closest_so_far, temp_rec)) {
                hit_anything = true;
                closest_so_far = temp_rec.t;
                rec = temp_rec;
            }
        }

        return hit_anything;
    }
};

#endif
```

