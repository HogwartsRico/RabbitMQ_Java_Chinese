##先复习一下2Sum问题

####方法一:空间换时间

用hashtable

```java

import java.util.Hashtable;

/**
 * Created by lrkin on 2016/10/28.
 */
public class TwoSumPractest01 {

    public void twoSum(int[] array, int key) {

        Hashtable<Integer, Integer> hashTable
                = new Hashtable<Integer, Integer>();
        for (int i = 0; i < array.length; i++) {
            hashTable.put(array[i], i);
        }
        for (int i = 0; i < array.length; i++) {
            int diff = key - array[i];
            if (hashTable.get(diff) != null) {
                int[] result = {array[i], array[hashTable.get(diff)]};
                for (int j = 0; j < result.length; j++) {
                    System.out.print(result[j] + "-");
                    break;
                }
            }
        }
    }

    public static void main(String[] args) {
        TwoSumPractest01 twoSumPractest01 = new TwoSumPractest01();
        int[] array = {6, 4, 3, 10, 8, 12};
        twoSumPractest01.twoSum(array, 13);
    }

}

```

####方法二: 嵌套遍历

```java

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashSet;
import java.util.List;
∂
/**
 * Created by lrkin on 2016/10/28.
 * <p>
 * 题目:
 * <p>
 * {-2,0,4,4,3,3,9,8,12}
 * <p>
 * key = 7
 * <p>
 * 输出:
 * <p>
 * (-2 , 9)
 * (3,4)
 */
public class TwoSumPractest02 {

    public ArrayList<int[]> twoSum(int[] array, int key) {

        ArrayList<int[]> result = new ArrayList<int[]>();

        for (int i = 0; i < array.length; i++) {
            if ((i + 1) <= array.length - 1 && array[i] == array[i + 1]) {
                continue;
            }
            for (int k = i + 1; k < array.length; k++) {
                if ((k + 1) <= array.length - 1 && array[k] == array[k + 1]) {
                    continue;
                }
                if ((array[i] + array[k]) == key) {
                    int[] arr = {array[i], array[k]};
                    result.add(arr);
                }
            }
        }
        return result;
    }

    public static void main(String[] args) {
        TwoSumPractest02 twoSumPractest02 = new TwoSumPractest02();
        int[] array = {-2, 0, 4, 4, 3, 3, 9, 8, 12};
        int key = 7;
        ArrayList<int[]> result = twoSumPractest02.twoSum(array, key);
        for (int[] item : result) {
            for (int i = 0; i < item.length; i++) {
                System.out.print(item[i] + "-");
            }
            System.out.println("");
        }
    }

}

```

##3Sum

那么同理,3sum就是多了一个枚举的2sum

看代码之前,我们要理清一个很基础的东西...

```for(int i = 0;i < array.length ; i++){
		//do something
}
```

这行代码的意思是:

i=0,如果i<array.length,那么就执行下方的代码,执行结束i+1
如果i<array.length继续成立,就继续执行下方的代码


好了,上3Sum的代码:

```java
import java.util.ArrayList;
import java.util.Arrays;

/**
 * Created by lrkin on 2016/10/23.
 * <p>
 * 题目:
 * <p>
 * Given an array S of n integers, are there elements a, b, c in S such that a + b + c = 0? Find all unique triplets in the array which gives the sum of zero.
 * Note:
 * Elements in a triplet (a,b,c) must be in non-descending order. (ie, a ≤ b ≤ c)
 * The solution set must not contain duplicate triplets.
 * For example, given array S = {-1 ,-1 ,0 ,0 ,1 ,2, -1, -4},
 * <p>
 * A solution set is:
 * (-1, 0, 1)
 * (-1, -1, 2)
 */
public class ThreeSum {

    public ArrayList<int[]> threeSum(int[] array, int key) {
        ArrayList<int[]> result = new ArrayList<>();
        Arrays.sort(array);
        for (int i = 0; i < array.length; i++) {

            if ((i + 2) <= array.length && (i-1) >=0 && array[i] == array[i-1]) {
                continue;
            }

            int diff = key - array[i];
            ArrayList<int[]> oneResult = twoSum(array, i+1, array.length, diff);
            if (oneResult.size() != 0) {
                for (int[] item : oneResult) {
                    result.add(item);
                }
            }
        }
        return result;
    }

    private ArrayList<int[]> twoSum(int[] array, int start, int end, int diff) {
        ArrayList<int[]> result = new ArrayList<>();
        if (start <= end) {
            for (int j = start; j < end; j++) {
                if ((j + 1) <= end - 1 && (j-1) >= 0 && array[j] == array[j - 1]) {
                    continue;
                }
                for (int k = j + 1; k < end; k++) {
                    if ((k + 1) <= end - 1 && (k-1) >=0 && array[k] == array[k + 1]) {
                        continue;
                    }
                    if ((array[j] + array[k]) == diff) {
                        int[] arr = {array[start], array[j], array[k]};
                        result.add(arr);
                    }
                }
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int[] array = {-1, -1, 0, 0, 0, 1, 2, -1, -4};
//        int[] array = {-1 ,-1 ,0 ,0 ,1 ,2, -1, -4};
//        int[] array = {-1 ,0, 1, 2, -1, -4};
        ThreeSum threeSum = new ThreeSum();
        int key = 0;
        ArrayList<int[]> result = threeSum.threeSum(array, key);
        for (int[] item : result) {
            for (int i = 0; i < item.length; i++) {
                System.out.print(item[i] + " ");
            }
            System.out.println("");
        }
    }
}
```

我的这个写法有点问题








