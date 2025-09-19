---
title: 最大公约数
---

#（gcd)最大公约数的三种求法
<!-- more -->
## 第一种求法：

```c
int gcd(int a, int b) {

	if (a == b)
		return a;
	if (a > b)
		return gcd(a, a - b);
	if (a < b)
		return gcd(b, b - a);
}
```
## 第二种求法：

```c
int gcd(int a, int b) {//默认a>b!

	if (b == 0)
		return a;
	else
		return gcd(b, a % b);
}
```

### 其实第二种方法的过程就是：

```c

​```c
int gcd(int a, int b) {

	if (a < b) {
		swao(a, b);
	}
	while (b != 0) {
		int r = a % b;
		a = b;
		b = r;
	}
	return a;
}
```

```

```