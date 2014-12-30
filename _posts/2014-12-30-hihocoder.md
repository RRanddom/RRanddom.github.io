---
layout: default
title:  Divided Product
---

### 问题描述

Given two positive integers N and M, please divide N into several integers A1, A2, ..., Ak (k >= 1), so that:

* 0<A1<A2<...<Ak;
* A1+A2+...+Ak=N;
* A1,A2,...,Ak are different with each other;
* The Product of them P=A1*A2*...*Ak is a multiple of M;

###我的答案

```cpp

	#include <iostream>

	using namespace std;

	int gcd(int a,int b)
	{
	int t;
	while(b)
	{
		t = b;
		b = a%b;
		a = t;
	}
	return a;
	}

	int search(int i,int j,int res)
	{
		if(i==0)
		{
			if(res==1) return 1;
			else return 0;
		}
	
		int count = 0;
	
		for(int k=j;k>=1;k--)
		{
			if(k*(k+1) < 2*i)
				break;
			else
				count += search(i-k,min(i-k,k),res/gcd(res,k));
			}
	
			return count;
		}

	int main(void){
	
		int N,M;
	
		cin>>N>>M;
	
		cout<<search(N,N,M)<<endl;
	
		return 0;
	}

```
