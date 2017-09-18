title: 笔试题
date: 2017-06-04 00:45:35
tags:
  - java
  - 笔试题
---

## 第n个无平方数因数的数

如果一个正整数不能被大于1的完全平方数所整除，那么我们就将该数称为无平方数因子的数。例如，靠前的一些无平方数因数的数是{1,2,3,5,6,7,10,11,13,14,15,17,19…}。给出一个整数n,返回第n个无平方数因子的数。

输入:

> 输入一个整数n. n 的取值范围为1到1,000,000(其中包括1和1,000,000)

输出:
> 返回第n个无平方数因数的数

举例:
> n = 13, 返回19.

思路:
> 统计 start = 1 到 end = n 中平方数倍数的个数cnt（注意避免重复），如果cnt>0， 继续统计 start = end+1 到 end = end+cnt 中平方数倍数的个数k，直至 cnt==0，输出 end 

<!-- more -->

```java

import java.util.*;

public class Main {

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int cnt, start = 1, end = scanner.nextInt();
    while ((cnt=count(start, end)) > 0) {
      start = end + 1;
      end += cnt;
    }
    System.out.println(end);

  }

  private static int count(int start, int end) {
    Set<Integer> factors = new HashSet<Integer>();
    for (int i = 2; i * i <= end; i++) {
      int square = i * i;
      for (int s = start / square, e = end / square; s <= e; s++) {
        int factor = s * square;
        if (factor >= start && factor <= end) {
          factors.add(factor);
        }
      }
    }
    return factors.size();
  }
}

```

## n阶乘结果0的个数

输入:

> 输入一个整数n

输出:

> 输出n阶乘结果0的个数

例子:

> n = 100， 返回 24

思路:

> 统计n阶乘因式分解中5的个数

```java
import java.util.*;

public class Main {

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int n = scanner.nextInt();
    int cnt = 0;
    while(n>0){
      cnt+=(n/5);
      n/=5;
    }
    System.out.println(cnt);
  }
}

```

## 出现一次的数字

描述:

> 给定一个数组，仅有一个数字出现了一次，其他数字都出现两次，请找出这个出现一次的数字

leetcode: [https://leetcode.com/problems/single-number/#/description](https://leetcode.com/problems/single-number/#/description)

思路:

>  a^b^a = b

```java

public class Solution {
  
  public int singleNumber(int[] nums) {
    int len = nums.length;
    int num = 0;
    while(--len>=0){
      num ^= nums[len];
    }
    return num;
  }
}

```

## 反转链表

leetcode: [https://leetcode.com/problems/reverse-linked-list/#/description](https://leetcode.com/problems/reverse-linked-list/#/description)

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */

public class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode P, Q = null;
        while(head!=null){
            P = head;
            head = head.next;
            P.next = Q;
            Q = P;
        }
        return Q;
    }
}
```

## 跳台阶

描述:
> 一只青蛙一次可以跳上1级台阶，也可以跳上2级。求该青蛙跳上一个n级的台阶总共有多少种跳法。

牛客: [https://www.nowcoder.com/questionTerminal/8c82a5b80378478f9484d87d1c5f12a4](https://www.nowcoder.com/questionTerminal/8c82a5b80378478f9484d87d1c5f12a4)

思路:
> f(n) = f(n-1) + f(n-2)

```java

public class Solution {
  public int JumpFloor(int target) {
    int previous = 0, next = 1;
    while(--target>=0){
      next += previous;
      previous = next - previous;
    }
    return next;
  }
}

```

## 二进制位中1的个数

leetcode: [https://leetcode.com/problems/number-of-1-bits/#/description](https://leetcode.com/problems/number-of-1-bits/#/description)

例子:

输入 5， 5的二进制表示为101, 所以输出为2

思路:

> 规律：n &= (n-1) 二进制中1的位数减一， 如 111 & 110 = 110 

```java
public class Solution {
  // you need to treat n as an unsigned value
  public int hammingWeight(int i) {
    int cnt = 0;
    while(i!=0){
      cnt ++;
      i &= (i-1);
    }
    return cnt;
  }
}

```

## 数组变换

描述:
> 牛牛有一个数组，里面的数可能不相等，现在他想把数组变为：所有的数都相等。问是否可行。
> 牛牛可以进行的操作是：将数组中的任意一个数改为这个数的两倍。
> 这个操作的使用次数不限，也可以不使用，并且可以对同一个位置使用多次。

输入描述:
> 输入一个正整数N (N <= 50)
> 接下来一行输入N个正整数，每个数均小于等于1e9.


输出描述:
> 假如经过若干次操作可以使得N个数都相等，那么输出"YES", 否则输出"NO"

输入例子:
> 2
> 1 2

输出例子:
> YES

思路:

> 满足YES条件，可知所有数因式分解后，只有2的个数不同.
> 因此一个循环，两个两个处理，用大数除以小数，得到商和余数.
> 如果商不是2的幂，或者余数不等于0，则终止循环，输出NO。

> 证明商是否2的指数幂，可以使用二进制规律，2的指数幂对应的二进制中1的个数为1.

```java
import java.util.*;
 
public class Main{
  public static void main(String[] args){
    Scanner scanner = new Scanner(System.in);
    int n = scanner.nextInt();
    int[] a = new int[n];
    for(int i=0;i<n;i++){
      a[i] = scanner.nextInt();
    }
    for(int i=1;i<n;i++){
      int small, big;
      if(a[i]>a[i-1]){
        small = a[i-1];
        big = a[i];
      }else{
        small = a[i];
        big = a[i-1];
      }
      int remainder = big % small;
      int quotient = big / small;
      if(remainder!=0||(quotient&(quotient-1))!=0)
      {
        System.out.println("NO");
        return;
      }
    }
    System.out.println("YES");
  }
}
```

## 第K大元素

描述:

> 找出数组中第k大元素

输入:

> k: 整数（1<=k<=n）， n: 整数，数组长度， nums: 长度为n的数组

输出:

> 第k大元素

例子:

> 输入: 3 5 
> 4 2 1 5 3

> 输出: 3

思路: 

> 解法一: 利用小顶堆构造k堆，最后插入调整后，nums[0]即为答案，复杂度： nlogk
> 解法二: quickselect，利用快速排序的分区思想，复杂度：最好n， 最差n平方
> quickselect wiki: [https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%80%9F%E9%80%89%E6%8B%A9](https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%80%9F%E9%80%89%E6%8B%A9)

```java
/*
 * 解法一
 */

import java.util.*;

public class Main {

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int k = scanner.nextInt();
    int n = scanner.nextInt();
    int[] nums = new int[n];
    for (int i = 0; i < n; i++) {
      nums[i] = scanner.nextInt();
    }
    for (int i = k / 2 - 1; i >= 0; i--) {
      adjust(nums, i, n);
    }
    for (int i = k; i < n; i++) {
      if (nums[i] > nums[0]) {
        nums[0] = nums[i];
        adjust(nums, 0, k);
      }
    }
    System.out.println(nums[0]);
  }

  private static void adjust(int[] nums, int parent, int length) {
    for (int child = 2 * parent + 1; child < length; child = 2 * parent + 1) {
      if (child + 1 < length && nums[child + 1] < nums[child]) {
        child++;
      }
      if (nums[parent] > nums[child]) {
        int temp = nums[parent];
        nums[parent] = nums[child];
        nums[child] = temp;
      } else {
        break;
      }
      parent = child;
    }
  }
}

/*
 * 解法二
 */ 

import java.util.*;

public class Main {

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int k = scanner.nextInt();
    int n = scanner.nextInt();
    int[] nums = new int[n];
    for (int i = 0; i < n; i++) {
      nums[i] = scanner.nextInt();
    }
    int i, low = 0, high = n - 1;
    while ((i = partition(nums, low, high)) != n - k) {
      if(i+k>n) high = i-1;
      else low = i+1;
    }
    System.out.println(nums[i]);
  }

  private static int partition(int[] nums, int low, int high) {
    int x = nums[low];
    while (low < high) {
      while (low < high && nums[high] >= x)
        high--;
      nums[low] = nums[high];
      while (low < high && nums[low] <= x)
        low++;
      nums[high] = nums[low];
    }
    nums[low] = x;
    return low;
  }
}

```

## 连续整数
牛牛的好朋友羊羊在纸上写了n+1个整数，羊羊接着抹除掉了一个整数，给牛牛猜他抹除掉的数字是什么。牛牛知道羊羊写的整数神排序之后是一串连续的正整数，牛牛现在要猜出所有可能是抹除掉的整数。例如：
10 7 12 8 11 那么抹除掉的整数只可能是9
5 6 7 8 那么抹除掉的整数可能是4也可能是9

输入描述:
> 输入包括2行：
> 第一行为整数n(1 <= n <= 50)，即抹除一个数之后剩下的数字个数
> 第二行为n个整数num\[i\] (1 <= num[i] <= 1000000000)

输出描述:
> 在一行中输出所有可能是抹除掉的数,从小到大输出,用空格分割,行末无空格。如果没有可能的数，则输出mistake

输入例子:
> 2
> 3 6

输出例子:
> mistake

解题思路:
> 找出最大值，最小值，求等差数列和，判断得出结果

```java
import java.util.*;

public class Main{
  public static void main(String[] args){
    Scanner scanner = new Scanner(System.in);
    int n = scanner.nextInt();
    int[] a = new int[n];
    int s = 0, min = Integer.MAX_VALUE, max = Integer.MIN_VALUE;
    for(int i=0;i<n;i++){
      a[i] = scanner.nextInt();
      s += a[i];
      if(a[i]>max) max = a[i];
      if(a[i]<min) min = a[i];
    }
    if(max-min+1==n){
      if(s==(max+min)*n/2){
        if(min>1){
          System.out.println((min-1)+" "+(max+1));
        }else{
          System.out.println(max+1);
        }
      }else{
        System.out.println("mistake");
      }
    }else{
      int m = (max+min)*(n+1)/2-s;
      if(max-min!=n||m<=min||m>=max){
        System.out.println("mistake");
      }else{
        System.out.println(m);
      }
    }
  }
}
```

## 序列和

给出一个正整数N和长度L，找出一段长度大于等于L的连续非负整数，他们的和恰好为N。答案可能有多个，我我们需要找出长度最小的那个。
例如 N = 18 L = 2：
5 + 6 + 7 = 18 
3 + 4 + 5 + 6 = 18
都是满足要求的，但是我们输出更短的 5 6 7

输入描述:
> 输入数据包括一行：
> 两个正整数N(1 ≤ N ≤ 1000000000),L(2 ≤ L ≤ 100)


输出描述:
> 从小到大输出这段连续非负整数，以空格分隔，行末无空格。如果没有这样的序列或者找出的序列长度大于100，则输出No

输入例子:
> 18 2

输出例子:
> 5 6 7

解题思路:
> 假设已知Sn和n，要求a1，由等差数列求和公式 Sn = (a1+an)\*n/2 得： a1 = (2\*Sn/n-(n-1)\*d)/2
> 已知Sn等于N，L<=n<=100，d=1，只要满足条件即有解 a1 = (2\*Sn-n\*n+n)/(2\*n);

代码:

```java
import java.util.*;

public class Main{
  public static void main(String[] args){
    Scanner scanner = new Scanner(System.in);
    int N = scanner.nextInt();
    int L = scanner.nextInt();
    for(int i=L;i<=100;i++){
      if((2*N-i*i+i)%(2*i)==0){
        int a = (2*N-i*i+i)/(2*i);
        while(i-->0){
          System.out.print(a++);
          if(i>0) System.out.print(" ");
        }
        return;
      }
    }
    System.out.println('No');
  }
}
```

## 循环单词

如果一个单词通过循环右移获得的单词，我们称这些单词都为一种循环单词。 例如：picture 和 turepic 就是属于同一种循环单词。 现在给出n个单词，需要统计这个n个单词中有多少种循环单词。 

输入描述:
> 输入包括n+1行：
> 第一行为单词个数n(1 ≤ n ≤ 50)
> 接下来的n行，每行一个单词word[i]，长度length(1 ≤ length ≤ 50)。由小写字母构成

输出描述:
> 输出循环单词的种数

输入例子:
> 5
> picture
> turepic
> icturep
> word
> ordw

输出例子:
> 2

解题思路：

> 旋转字符串，单词拼接自身，查找并标记同类单词。

代码：

```java
import java.util.*;

public class Main{
  public static void main(String[] args){
    Scanner scanner = new Scanner(System.in);
    int n = scanner.nextInt();
    String[] a = new String[n];
    boolean[] b = new boolean[n];
    for(int i=0;i<n;i++){
      a[i] = scanner.next();
    }
    Arrays.fill(b, false);
    int c = 0;
    for(int i=0;i<n;i++){
      if(b[i]) continue;
      c++;
      String s = a[i]+a[i];
      for(int j=i+1;j<n;j++){
        if(a[i].length()!=a[j].length()||!s.contains(a[j]))
          continue;
        b[j] = true;
      }
    }
    System.out.println(c);
  }
}
```
