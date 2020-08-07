--- 
title: Comparison
description: Explanation of the various pitfalls in comparing floating-point numbers.
---

Due to rounding errors, most [floating-point](/formats/fp/) numbers end up being slightly imprecise. As long as this
imprecision stays small, it can usually be ignored. However, it also means that numbers expected
to be equal (e.g. when calculating the same result through different correct methods) often differ
slightly, and a simple equality test fails. For example:

		float a = 0.15 + 0.15
		float b = 0.1 + 0.2
		if(a == b) // can be false!
		if(a >= b) // can also be false!

Don't use absolute error margins
--------------------------------
The solution is to check not whether the numbers are exactly the same, but whether their difference is
very small. The error margin that the difference is compared to is often called *epsilon*. 
The most simple form:

		if( Math.abs(a-b) < 0.00001) // wrong - don't do this

This is a bad way to do it because a fixed epsilon chosen because it "looks small" could actually be way too
large when the numbers being compared are very small as well. The comparison would return "true" for numbers that are quite different. And when the numbers are very large, the epsilon
could end up being smaller than the smallest rounding error, so that the comparison always returns "false".
Therefore, it is necessary to see whether the *relative error* is smaller than epsilon:

		if( Math.abs((a-b)/b) < 0.00001 ) // still not right!

Look out for edge cases
-----------------------
There are some important special cases where this will fail: 

* When both `a` and `b` are zero. `0.0/0.0` is "not a number", which causes an exception on some platforms, or returns false for all comparisons. 
* When only `b` is zero, the division yields "infinity", which may also cause an exception, or is greater than epsilon even when `a` is smaller. 
* It returns `false` when both `a` and `b` are very small but on opposite sides of zero, even when they're the smallest possible non-zero numbers.

Also, the result is not commutative (`nearlyEquals(a,b)` is not always the same as `nearlyEquals(b,a)`). To fix these problems, the code has to get a lot more complex, so we really need to put it into a function of its own:

		public static boolean nearlyEqual(float a, float b, float epsilon) {
			final float absA = Math.abs(a);
			final float absB = Math.abs(b);
			final float diff = Math.abs(a - b);

			if (a == b) { // shortcut, handles infinities
				return true;
			} else if (a == 0 || b == 0 || diff < Float.MIN_NORMAL) {
				// a or b is zero or both are extremely close to it
				// relative error is less meaningful here
				return diff < (epsilon * Float.MIN_NORMAL);
			} else { // use relative error
				return diff / Math.min((absA + absB), Float.MAX_VALUE) < epsilon;
			}
		}


This method [passes tests](../NearlyEqualsTest.java) for many important special cases, but as you can see, it
uses some quite non-obvious logic. In particular, it has to use a completely different definition of error margin
when `a` or `b` is zero, because the classical definition of relative error becomes meaningless in those cases.

There are some cases where the method above still produces unexpected results (in particular, it's much stricter when one value is nearly zero than when it is exactly zero), and some of the tests
it was developed to pass probably specify behaviour that is not appropriate for some applications. Before using it, make sure it's appropriate for your application!

Comparing floating-point values as integers
-------------------------------------------
There is an alternative to heaping conceptual complexity onto such an apparently simple task: instead of comparing `a` and `b` as [real numbers](http://en.wikipedia.org/wiki/Real_numbers), we can think about them as discrete steps and define the error margin as the maximum number of possible floating-point values between the two values. 

This is conceptually very clear and easy and has the advantage of implicitly scaling the relative error margin with the magnitude of the values. Technically, it's a bit more complex, but not as much as you might think, because IEEE 754 floats are designed to maintain their order when their bit patterns are interpreted as integers.

However, this method does require the programming language to support conversion between floating-point values and integer bit patterns. Read the [Comparing floating-point numbers](/references/) paper for more details.



| 标题 | 描述                       |
| ---- | -------------------------- |
| 比较 | 解释浮点数比较中的不同陷阱 |

舍入错误，使得大部分浮点数结果有微小误差。只要误差足够小，通常都可以忽略。然而，这也意味着两个理论上相等的数（例如：当用不同的正确方法计算相同的结果时）比较结果可能不相等。例如：

		float a = 0.15 + 0.15
		float b = 0.1 + 0.2
		if(a == b) // can be false!
		if(a >= b) // can also be false!

不要使用绝对误差
--------------------------------

解决方法是，不要去判断两个数是否完全相等，而是去判断两个数是否相差足够小。与之比较的误差范围通常称为_浮点可表示的最小值_.

		if( Math.abs(a-b) < 0.00001) // wrong - don't do this

这不是一个好方法，因为一个选定浮点可以表示的最小值只是“看起来小”，但是对于两个也特别小的数进行比较，这个浮点可表示的最小值就相对来说太大了。这样两个完全不等的数比较会得到一个“相等”的结果。当比较数特别大的时候，浮点可表示的最小值可能比舍入最小误差还要小，比较往往会得到“不相等”的结果。因此，有必要判断_相关误差_是否小于浮点可表示的最小值:

		if( Math.abs((a-b)/b) < 0.00001 ) // still not right!

注意边界情况
-----------------------

有一些重要的特殊情况可能会出错：

* 当 `a` 和 `b`  都是0。 `0.0/0.0` 是个”非数“， 这种情况在一些平台上会产生异常，或者所有的比较返回错误。
  * 当只有`b`是0，除法会产生“无穷，这可能会产生异常，或者即使`a`更小结果也会比浮点可表示的最小值小。
* 当`a` 和 `b`是符号相反的极小数时，也可能会返回“错误‘，尽管它们可能是是最小的非0数。