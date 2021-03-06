---
title: tree
date: 2016-05-31 13:44:28
tags: data struct 
---
## 前言
　　好久没有更新我的博客了，最近快开始校招了。因为长期在外面做开发，加上离考研已经很久了。数据结构的知识都已经记得模模糊糊了。为了准备笔试，同时提高自己的计算机素养。自己试着回忆了一下，树结构的基本算法。

## 数据结构
　　二叉树存储解构是一个数据两个指针。换到java中，就是一个变量，两个引用

{%codeblock%}
public static class Node{
		public Node left;
		public Node right;
		public int val;
	}

{%endcodeblock%}

## 二叉树的建立
　二叉树能顺序存储，也能链式存储。但是链式存储更能直观的表现出二叉树的特征。下面这个算法是由顺序存储结构生成链式存储结构。我把没有数据的结点在数组中用"-1"表示（貌似很多教材都用'＃'表示）。
{%codeblock%}
public  Node buildTree(int[] nums,int i) {
		// TODO Auto-generated method stub
		if(i > nums.length -1){
			return null;
		}
		if(nums[i]== -1 ){
			return null;
		}
	
		Node n  = new Node();
		n.val = nums[i];
		n.left = buildTree(nums, 2*i);
		n.right = buildTree(nums, 2*i+1);
		return n;
	}

{%endcodeblock%}
## 二叉树的遍历
　　　二叉树的遍历是考试中最经常考的内容，他的递归遍历代码优雅，简洁。有一种让人过目不忘的感觉。这里就不给出了，但是值得注意的是，一个结点被无论是哪种遍历，在递归的时候，它已经被被经过了三次。上面上一张考研材料上的图，
{% img /images/tree.png%}

标注为１的，是前序遍历
标注为2的，是中序遍历
标注为3的，是后序遍历

还有一点头脑风暴的感觉，就是用栈来实现递归，其实思想也不难，就是有点绕。
总体都是，按照上述路线入栈，前中序当他是最后叶子结点时候，出栈。后序遍历是经过第二次时候才出栈。(原谅我令人可怜的语文水平吧)
下面是我写的非递归前序遍历。
{%codeblock%}
	void preorder2(Node root){
		Stack<Node> s = new Stack<Node>();
        
		while(root !=null || !s.isEmpty()){
			while(root!= null){
				s.push(root);
				System.out.print(root.val + "--->");
			}
			if(!s.isEmpty() && root == null){
				root=s.pop();
				root=root.right;
			}
		}
		
	}
{%endcodeblock%}

层次遍历就更简单了，利用队列实现，当队列不为空，出列，读取他的数据，并将他的左右孩子入队列。有趣的是，层次遍历正好是数的线性存储。默认数组位置　０　不存放。
{%codeblock%}
	void queueOrder(Node root){
		List<Node> l = new ArrayList<Node>();
		l.add(root);
		
		while(l.size() != 0){
			Node n=l.remove(0);
			System.out.print("  " + n.val);
			if(n.left != null) l.add(n.left);
			if(n.right != null) l.add(n.right);
		}
	}
{%endcodeblock%}

## 其他
树是非常有用的数据结构，包括平衡二叉树，二叉查询树，哈夫曼编码都是树有意思的应用。我还没有总结成代码。但是，这些东西挺有趣，包括java中很多容器都利用到树的知识。比如TreeMap ，就是红黑树。

## 练习代码　拿出来献丑了

{%codeblock%}

package com.may.eighteen;

import java.util.ArrayList;
import java.util.List;
import java.util.Stack;

public class Solution {

	public static class Node{
		public Node left;
		public Node right;
		public int val;
	}
	static int[] nums = {-1,1,2,3,4,5,6,7,8,9};
	public static void main(String args[]){
		Solution s = new Solution();
		Node root = s.buildTree(nums,1);
		System.out.print("-----hello----");
		s.preorder(root);
		System.out.println("");
		s.preorder(root);
		System.out.println("");
		s.queueOrder(root);
		System.out.println("");
		System.out.print(s.findNodeCount(root));
		
	}
	
	public  Node buildTree(int[] nums,int i) {
		// TODO Auto-generated method stub
		if(i > nums.length -1){
			return null;
		}
		if(nums[i]== -1 ){
			return null;
		}
	
		Node n  = new Node();
		n.val = nums[i];
		n.left = buildTree(nums, 2*i);
		n.right = buildTree(nums, 2*i+1);
		return n;
	}
	

	
	public void buileTree(int[] preOrder, int[] inOrder, Node root){
		if(root == null) return;
		
		
	}
	
	
	void queueOrder(Node root){
		List<Node> l = new ArrayList<Node>();
		l.add(root);
		
		while(l.size() != 0){
			Node n=l.remove(0);
			System.out.print("  " + n.val);
			if(n.left != null) l.add(n.left);
			if(n.right != null) l.add(n.right);
		}
	}
	
	void preorder(Node root){
		if(root == null) return;
		System.out.print(root.val + "---");
		preorder(root.left);
		preorder(root.right);
	}
	
	void preorder2(Node root){
		Stack<Node> s = new Stack<Node>();
        
		while(root !=null || !s.isEmpty()){
			while(root!= null){
				s.push(root);
				System.out.print(root.val + "--->");
			}
			if(!s.isEmpty() && root == null){
				root=s.pop();
				root=root.right;
			}
		}
		
	}

	
   int findNodeCount(Node root){
		if(root == null){return 0;}
		return findNodeCount(root.left) + findNodeCount(root.right) +1;
	}
	
}

{%endcodeblock%}



