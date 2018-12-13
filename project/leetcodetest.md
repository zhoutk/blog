# 简单易用的leetcode开发测试工具（npm）
## 描述

最近在用es6解leetcode，当问题比较复杂时，有可能修正了新的错误，却影响了前面的流程。要用通用的测试工具，却又有杀鸡用牛刀的感觉，所以就写了个简单易用的leetcode开发测试工具，分享与大家。

## 工具安装

npm i leetcode_test

## 使用示例1 (问题010)

codes:

```
let test = require('leetcode_test').test
/**
 * @param {string} s
 * @param {string} p
 * @return {boolean}
 */
var isMatch = function (s, p) {
    if (p.length === 0) {
        return s.length === 0
    }
    firstMath = s.length > 0 && 
                (p[0] === s[0] ||
                p[0] === '.')
    if (p.length >= 2 && p[1] === '*') {
        //下面两部分的顺序不能交换
        return firstMath && isMatch(s.substring(1), p) || isMatch(s, p.substring(2))
    } else {
        return firstMath && isMatch(s.substring(1), p.substring(1))
    }
};
let cases = [              // [[[],''],],   //第一个参数是空数组
    [['abbabaaaaaaacaa', 'a*.*b.a.*c*b*a*c*'], true],
    [['aaa', 'a*ac'], true],                //故意写错答案，展示测试失败输出效果
    [['a', '..*'], true],
]
test(isMatch, cases)
```
> 测试用例编写说明

leetcode要测试的都是函数，参数个数不定，但返回值是一个。因此，我设计用例的输入形式为一个用例就是一个两个元素的数组，第一个元素是一个数组：对应输入参数；第二个元素是一个值。
上面例子的输入参数是（[2, 7, 11, 15], 91），第一个参数是数组，第二个参数是数值；返回值是一个数组（[0, 1]）。 如果要测试的函数的输入参数就是一个数组，要注意输入形式，比如，求[1,2,3,4]平均值，要这样输入测试用例： [[[1,2,3,4]],2.5]

out:

```
test [1] success, Input: ('abbabaaaaaaacaa','a*.*b.a.*c*b*a*c*'); Expected: true; Output: true
test [2] fail, Input: ('aaa','a*ac'); Expected: true; Output: false
test [3] success, Input: ('a','..*'); Expected: true; Output: true
Result: test 3 cases, success: 2, fail: 1
running 5 ms
```
## 使用示例2 (问题015)

codes:

```
let test = require('leetcode_test').test
/**
 * @param {number[]} nums
 * @return {number[][]}
 */
var threeSum = function (nums) {
    nums = nums.sort((a,b) => a - b);
    const rs = [];
    let i = 0;
    while (i < nums.length) {
        let one = nums[i];
        let two = i + 1;                    //从队列头部开始
        let three = nums.length - 1;        //从队列尾部开始

        while (two < three) {
            let sum = one + nums[two] + nums[three];
            if (sum === 0) {
                rs.push([one,nums[two],nums[three]]);
                two++;
                three--;
                while (two < three && nums[two] === nums[two - 1]) {
                    two++;
                }
                while (two < three && nums[three] === nums[three + 1]) {
                    three--;
                }
            } else if (sum > 0) three--;
            else two++;
        }
        i++;
        while (i < nums.length && nums[i] === nums[i - 1]) i++;
    }
    return rs;
};
let cases = [               // [[[],''],],   //第一个参数是空数组
    [[[]],[]],
    [[[1,-1,-1,0]],[-1,0,1]],
    [[[-1,0,1,0]],[[-1,0,1]]],
    [[[0,0,0,0]],[0,0,0]],
    [[[-1,2,-1]],[-1,-1,2]],
    [[[0,0,0]],[0,0,0]],
    [[[-1,0,1,2,-1,-4]],[[-1,-1,2],[-1,0,1]]],            //answer's sequence is not important
    [[[-1,0,1,2,-1,-4]],[[-1,0,1],[-1,-1,2]]],            //answer's sequence is not important
    [[[-4,-2,-2,-2,0,1,2,2,2,3,3,4,4,6,6]],[[-4,-2,6],[-4,0,4],[-4,1,3],[-4,2,2],[-2,-2,4],[-2,0,2]]],
    [[[-4,-2,1,-5,-4,-4,4,-2,0,4,0,-2,3,1,-5,0]],[[-5,1,4],[-4,0,4],[-4,1,3],[-2,-2,4],[-2,1,1],[0,0,0]]],
]
test(threeSum,cases)
```
> 测试用例编写说明

测试用例的7与8，期待结果的数组元素顺序并不影响答案的判定。

out:

```
test [1] success, Input: ([]); Expected: []; Output: []
test [2] success, Input: ([-1,-1,0,1]); Expected: [-1,0,1]; Output: [[-1,0,1]]
test [3] success, Input: ([-1,0,0,1]); Expected: [[-1,0,1]]; Output: [[-1,0,1]]
test [4] success, Input: ([0,0,0,0]); Expected: [0,0,0]; Output: [[0,0,0]]
test [5] success, Input: ([-1,-1,2]); Expected: [-1,-1,2]; Output: [[-1,-1,2]]
test [6] success, Input: ([0,0,0]); Expected: [0,0,0]; Output: [[0,0,0]]
test [7] success, Input: ([-4,-1,-1,0,1,2]); Expected: [[-1,-1,2],[-1,0,1]]; Output: [[-1,-1,2],[-1,0,1]]
test [8] success, Input: ([-4,-1,-1,0,1,2]); Expected: [[-1,-1,2],[-1,0,1]]; Output: [[-1,-1,2],[-1,0,1]]
test [9] success, Input: ([-4,-2,-2,-2,0,1,2,2,2,3,3,4,4,6,6]); Expected: [[-2,-2,4],[-2,0,2],[-4,-2,6],[-4,0,4],[-4,1,3],[-4,2,2]]; Output: [[-2,-2,4],[-2,0,2],[-4,-2,6],[-4,0,4],[-4,1,3],[-4,2,2]]
test [10] success, Input: ([-5,-5,-4,-4,-4,-2,-2,-2,0,0,0,1,1,3,4,4]); Expected: [[-2,-2,4],[-2,1,1],[-4,0,4],[-4,1,3],[-5,1,4],[0,0,0]]; Output: [[-2,-2,4],[-2,1,1],[-4,0,4],[-4,1,3],[-5,1,4],[0,0,0]]
Result: test 10 cases, success: 10, fail: 0
```

## 项目地址

工具地址：https://github.com/zhoutk/leetcode_test
解答地址：https://github.com/zhoutk/leetcode

最近一直在用，已经把输出的样子调得还能看过眼了，答案对比算法，也改进了。遇到问题，我会持续改进，大家遇到问题也可提bug给我，我会尽快处理。
