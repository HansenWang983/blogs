---
title: "Unity 3D Lecture Note 9"
date: 2018-06-19T11:38:52+08:00
lastmod: 2018-06-19T11:41:52+08:00
menu: "main"
weight: 50
author: "hansenbeast"
tags: [
    "Unity"
]
categories: [
    "Learning Notes"
]
# you can close something for this content if you open it in config.toml.
comment: false
mathjax: false
---

# 第十章、游戏智能

## 作业与练习

Priests and Devils 过河游戏智能帮助实现，程序具体要求：

- 实现状态图的自动生成
- 讲解图数据在程序中的表示方法
- 利用算法实现下一步的计算
- 参考：[P&D 过河游戏智能帮助实现](https://blog.csdn.net/kiloveyousmile/article/details/71727667)

### 博客地址：

https://hansenbeast.github.io/post/2018-06-19-unity3d-lecture-9/

### 效果视频地址：

https://pan.baidu.com/s/1--Ql7bwLUCtVjY8m_go9uQ

### 效果图：

![1](Assets/1.png)

在之前Priest and Devil V2的基础上设计。

### 设计思路：

> 参考博客：https://blog.csdn.net/Kiloveyousmile/article/details/71727667

基于当前游戏的状态图寻找下一步的状态图。

![state](Assets/state.jpeg)

该状态图只记录了游戏过程中左岸的情况。P代表牧师，D代表魔鬼，B代表船。当船在右岸时不记录。双箭头代表两个状态可以相互转化。 



将游戏状态抽象成AIState类，保存左右岸的牧师和魔鬼数量，船的位置和上一步的游戏状态

```c#
// AIState.cs
public class AIState {
	//左右岸的数量
    private int leftPriests;
    private int leftDevils;
    private int rightPriests;
    private int rightDevils;
    //true为左岸，false为右岸
    private bool location;
    //上一步的状态
    private AIState last;
}
```

判断状态是否合法，即是否会导致游戏结束或玩家胜利。

```c#
//判断状态是否合法
public bool isValid() {
	return ((this.leftPriests == 0 || this.leftPriests >= this.leftDevils) &&
	(this.rightPriests == 0 || this.rightPriests >= this.rightDevils));
}
```



在UserGUI中添加场记，维护当前游戏状态和最终获胜的游戏状态。

用户在每次点击船后更新当前游戏状态。

```c#
//UserGUI.cs
public static AIState StartState = new AIState(0, 0, 3, 3, false, null);
public static AIState endState = new AIState(3, 3, 0, 0, true, null);
private string hint = "";
public static FirstController sceneController;
...
public void OnMouseDown() {
		Debug.Log ("OnMouseDown");
		if (status != 1 && status != 2) {
			if (gameObject.name == "boat") {
				action.moveBoat ();
			 	//获得当前信息
			 	int rightPriests = sceneController.leftCoast.getCharacterNum()[0];
				int rightDevils = sceneController.leftCoast.getCharacterNum()[1];
				int leftPriests = sceneController.rightCoast.getCharacterNum()[0];
				int leftDevils = sceneController.rightCoast.getCharacterNum()[1];
				bool direction = sceneController.boat.get_to_or_from () == 1 ? false : true;
				int priest_count = sceneController.boat.getCharacterNum()[0];
				int devil_count = sceneController.boat.getCharacterNum()[1];
				//true为船在左岸，左岸角色的数量要加上船上的角色数量；否则在右岸
				if (direction) {
					leftPriests += priest_count;
					leftDevils += devil_count;
				} else {
					rightPriests += priest_count;
					rightDevils += devil_count;
				}
				//更新当前状态
				StartState = new AIState(leftPriests, leftDevils, rightPriests, rightDevils, direction , null);
			} else {
				action.characterIsClicked (characterController);
            }
		}
	}
```



每个游戏状态的下一步所有可能的情况

AIState.cs，temp为当前游戏状态，next为下一步的合法游戏状态

![source1](Assets/source1.png)

![source2](Assets/source2.png)

![source3](Assets/source3.png)

![source4](Assets/source4.png)



### 算法分析：

博客中使用了随机选择下一步合法游戏状态的方法，而我维护一个queue负责将当前游戏状态的所有合法的下一步状态入队，利用广度优先搜索，直到找到最终游戏状态停止，并回溯寻找其根节点即位下一步的状态。



注意判断结束条件，即回溯最终状态节点。

```c#
while (queue.Count > 0) {
    temp = queue.Peek();

    if (temp == end) {
        while (temp.last != start) {
            temp = temp.last;
        }
        return temp;
    }
    ...
}
```





在UserGUI中添加Hint按钮负责显示或者隐藏提示，添加一个Label负责输出提示内容。注意游戏结束也要更新当前游戏状态为初始状态，因为restart函数中未实现更新。

```c#
//UserGUI.cs
GUI.Label(new Rect(Screen.width / 2 - 50, Screen.height / 2 - 250, 100, 50),hint, description);

		if (GUI.Button(new Rect(Screen.width / 2 - 50, Screen.height / 2 - 90, 100, 50), "Hint", buttonStyle)) {
			if(hint!=""){
				flag = true;
			}
			// Debug.Log("StateRight: " + StartState.rightDevils + " " + StartState.rightPriests);
			// Debug.Log("StateLeft: " + StartState.leftDevils + " " + StartState.leftPriests);
			AIState temp = AIState.BFS(StartState, endState);
			// Debug.Log("NextRight: " + temp.rightDevils + " " + temp.rightPriests);
			// Debug.Log("NextLeft: " + temp.leftDevils + " " + temp.leftPriests);
			hint = "Hint:\n" + 
			"You should make the next state is like :\n"+
			"In Left Coast: "+temp.getLD()+"   Devils and  "+temp.getLP()+"   Priests:\n"+
			"In Right Coast: "+temp.getRD()+" Devils and "+temp.getRP()+"   Priests\n";
			if(flag){
				hint = "";
				flag = false;
			}
		}
```

