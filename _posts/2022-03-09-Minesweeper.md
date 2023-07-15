---
layout: post
title: 挖雷
tags:
- dfs
- bfs
categories: leetcode
description: 挖雷
---
让我们一起来玩扫雷游戏！

给你一个大小为 `m x n` 二维字符矩阵 `board` ，表示扫雷游戏的盘面，其中：

'M' 代表一个 未挖出的 地雷，

'E' 代表一个 未挖出的 空方块，

'B' 代表没有相邻（上，下，左，右，和所有4个对角线）地雷的 已挖出的 空白方块，

数字（'1' 到 '8'）表示有多少地雷与这块 已挖出的 方块相邻，

'X' 则表示一个 已挖出的 地雷。

给你一个整数数组 click ，其中 click = [clickr, clickc] 表示在所有 未挖出的 方块（'M' 或者 'E'）中的下一个点击位置（clickr 是行下标，clickc 是列下标）。

根据以下规则，返回相应位置被点击后对应的盘面：

如果一个地雷（'M'）被挖出，游戏就结束了- 把它改为 'X' 。

如果一个 没有相邻地雷 的空方块（'E'）被挖出，修改它为（'B'），并且所有和其相邻的 未挖出 方块都应该被递归地揭露。

如果一个 至少与一个地雷相邻 的空方块（'E'）被挖出，修改它为数字（'1' 到 '8' ），表示相邻地雷的数量。

如果在此次点击中，若无更多方块可被揭露，则返回盘面。

<img src="/assets/img/mine1.jpg" width="500"/>
<img src="/assets/img/mine2.jpg" width="500"/>


提示：

- m == board.length
- n == board[i].length
- 1 <= m, n <= 50
- board[i][j] 为 'M'、'E'、'B' 或数字 '1' 到 '8' 中的一个
- click.length == 2
- 0 <= clickr < m
- 0 <= clickc < n
- `board[clickr][clickc] 为 'M' 或 'E'`

## 分析
1. 深度优先搜索
   1. 如果扫到雷，标记结束。
   2. 如果未扫到雷。
      1. 计算当前坐标x,y周围雷的个数mines。
      2. 如果mines为0，则当前坐标标记为'B'。对于周围标识为`E`的节点坐标做递归标记。(`重要`)
      3. 如果mines为1-9, 标记，`不再扩展`。

2. 广度优先搜索
   1. 方法同深度优先遍历类似，只不过需要一个队列存储待处理的节点坐标。
   2. 当前处理节点周围无雷且周边节点为E时，周边节点才入队列。
   3. 需要一个辅助二维数组，标记已入过队列的节点坐标，防止重复进入队列。


## 代码

### 深度优先搜索
重点是不必要的坐标不再扩展。

```java
    class Solution {
        private int[][] directions = new int[][]{
            {1,0}, {1,-1}, {0,-1}, {-1,-1}, {-1,0}, {-1,1},{0,1},{1,1}
            };
        private int M ;
        private int N;
        public char[][] updateBoard(char[][] board, int[] click) {
            M = board.length;
            N = board[0].length;
            int row = click[0];
            int col = click[1];
            if(board[row][col] == 'M'){
                board[row][col] = 'X';
                return board;
            }
            dfs(board,  row,  col);
            return board;
        }

        public void dfs(char[][] board, int row, int col){
            if(col >= N || col < 0 || row >= M || row < 0){
                    return ;
            }
            if(board[row][col] > '0' && board[row][col] < '9'){
                return;
            }

            if(board[row][col] == 'E'){
                int mines = getRelativeMines(board, row, col);
                if(mines == 0){
                    board[row][col] = 'B';
                    for(int[] direction : directions){
                        int newCol = col + direction[0];
                        int newRow = row + direction[1];
                        // 当周围是'E'，未挖出的方块时，才会去扩展
                        // 重点重点，不必要的坐标不再扩展
                        if(newCol >= N || newCol < 0 || newRow >= M 
                        || newRow < 0 || board[newRow][newCol] != 'E'){
                            continue;
                        }
                        dfs(board, row + direction[1], col + direction[0]);
                    }
                }else if(mines >0 && mines < 9){
                    board[row][col] = (char)('0' + mines);
                }
            }
            
        }

        public int getRelativeMines(char[][] board, int row, int col){
            int relativeMines = 0;
            for(int[] direction : directions){
                int newCol = col + direction[0];
                int newRow = row + direction[1];
                if(newCol >= N || newCol < 0 || newRow >= M || newRow < 0){
                    continue;
                }
                if(board[newRow][newCol] == 'M'){
                    relativeMines +=1;
                }
            }
            return relativeMines;
        }
        
    }
```

***复杂度分析***

- 时间复杂度:O(nm)
- 空间复杂度:O(nm), 空间复杂度取决于递归的`栈深度`。
  
### 广度优先搜索
重点是`visit`数组标记已处理过的`E`节点。

```java
class Solution {
    private int[][] directions = new int[][]{
        {1,0}, {1,-1}, {0,-1}, {-1,-1}, {-1,0}, {-1,1},{0,1},{1,1}
        };
    private int M ;
    private int N;
    public char[][] updateBoard(char[][] board, int[] click) {
        M = board.length;
        N = board[0].length;
        int row = click[0];
        int col = click[1];
        if(board[row][col] == 'M'){
            board[row][col] = 'X';
            return board;
        }
        bfs(board,  row,  col);
        return board;
    }
    // 广度优先搜索
    private void bfs(char[][] board, int r, int c){
        LinkedList<int[]> list = new LinkedList<int[]>();
        list.offer(new int[]{r, c});
        boolean visit[][] = new boolean[M][N];
        // 重点
        visit[r][c] = true;
        while(!list.isEmpty()){
            int[] e = list.poll();
            int row = e[0];
            int col = e[1];
            int mines = getRelativeMines(board, row, col);
            if(mines == 0){
                board[row][col] = 'B';
                for(int[] direction : directions){
                    int newCol = col + direction[0];
                    int newRow = row + direction[1];
                    // 当周围是'E'，未挖出的方块时，才会去扩展
                    if(newCol >= N || newCol < 0 || newRow >= M 
                    || newRow < 0 || board[newRow][newCol] != 'E'
                    || visit[newRow][newCol]){
                        continue;
                    }
                    list.offer(new int[]{newRow, newCol});
                    visit[newRow][newCol] = true;
                }
            }else if(mines > 0 && mines < 9){
                board[row][col] = (char)('0' + mines);
            }
        }
    }

    public int getRelativeMines(char[][] board, int row, int col){
        int relativeMines = 0;
        for(int[] direction : directions){
            int newCol = col + direction[0];
            int newRow = row + direction[1];
            if(newCol >= N || newCol < 0 || newRow >= M || newRow < 0){
                continue;
            }
            if(board[newRow][newCol] == 'M'){
                relativeMines +=1;
            }
        }
        return relativeMines;
    }
}
```

***复杂度分析***

- 时间复杂度:O(nm)
- 空间复杂度:O(nm), visit标记数组所需要。

## 参考
- [扫雷游戏](https://leetcode.cn/problems/minesweeper/)

