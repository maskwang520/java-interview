![ Nth Digit](nth_digit.png)
```java
public static int findNthDigit(int n) {

        int len = 1;
        double sum = 0;
        while (true) {
            sum += Math.pow(10, len - 1) * 9 * len;
            if (sum >= n) {
                sum -= Math.pow(10, len - 1) * 9 * len;
                break;
            }
            len++;
        }
        //先确定哪个数
        int num = 0;
        //分是否整除
        if ((n - sum) % len == 0) {
            num = (int) ((n - sum) / len + Math.pow(10, len - 1) - 1);
        } else {
            num = (int) ((n - sum) / len + Math.pow(10, len - 1));
        }

        int digit = (int) ((n - sum) % len);
        //分是否整除，如果是整除，则是最后一位，否则-1
        if (digit == 0) {
            digit = len -1;
        } else {
            digit--;
        }
        //用string来表达的位操作
        String str = String.valueOf((num)).charAt(digit) + "";

        return Integer.valueOf(str);
    }
```