---
layout: post
title: 后缀表达式与中缀表达式相互转换
category: algorithm
tags: [php, 算法]
---

最近做对账平台的动态配置时，有一个解析表达式的需求。在页面输入一个表达式，比如(sysno-outno，100\*money)，系统可根据表达式的类型来决定求其值还是仅仅替换掉变量。刚开始想到用php的eval函数，输入'100\*3'，eval('100\*3')马上就能得到结果，但当输入'sysno-outno'这种需替换变量时就显得很不方便了,还是得解析下表达式。

最初想到了前缀表达式，简单且能快速实现。例如用户配置某个字段的值等于100\*(a+b),则输入

    ["*",[100,["+",["a","b"]]]]

后台先json_decode,然后使用递归马上就能解析到其字符串形式。然而这样方便了自己，但给用户配置带来了不少困难。想到最终的系统应该是简单易用，还是得想想其他的方法。如果使用后缀表达式呢？用户直接输入中缀表达式，后台将其转换成后缀表达式，再重新拼装成所需的字符串形式的中缀表达式。

这个过程主要的难点是怎样把中缀表达式转换成[后缀表达式](http://zh.wikipedia.org/wiki/%E9%80%86%E6%B3%A2%E5%85%B0%E8%A1%A8%E7%A4%BA%E6%B3%95)。以我的理解，后缀表达式即两个操作数在前，操作符在后。转换的过程可以简单的表述如下：<em>初始化两个栈，操作数栈s1和操作符栈s2，从左往右遍历中缀表达式，遇到操作数入栈s1，遇到右括号，或者是其他符号并且比当前s2栈顶符号的优先级低，则将s2栈顶符号弹出并入栈s1，并将当前符号入栈s2（我的理解：符号优先级越高，则其越靠近栈顶，优先参与运算），重复以上操作直到结束。最终的s1元素即为所求后缀表达式</em>。

使用php实现如下：

    /*
     * 后缀表达式与中缀表达式相互转换的一个简单实现
     */

    class Formula
    {
        /** 操作符列表 */
        private static $operator = array('(', ')', '+', '-', '*', '/');
        /** 算法优先级，数字越大则优先级越高 */
        private static $priority = array('#' => 0, '(' => 10, '+' => 20, '-' => 20, '*' => 30, '/' => 30);

        /**
         * @brief 字符串表达式转换成逆波兰式
         * @expression String
         * @return Array
         */
        public static function exp2rpn($expression)
        {
            /** 操作符栈 */
            $operator_stack = array('#');
            /** 操作数栈 */
            $operand_stack = array();
            $len = strlen($expression);
            $word = '';
            for ($i = 0; $i < $len; $i++) {
                $char = substr($expression, $i, 1);
                if (!in_array($char, self::$operator)) {
                    $word .= $char;
                    continue;
                }
                if (!empty($word)) {
                    array_push($operand_stack, $word);
                    $word = '';
                }
                if ($char == '(') {
                    array_push($operator_stack, $char);
                    continue;
                } else if ($char == ')') {
                    while (count($operator_stack)) {
                        $lex = array_pop($operator_stack);
                        if ($lex == '(') {
                            break;
                        } else {
                            array_push($operand_stack, $lex);
                        }
                    }
                    continue;
                } else if (self::$priority[$char] <= self::$priority[end($operator_stack)]) {
                    array_push($operand_stack, array_pop($operator_stack));
                    array_push($operator_stack, $char);
                    continue;
                } else {
                    array_push($operator_stack, $char);
                    continue;
                }
            }
            /* 最后一个操作数入栈 */
            array_push($operand_stack, $word);
            while (count($operator_stack)) {
                if (end($operator_stack) == '#') {
                    break;
                }
                array_push($operand_stack, array_pop($operator_stack));
            }
            return $operand_stack;
        }

        /**
         * @brief 将逆波兰式转换成字符串的表达式
         * @rpn Array 波兰式数组
         * @withC Boolean 输出结果是否带括号
         * @return String
         */
        public static function rpn2exp($rpn, $withC = true)
        {
            if (!is_array($rpn)) {
                return '';
            }
            $stack = array();
            $len = count($rpn);
            for ($i = 0; $i < $len; $i++) {
                if (in_array($rpn[$i], self::$operator)) {
                    $operator = $rpn[$i];
                    $right_operand = array_pop($stack);
                    $left_operand = array_pop($stack);
                    $res = $left_operand . $operator . $right_operand;
                    if ($withC) {
                        $res = '(' . $res . ')';
                    }
                    array_push($stack, $res);
                } else {
                    array_push($stack, $rpn[$i]);
                }
            }
            return current($stack);
        }

        /*
         * @brief 将表达式中指定变量替换成数值
         * @exp String 字符串表达式
         * @fields Array 包含变量的关联数组
         * @withC boolean 输出结果是否带括号
         * @return String
         */
        public static function exp_replace($exp, $fields, $withC = false)
        {
            $rpn = self::exp2rpn($exp);
            $stack = array();
            $len = count($rpn);
            for ($i = 0; $i < $len; $i++) {
                if (in_array($rpn[$i], self::$operator)) {
                    $operator = $rpn[$i];
                    $right_operand = array_pop($stack);
                    $left_operand = array_pop($stack);
                    if (isset($fields[$right_operand])) {
                        $right_operand = $fields[$right_operand];
                    }
                    if (isset($fields[$left_operand])) {
                        $left_operand = $fields[$left_operand];
                    }
                    $res = $left_operand . $operator . $right_operand;
                    if ($withC) {
                        $res = '(' . $res . ')';
                    }
                    array_push($stack, $res);
                } else {
                    array_push($stack, $rpn[$i]);
                }
            }
            return current($stack);
        }
    }

    $exp = '(10*(sysno+outno)-a/b)*c';
    $r = Formula::exp2rpn($exp);
    var_dump($r);
    $a = Formula::rpn2exp($r, true);
    var_dump($a);
    $a = Formula::exp_replace('sysno-outno', array('sysno' => '2323232', 'outno' => '998979'));
    var_dump($a);

###参考
1、[波兰式、逆波兰式与表达式求值](http://zhouliang.pro/2013/08/18/%E9%80%86%E6%B3%A2%E5%85%B0%E5%BC%8F/)

2、[逆波兰表示法](http://zh.wikipedia.org/wiki/%E9%80%86%E6%B3%A2%E5%85%B0%E8%A1%A8%E7%A4%BA%E6%B3%95)





