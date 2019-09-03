---
title: 车牌号码正则表达式
date: 2019-09-03 07:00:00
categories:
- Other
tags:
- 正则表达式
description: 车牌号码正则表达式
---

根据网络上一些资料搜集整理的车牌号码的正则表达式，不一定全

# 民用车牌，挂车，新能源

* 京[A-HJ-NPQY]
* 沪[A-HJ-N]
* 津[A-HJ-NPQR]
* 渝[A-DFGHN]
* 冀[A-HJRST]
* 晋[A-FHJ-M]
* 蒙[A-HJKLM]
* 辽[A-HJ-NP]
* 吉[A-HJK]
* 黑[A-HJ-NPR]
* 苏[A-HJ-N]
* 浙[A-HJKL]
* 皖[A-HJ-NP-S]
* 闽[A-HJK]
* 赣[A-HJKLMS]
* 鲁[A-HJ-NP-SUVWY]
* 豫[A-HJ-NP-SU]
* 鄂[A-HJ-NP-S]
* 湘[A-HJ-NSU]
* 粤[A-HJ-NP-Y]&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Z[0-9A-HJ-NP-Z][0-9]{3}[港澳]`
* 桂[A-HJ-NPR]
* 琼[A-F]
* 川[A-HJ-MQ-Z]
* 贵[A-HJ]
* 云[AC-HJ-NP-SV]
* 藏[A-HJ]
* 陕[A-HJKV]
* 甘[A-HJ-NP]
* 青[A-H]
* 宁[A-E]
* 新[A-HJ-NP-S]

```
([0-9A-HJ-NP-Z]{4}[0-9A-HJ-NP-Z挂试]|[0-9]{4}学|[A-D0-9][0-9]{3}警|[DF][0-9A-HJ-NP-Z][0-9]{4}|[0-9]{5}[DF])
```

# 武警车牌

* WJ[京沪津渝冀晋蒙辽吉黑苏浙皖闽赣鲁豫鄂湘粤桂琼川贵云藏陕甘青宁新]?[0-9]{4}[0-9JBXTHSD]

# 军队车牌

```
(V[A-GKMORTV]|K[A-HJ-NORUZ]|H[A-GLOR]|[BCGJLNS][A-DKMNORVY]|G[JS])[0-9]{5}
```

# 使馆车牌

```
[0-9]{6}使
```

# 领馆车牌

```
([沪粤川渝辽云桂鄂湘陕藏黑]A|闽D|鲁B|蒙[AEH])[0-9]{4}领
```

# 车牌正则表达式

```
^(京[A-HJ-NPQY]|沪[A-HJ-N]|津[A-HJ-NPQR]|渝[A-DFGHN]|冀[A-HJRST]|晋[A-FHJ-M]|蒙[A-HJKLM]|辽[A-HJ-NP]|吉[A-HJK]|黑[A-HJ-NPR]|苏[A-HJ-N]|浙[A-HJKL]|皖[A-HJ-NP-S]|闽[A-HJK]|赣[A-HJKLMS]|鲁[A-HJ-NP-SUVWY]|豫[A-HJ-NP-SU]|鄂[A-HJ-NP-S]|湘[A-HJ-NSU]|粤[A-HJ-NP-Y]|桂[A-HJ-NPR]|琼[A-F]|川[A-HJ-MQ-Z]|贵[A-HJ]|云[AC-HJ-NP-SV]|藏[A-HJ]|陕[A-HJKV]|甘[A-HJ-NP]|青[A-H]|宁[A-E]|新[A-HJ-NP-S])([0-9A-HJ-NP-Z]{4}[0-9A-HJ-NP-Z挂试]|[0-9]{4}学|[A-D0-9][0-9]{3}警|[DF][0-9A-HJ-NP-Z][0-9]{4}|[0-9]{5}[DF])$|^WJ[京沪津渝冀晋蒙辽吉黑苏浙皖闽赣鲁豫鄂湘粤桂琼川贵云藏陕甘青宁新]?[0-9]{4}[0-9JBXTHSD]$|^(V[A-GKMORTV]|K[A-HJ-NORUZ]|H[A-GLOR]|[BCGJLNS][A-DKMNORVY]|G[JS])[0-9]{5}$|^[0-9]{6}使$|^([沪粤川渝辽云桂鄂湘陕藏黑]A|闽D|鲁B|蒙[AEH])[0-9]{4}领$|^粤Z[0-9A-HJ-NP-Z][0-9]{3}[港澳]$
```

 


 



