
+++
title = "Some calculus fun"
description = ""
tags = [
    "people",
]
date = "2019-01-13"
categories = [
    "Math",
]
menu = "main"
math = true
+++

On one of my commutes home a couple of days ago I had for whatever reason the desire to calculate to the circumference of a circle using calculus. The idea was to calcute the lenght of the base of a triagle whose sides are the radius, summing all those distances as they wrap a circle. As the angle between the two sides of the triangle of lenght $r$ tends to 0, the sum of the bases should equal the circumference of the circle . 
If we know that the radius of a cirlce is $r$. By the law of cosines we know that the side $x$ is equal to $\sqrt{r^2+r^2-2r^2\cos(\theta)}$. Simplifying we get
$$ x = r\sqrt{2-2\cos(\theta)}$$
The equation above as is is still a bit hard to work with so after some searching and remembering trigonometric identities.
$$ \sqrt{\frac{1 - \cos(\theta)}{2}} = \sin(\theta/2) $$
Then
$$ \frac{x}{2} = r \sqrt{\frac{2 - 2\cos(\theta)}{4}} $$
$$ x = 2r \sqrt{\frac{1 - \cos(\theta)}{2}} $$
$$ x = 2r\sin(\frac{\theta}{2}) $$

The above expression is way easier to reason about and work with. Also noticed I already have some of the terms that I am looking for, namely $2 r$. Now with the above function we can imagine that as the angle $\theta$ approaches 0 the value of the sum of all x(arounde the circle) approaches the circumference $C$ for a circle or radius $r$. Since all $x$ are the same size we can multiply by $2\pi/\theta$. Then
$$ C = \lim\_{\theta\to0}\frac{2\pi}{\theta} 2r\sin(\theta/2) $$
$$ C = \lim\_{\theta\to0}4\pi r \frac{\sin(\theta/2)}{\theta}  $$
$$ C = 4\pi r \lim\_{\theta\to0}\frac{\sin(\theta/2)}{\theta}  $$

So the problem now boils down to calculating $\lim\_{\theta\to0}\frac{\sin(\theta/2)}{\theta}$. Given that we have a division by 0, it is not super straighfoward to calculate the limit, but easy enough using other rules. Such
$$ \lim\_{x \rightarrow a} \frac{f(x)}{g(x)} =  \lim\_{x \rightarrow a} \frac{f'(x)}{g'(x)}  $$
$$ \lim\_{\theta\to0} \frac{\sin(\theta/2)}{\theta} =\lim\_{\theta\to0} \frac{1}{2}\frac{\cos(\theta/2)}{1} $$
$$ \lim\_{\theta\to0} \frac{\sin(\theta/2)}{\theta} = \frac{1}{2} \cos(0) = \frac{1}{2} $$

Finally:
$$ C = 4 \pi r \frac{1}{2}$$
$$ C = 2 \pi r $$






