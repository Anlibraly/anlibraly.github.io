---
layout: post
title: 全排列的递归与非递归实现
description: "全排列是一个十分基础的概念，掌握起来也很简单"
tags: [算法]
image:
  background: witewall_3.png
comments: true
share: true
---

		全排列是一个十分基础的概念，是一串有可比权值的元素出现的所有排列形式，例如 '张全蛋'、'张蛋全'、'全张蛋'、'全蛋
	张'、'蛋全张'、'蛋张全'就是张全蛋的全排列，所以我们发现全排列用来取名字是很不错的，如果对每个汉字在名字中的权值做一
	张表，再来一张可能出现的不同字同时出现在名字中的关联权值表，那么全排列可以算出一张取名字的优先表，暂不表。
		那么全排列基本的一个规律我们可以很明显的发现，一个元素集合的全排列可以由该集合U内任意元素a加上剔除该元素后的集合
	u的全排列组成，而集合u也可以使用该方式去求全排，而最后不可细分的集合就是一个元素。这个描述很适合由递归来做，拿'张全
	蛋'来说，它的全排列由:
		'张'+'全蛋'的全排       '全'+'张蛋'的全排       '蛋'+'张全'的全排
		简而言之 由集合U中每个元素与U的第一个元素交换位置后，剩余的元素集合u递归去做该操作，直到u中只剩1个元素时(即递归
	到集合最后一个元素时)输出一个排列，最终得到集合U全排列。

<!--more-->
	
<pre><code>
//求集合U的全排递归实现 
//all-s的值即为剩余元素集合u的大小
//s即为u集合的起点位置，起始位置为0
void allSorts(char U[],int s,int all){
	if(s == all){
		printf("%s\n",U);	
	}else{
		for(int i=s;i < all;i++){
			swaps(U,i,all-1);
			allSorts(U,s+1,all);
			swaps(U,i,all-1); //换回去等待下一个循环(i+1)与末尾换
		}
	}
}
void swaps(char s[],int i,int j){
	char t = s[i];
	s[i] = s[j];
	s[j] = t;
}
</code></pre>

		第二种方式是用非递归的方法去操作，对于'1234'来说全排列按照字典序'1234'的下一个排列为'1243'，最后一种排
	为'4321'，权值越大的元素在高位的排列应该越靠后，那么如果要靠生成的方法去打印出每一个排列，如果出现权值大的元素
	出现在序列的后部，将其向前放，再将该元素后的元素列翻转可以得到该排列的下一个排列。如求'23154'的后一个排列，从
	右往左第一次出现递减的位置是 5>1 ，那么将1这个位置标记为待替换的位置M，在从右往左遍历第一个大于1的数字为4，那
	么交换1,4后有'23451'，再将位置M后的串翻转有'23415'。按照这种方式可以打印出整个全排列
		
		
<pre><code>
//非递归求U的全排列，l为U中元素个数
void allSort(char U[],int l){
	while(true){
		printf("%s\n",U);
		int i = l-2;
		for(;i>=0;i--){
			if(U[i+1]>U[i]){
				break;
			}else if(i==0){
				return;
			}
		}
		for(int j=l-1;j>i;j--){
			if(U[j]>U[i]){
				swaps(U,i,j);
				resevert(U,i,l-1);
				break;
			}
		}	
	}
//交换i，j位置的
void swaps(char s[],int i,int j){
	char t = s[i];
	s[i] = s[j];
	s[j] = t;
}
//翻转i-j
void resevert(char s[],int i,int j){
	while(i < j){
		swaps(s,i,j);
		i++;
		j--;
	}
}
</code></pre>

		到这里两种全排列的实现都说完了，但是要注意的是在递归实现全排列时如果在元素集合中存在两个相同的元素时，上述递归全
	排列会有重复。比如1223,第一个2与1交换有2123，第二个2与1交换有2213，这时之后的全排为2+123的全排与2+213的全排，明
	显123的全排与213的全排都是重复的。
		非递归的全排列方法并不会出现该问题，因为在判断是否翻转的地方已经规避了相同元素的换位。要规避掉递归当中的重复只要
	**在集合u中元素a与第一个元素交换时判断第一个元素与a之间是否存在a重复的元素，对于有重复元素的不予交换。**当然还要注意
	的地方有，如果想要得到一组完全按照字典序排列的全排列，在递归实现方法中对每一次需要获取全排列的元素集合u做一个递增
	排序，否则如果输入的不是一个最小排列得到的全排列将不是按照字典序的，代码如下
		

<pre><code>
#include < iostream>
#include < cstdio>
#include < cstdlib>
using namespace std;
char aa[100];
//快排 
void qsorts(char a[],int s,int e){
	if(s >= e){
		return;
	}
	char b = a[s];
	int m = s,ss=s,ee=e;
	while(s<e){
		for(;e>=s;e--){
			if(a[e]<b){
				a[m] = a[e];
				m = e;
				break;
			}
		}
		for(;s<=e;s++){
			if(a[s]>b){
				a[m] = a[s];
				m = s;
				break;
			}
		}
	}
	a[m] = b;
	qsorts(a,ss,m-1);
	qsorts(a,m+1,ee);
}
bool isSwaped(char s[],int i,int j){
	for(int k=i;k < j;k++){
		if(s[j] == s[k]){
			return false;	
		}
	}
	return true;
}
void swaps(char s[],int i,int j){
	char t = s[i];
	s[i] = s[j];
	s[j] = t;
}
//递归全排列 
void quanpai1(char s[],int st,int ed){
	if(st == ed){
		printf("%s\n",s);
		return;
	}else{
		qsorts(s,st,ed);
		for(int i=st;i < = ed;i++){
			if(isSwaped(s,st,i)){
				swaps(s,i,st);
				quanpai1(s,st+1,ed);
				swaps(s,i,st);
			}
		}	
	}
}
int main(){
	while(~scanf("%s",aa)){
		int l = strlen(aa);
		quanpai1(aa,0,l-1);
	}
}
</code></pre>

	除了取名字的脑洞之外，全排列貌似用来做彩票开奖预测是比较多的~~~~~~
