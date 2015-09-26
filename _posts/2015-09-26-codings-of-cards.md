---
layout: post
title: 银行卡身份证编码及其验证
category: 设计
tags: [php, 编码]
---

最近做用户身份验证，涉及到身份证、银行卡，发现这两个都有自己的编码规则，于是记录下来。

###银行卡

我们常见的银行卡由16位纯数字组成，如中国红十字会捐款账户 6215582201003818703。那么这一串数字由哪些部分组成呢？

1、IIN(Issuer identifier number)

前面6位表示发行者身份标识，由国际卡组织分配，其中首位是4的是VISA的卡，5是Master, 622126~622925是银联标准卡。发行者不仅仅指银行，还包含其他一些商业机构。

有时IIN也被指 bank identification number (bin)。这六位中的第一位又用来唯一标识卡发行者的主要行业。以下是行业对应的编码：

| MII digit | value	Issuer category  |
| --------- | ---------------------- |
| 0	        | ISO/TC 68 and other industry assignments |
| 1	        | Airlines |
| 2	        | Airlines, financial and other future industry assignments |
| 3	        | Travel and entertainment |
| 4	        | Banking and financial |
| 5	        | Banking and financial |
| 6	        | Merchandising and banking/financial |
| 7	        | Petroleum and other future industry assignments |
| 8	        | Healthcare, telecommunications and other future industry assignments |
| 9	        | For assignment by national standards bodies |


2、个人账户标识

从第二位开始至倒数第二位，有发卡行自定义，每个银行的个人账号标识规则有所不同。有的银行在这部分内容中会包含分行、支行、储蓄网点等代码信息。

3、校验位

最后一位为银行卡校验位，用来验证前面的一串数字是否为合法的卡号。校验位是如何生成的呢？

#####Luhm 校验算法

1，将未带校验位的 15 位卡号从右依次编号 1 到 15，位于奇数位号上的数字乘以 2, 第 16 位是校验码

2，将奇位乘积的个十位全部相加，再加上所有偶数位上的数字

3，将加法和加上校验位能被 10 整除。

例如银行卡6215582201003818703，出去最后一位校验位3，其计算规则是基数位乘以2：

    6    2   1   5    5   8   2   2   0  1   0   0  3   8    1   8   7  0
-------------------------------------------------------------------------    
    6    4   1   10   5  16   2   4   0  2   0   0  3   16   1   16  7  0

结果相加：

6 + (4) + 1 + (1 + 0) + 5 + (1 + 6) + 2 + (4) + 0 + (2) + 0 + (0) + 3 + (1 + 6) + 1 + (1 + 6) + 7 + (0) = 57

(57 + 3) % 10 == 0

php程序实现：

	  public function bankcard_check($bankcard) {
        $bankcard = str_replace(' ', '', trim($bankcard));
        if (!$bankcard || !preg_match('/^\d+$/', $bankcard) || strlen($bankcard) < 5) {
            return FALSE;
        }
        $checkcode = substr($bankcard, -1);
        $bankcard = substr($bankcard, 0, -1);
        $len = strlen($bankcard);
        $luhm_sum = 0;
        $j = 1;
        for ($i = $len - 1; $i >= 0; $i--)  {
            $k = intval($bankcard[$i]);
            if ($j % 2 == 1) {
                $k *= 2;
                $k = intval($k / 10) + $k % 10;
            }

            $luhm_sum += $k;
            $j++;
        }
        if (($luhm_sum + $checkcode) % 10 == 0) {
            return $bankcard . $checkcode;
        }
        return FALSE;
    }

#####附

[银行列表](/assets/files/banks.php.txt)，不是最新的，但包含大多数银行，可参考。



###身份证

中国公民第二代身份证由18位数字字符组成，分成四个部分：

1、地址码

表示编码对象常住户口所在县（市、旗、区）的行政区划代码，省市区都是两位

省编码：

	$city = array(
		11 => "北京",
		12 => "天津",
		13 => "河北",
		14 => "山西",
		15 => "内蒙古",
		21 => "辽宁",
		22 => "吉林",
		23 => "黑龙江",
		31 => "上海",
		32 => "江苏",
		33 => "浙江",
		34 => "安徽",
		35 => "福建",
		36 => "江西",
		37 => "山东",
		41 => "河南",
		42 => "湖北",
		43 => "湖南",
		44 => "广东",
		45 => "广西",
		46 => "海南",
		50 => "重庆",
		51 => "四川",
		52 => "贵州",
		53 => "云南",
		54 => "西藏",
		61 => "陕西",
		62 => "甘肃",
		63 => "青海",
		64 => "宁夏",
		65 => "新疆",
		71 => "台湾",
		81 => "香港",
		82 => "澳门",
		91 => "国外"
	);

2、出生日期码

表示编码对象出生的年、月、日，年（YYYY)，月(mm), 日(dd)

3、顺序码

表示在同一地址码所标识的区域范围，对同年、同月、同日出生的人编定的顺序号，顺序码奇数分配给男性，偶数分配给女性。


4、校验码

生成规则遵循 [iso7064](https://en.wikipedia.org/wiki/ISO_7064) mod11-2

十七位数字本体码加权求和公式：

S= SUM(Ai * Wi), i=0, ... , 16

Ai：表示第i位置上的身份证号码数字值

Wi：表示第i位置上的加权因子，值为：[7 9 10 5 8 4 2 1 6 3 7 9 10 5 8 4 2]

S对11取模，得到一个校验码索引，根据索引可得到校验码的值：[1 0 X 9 8 7 6 5 4 3 2]

php代码校验：

	function id_number_check($number) {
		if (!preg_match('/^(\d{6})(19|20)?(\d{2})([01]\d)([0123]\d)(\d{3})(\d|x)?$/i', $number)) {
			return FALSE;
		}

		// $city 变量上面已定义
		if (empty($city[substr($number, 0, 2)])) {
			return FALSE;
		}
		// 加权因子
		$factor = [7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2];
		// 校验位
		$parity = [1, 0, 'X', 9, 8, 7, 6, 5, 4, 3, 2];
		$sum = 0;
		$ai = 0;
		$wi = 0;
		for ($i = 0; $i < 17; $i++) {
			$ai = $number[$i];
			$wi = $factor[$i];
			$sum += $ai * $wi;
		}
		$last = $parity[$sum % 11];
		if ($last != $number[17]) {
			return FALSE;
		}
		return TRUE;
	}



###参考

1、[ISO/IEC_7812](https://en.wikipedia.org/wiki/ISO/IEC_7812)

2、[银行卡编码规则](http://blog.sina.com.cn/s/blog_12fc3a84d0101u7o8.html)

3、[GB 11643-1999](http://vdisk.weibo.com/s/tFDQbBuhBAKN?from=page_100505_profile&wvr=6)


