！[longestSubstring](longestsubstring.png)
```java
 public static int longestSubstring(String s, int k) {
        int len = 0;
        int[] cnts = new int[26];
        for (int i = 1; i <= 26; i++) {
            Arrays.fill(cnts, 0);
            int begin = 0, end = 0, uniqueChar = 0;
            while (end < s.length()) {
                boolean valid = true;
                if (cnts[s.charAt(end++) - 'a']++ == 0) uniqueChar++;

                // need exactly i unique characters
                while (uniqueChar > i)
                    if (cnts[s.charAt(begin++) - 'a']-- == 1) uniqueChar--;

                // if the string has any character with less than k occurrences, the string is invalid
                for (int j = 0; j < 26; j++)
                    if (cnts[j] > 0 && cnts[j] < k) valid = false;

                if (valid) len = Math.max(len, end - begin);
            }
        }
        return len;
    }
```
* 利用滑动窗口，根据有个i（1~26）个不同的字符，滑动窗口，从而得到结果。