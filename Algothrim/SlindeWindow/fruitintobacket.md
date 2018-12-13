!(fruitintobucket)[./fruitintobucket.png]
这是一道非常经典的滑动窗口题目,题目转化成这个意思：求一个序列里面，只包含两种字符的最长子序列
```java
public static int totalFruit(int[] tree) {
        Map<Integer, Integer> count = new HashMap<Integer, Integer>();
        int res = 0, i = 0;
        for (int j = 0; j < tree.length; ++j) {
            count.put(tree[j], count.getOrDefault(tree[j], 0) + 1);
            //维持窗口的大小为2
            while (count.size() > 2) {
                count.put(tree[i], count.get(tree[i]) - 1);
                if (count.get(tree[i]) == 0) count.remove(tree[i]);
                i++;
            }
            //记录最大程度
            res = Math.max(res, j - i + 1);
        }
        return res;
    }
```