---
title: 项目中npm依赖问题
date: 2018-06-11
tags: [npm,依赖,JavaScript]
author: gcyStar
---

最近开发遇到一个npm依赖导致的问题，报错如下。
![slot-error](https://img.wuage.com/public/152869941467897690642_1076_206.png)
经过错误定位，发现原因是项目中用了vue不同版本语法的写法，如下所示 ，重点关注的是被**包围的写法。

```
 <el-table-column
          label="成交单数"
          align="center"
          min-width="100">
        **  <template scope="scope">**
            <a :href="scope.row.quoteInfo.trdCntUrl" v-if="scope.row.quoteInfo.trd_cnt > 0" target="_blank">{{scope.row.quoteInfo.trd_cnt}}</a>
            <span v-else>{{scope.row.quoteInfo.trd_cnt}}</span>
          </template>
        </el-table-column>

        <el-table-column
          label="主营产品"
          class-name="item nb"
          align="center"
          min-width="100">
       **   <template slot-scope="scope">**
            <el-table
              :data="scope.row.preferenceList"
              size="100%"
              :show-header=false>
              <el-table-column
                class-name="qqqqqq"
                prop="mainIndustry"
                show-overflow-tooltip>
              </el-table-column>
            </el-table>
          </template>
        </el-table-column>
```
之前的开发人员用了两种方法指定插槽（slot）的作用域，查看了下官方的更新记录

In [2.5](https://gist.github.com/yyx990803/9bdff05e5468a60ced06c29c39114c6b#simplified-scoped-slots-usage), the scope attribute has been deprecated (it still works, but you will get a soft warning). Instead, we now use slot-scope to denote a scoped slot, and it can be used on a normal element/component in addition to <template>


意思就是说在2.5以后,把scope换成了slop-scope,而我们项目中﻿package.json中的vue版本

```
﻿"vue": "^2.3.3",
```
﻿package.lock中的版本
```
"vue": {
  "version": "2.5.9",
}
```
package.lock文件是后期添加的，因为npm5.0+后才支持这个功能。通过package.lock可以记录和锁定依赖树的信息。初次install的时候，会自动生成package.lock文件。 需要注意的是在V5.4.2版本后如果改了package.json，且package.json和lock文件不同，那么执行npm install时npm会根据package中的版本号以及语义含义去下载最新的包，并更新package.lock文件。
锁定后带来的的问题是每次依赖包有bugfix（修订版本号）或者进行兼容性功能添加（次版本号）版本更新后，install的时候不会自动更新。

在查明起因后，解决的方法是安装使用新的vue版本，把原先的依赖包删掉，此时有遇到一个问题了，因为我的cnpm是4.X.X，每次安装的时候是2.3.3，而换用npm（5.6）安装是可以的（结果会是package.lock指定的版本）或者去掉﻿package.lock重新install（结果会是最新的vue包版本，还有配套的其它依赖更新）。
```
➜  small git:(20180531162035883_1003364(hrd5)) ✗ cnpm -v
4.3.2
  small git:(20180531162035883_1003364(hrd5)) ✗ which cnpm
/usr/local/bin/cnpm

```
如上所示，cnpm的重新安装了不成功的原因是我使用了nvm来管理，每次是安装对应到nvm安装目录下，对应当前node环境的node-modules目录下，把老的全局cnpm删了，重新装了下就可以了，结果如下。
```
 small git:(20180531162035883_1003364(hrd5)) ✗ cnpm -v
cnpm@6.0.0
  small git:(20180531162035883_1003364(hrd5)) ✗ which cnpm
/Users/jsdt/.nvm/versions/node/v8.9.4/bin/cnpm

```


参考链接
https://gist.github.com/yyx990803/9bdff05e5468a60ced06c29c39114c6b#simplified-scoped-slots-usage
https://cloud.tencent.com/developer/article/1020507